---
title: "Nginx 缓存体系——proxy_cache 的分层存储、缓存策略与淘汰机制"
date: 2026-07-29
description: 从 proxy_cache_path 的 levels 二级目录设计原理、proxy_cache_key 的精确命中控制、proxy_cache_valid 的时间策略与 proxy_cache_use_stale 的降级兜底、到缓存清理与缓存的一致性回源逻辑，拆解 Nginx 反向代理缓存的完整生命周期。
tags: ["Nginx","缓存","proxy_cache","CDN","性能优化"]
categories: ["网络"]
---

# 历史背景——Nginx 缓存是"穷人版 CDN"

2005 年前后，CDN 还是大公司的专利——Akamai 按流量计费，中国版蓝汛的报价是按 Mbps 月租。中小网站根本负担不起。Nginx `proxy_cache` 的出现让"在自有服务器上搭一个缓存层"变得只需要几行配置——把后端 API 的响应缓存在 Nginx 服务器的磁盘上，下次同样请求直接返回，不再穿透到后端。

但缓存不是免费的午餐。**缓存的难点从来不是"怎么存"，而是"什么时候让它失效"和"后端挂了怎么办"。** Nginx 在这两个问题上做了一套非常务实的方案：多层次过期策略、多级降级兜底、基于文件系统的两级目录管理。理解这些设计细节，你才能在生产环境放心地对 API 开启缓存。

---

# 一、缓存的基础配置与分层存储

## 1.1 一个最小的缓存配置

```nginx
http {
    # ① 定义缓存存储区域
    proxy_cache_path /data/nginx/cache 
                     levels=1:2 
                     keys_zone=my_cache:10m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;
    
    # ② 在需要缓存的地方引用
    server {
        location /api/ {
            proxy_cache my_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;
            proxy_pass http://backend;
        }
    }
}
```

## 1.2 levels=1:2——为什么不是"一个目录全塞进去"？

`levels=1:2` 表示缓存文件在磁盘上按两级散列目录存储：

```
/data/nginx/cache/
  ├── 3/
  │   ├── 2e/
  │   │   └── e1b2d9f8a3c4... → 缓存文件（第二级目录）
  │   ├── 5a/
  │   │   └── ...
  │   └── ...
  ├── 7/
  │   └── ...
  └── f/
      └── ...
```

**为什么分两层？** Nginx 对 `proxy_cache_key` 做 MD5，用结果的前几位来分布文件：
- `levels=1:2` → 第一层用 MD5 的最后 1 个字符（最多 16 个子目录）
- 第二层用接下来的 2 个字符（每第一层子目录下最多 256 个子目录）
- 总计 16 × 256 = 4096 个节点

```
假设 proxy_cache_key 的 MD5 = e1b2d9f8a3c4...
  levels=1:2 → 第一层: MD5 末 1 位 = "3" → 第二层: 接下来 2 位 = "2e"
  → 缓存文件放在 /data/nginx/cache/3/2e/e1b2d9f8a3c4...

为什么不是 levels=1（共 16 目录，每个目录可能有几十万文件）？
  → ext4/xfs 在一个目录中有几十万文件时，ls/rm 都是 O(N)，崩溃慢

为什么不是 levels=2:2:2（三层，每层 256 个，共 16,777,216 个目录）？
  → 目录数太多，inode 开销巨大，性价比低

levels=1:2 → 16 个一级目录 × 256 个二级目录 = 4096 个叶子目录
  10GB 缓存 ÷ 4096 ≈ 2.4MB/目录 → 目录大小在 ext4 的舒适区（< 1000 文件/目录）
```

## 1.3 keys_zone——缓存的"索引"存在内存中

```
proxy_cache_path keys_zone=my_cache:10m;

10MB 的共享内存用来存缓存的元数据：
  - 每个缓存项约占用 100-200 字节的元数据
  - 10MB ≈ 50000-100000 个缓存项
  - 这意味着你的缓存项数不宜超过 8 万
```

这不是限制，而是指导——如果你的缓存项数超过 8 万，要么加大 `keys_zone`，要么用 `proxy_cache_key` 做更粗粒度的缓存（合并相似的 key）。

---

# 二、缓存策略——哪些响应缓存、缓存多久、怎么命中

## 2.1 proxy_cache_key——缓存命中唯一定义

```nginx
# 默认值：
proxy_cache_key "$scheme$proxy_host$request_uri";

# 如果你要根据更多维度区分缓存：
proxy_cache_key "$scheme$request_method$host$request_uri$http_accept";

# 如果你想"忽略某些 query 参数"做缓存合并：
proxy_cache_key "$scheme$host$uri$arg_id";  # 只按 id 缓存，忽略 timestamp/random 参数
```

**key 的细微差别决定了"两个请求是否共享缓存"**——`$request_uri` 包含 query string，`$uri` 是规范化的（不含 query string）。如果你有 `?timestamp=xxx` 这种随机参数，务必在 key 中去掉（用 `$uri` 代替 `$request_uri`），否则缓存永远不会命中。

## 2.2 proxy_cache_valid——按 HTTP 状态码设置缓存时间

```nginx
proxy_cache_valid 200 302 10m;   # 成功和重定向缓存 10 分钟
proxy_cache_valid 404 1m;        # Not Found 缓存 1 分钟
proxy_cache_valid any 1m;        # 其余所有状态缓存 1 分钟
```

**后端已经指定了 Cache-Control 头怎么办？**

Nginx 有 `proxy_cache_valid` 和 `proxy_ignore_headers` 来协调：

```nginx
# 方式 1：让后端响应头决定缓存时长
proxy_cache_valid 200 10m;  # 如果后端没 Cache-Control，用这个兜底

# 方式 2：强制忽略后端 Cache 相关头
proxy_ignore_headers Cache-Control Expires Set-Cookie;
proxy_cache_valid 200 10m;
# 服务器强制缓存策略——后端无法说"不要缓存"
```

## 2.3 proxy_no_cache 与 proxy_cache_bypass——条件性缓存关闭

```nginx
# 场景：有 Authorization 头的请求不缓存（每个用户自己的数据）
proxy_no_cache $http_authorization;
proxy_cache_bypass $http_authorization;

# 场景：有特定 cookie 的请求不缓存（用户已登录，需要实时数据）
proxy_no_cache $cookie_sessionid;
proxy_cache_bypass $cookie_sessionid;

# 场景：任何请求体不为空的请求不缓存（POST/PUT 通常不缓存）
proxy_no_cache $request_body;
```

`proxy_no_cache` = 不给这次**请求的响应**建缓存（但以前的缓存还能用）。`proxy_cache_bypass` = 这次请求不读缓存（即使有缓存也穿透到后端去取新的）。两者配合可以在"特定条件下跳过缓存"而不影响其他请求。

---

# 三、缓存降级——use_stale 的"后端挂了也给你返回"

这是 Nginx 缓存最强大的生产特性。默认情况下，如果后端 502/超时，Nginx 返回 502 给客户端。但 `proxy_cache_use_stale` 让 Nginx 在**后端不可用时返回过期的缓存内容**：

```nginx
location /api/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 10m;
    
    # 降级策略：后端挂了返回过期缓存
    proxy_cache_use_stale error timeout updating
                          http_500 http_502 http_503 http_504;
    
    # 在更新缓存时也返回旧缓存（不阻塞请求去回源）
    proxy_cache_background_update on;  # 1.11.10+
    proxy_cache_lock on;               # 同时只有 1 个请求去回源
    proxy_cache_lock_age 5s;           # 锁 5 秒超时
    
    proxy_pass http://backend;
}
```

```
降级状态机：
  正常 → 后端返回 200 → 缓存 N 分钟 → 返回给客户端
  异常 → 后端 502/超时 → 查看是否有过期缓存：
        有 → 返回过期缓存（proxy_cache_use_stale 控制的头部）
        没有 → 返回 502
```

**这个机制意味着：即使后端全挂，只要之前的请求在缓存有效期内，Nginx 可以独立提供降级服务（用可能的旧数据）**。

---

# 四、缓存清理——内网管理接口

Nginx 没有内置的"过期缓存清掉"API。标准的做法是通过第三方模块 `ngx_cache_purge` 暴露一个管理端点：

```nginx
# 安装 ngx_cache_purge 模块后
location ~ /purge(/.*) {
    allow 127.0.0.1;
    deny all;
    proxy_cache_purge my_cache "$scheme$request_method$host$1";
}
# curl -X PURGE http://localhost/purge/api/users/123
# → 精确清除这个 URL 的缓存条目
```

如果没有 `ngx_cache_purge`，替代方案：
1. **设较短的过期时间**（1-2 分钟）+ 后端主动推送新数据
2. **动态切缓存 key**（如 `cache_key` 里加版本号 `$http_x_api_version`）——新版本号的请求自动 miss，旧版本号缓存自然过期
3. **清掉缓存文件**（`rm /data/nginx/cache/*/*/*`）+ `nginx -s reload`（粗暴但有效）

---

# 五、完整的缓存配置

```nginx
http {
    # 缓存存储
    proxy_cache_path /data/nginx/cache
                     levels=1:2
                     keys_zone=api_cache:100m
                     max_size=50g
                     inactive=120m
                     use_temp_path=off
                     loader_files=100         # 启动时一次性加载的缓存文件数
                     loader_sleep=50ms        # 批次间休眠
                     loader_threshold=300ms;  # 清仓的休眠阈值
    
    server {
        location /api/ {
            # 缓存引用
            proxy_cache api_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            
            # 缓存策略
            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_valid any 1m;
            
            # 条件跳过
            proxy_no_cache $http_authorization;
            proxy_cache_bypass $http_authorization;
            
            # 降级
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
            proxy_cache_background_update on;
            proxy_cache_lock on;
            
            # 代理
            proxy_pass http://backend;
            add_header X-Cache-Status $upstream_cache_status;  # 返回 HIT/MISS/BYPASS/EXPIRED/STALE
        }
    }
}
```

**`$upstream_cache_status` 是缓存调试的法宝**——在响应头加上这个字段，可以看到缓存是否命中。

---

# 六、总结

| 配置 | 作用 | 建议 |
|------|------|------|
| `levels=1:2` | 两级散列目录 | 默认就好，超大缓存用 `=1:2:2` |
| `keys_zone=100m` | 内存缓存索引 | 1MB ≈ 10000 项，以此估算 |
| `proxy_cache_valid` | 按状态码缓存时长 | 200: 10m/404: 1m |
| `proxy_no_cache` | 特定条件不建缓存 | 跳过 Authorization/cookie 的请求 |
| `proxy_cache_use_stale` | **后端挂了返回过期缓存** | 生产必开！error+timeout+5xx |
| `proxy_cache_lock` | 防止缓存击穿 | 同一 key 同时只会有一个请求穿透到后端 |

# 延伸阅读

**Do——动手验证：**
- 配置 `add_header X-Cache-Status $upstream_cache_status` 后，发两次同样的请求，观察第一次 MISS、第二次 HIT
- 停掉后端，发第三次请求，观察是否返回 STALE（`proxy_cache_use_stale updating` 需要生效）
- 用 `curl -X PURGE http://localhost/purge/api/test` 清理特定缓存（需 `ngx_cache_purge` 模块）

**Todo——深入方向：**
- `proxy_cache_use_stale` 与服务降级的最佳实践——什么场景用过期数据，什么场景宁可报错
- `proxy_cache_background_update` 的"异步续命"机制——缓存快过期时后台自动回源刷新
- Nginx Plus 的 `slice` 模块——大文件（视频/下载）的分片缓存

*本文参考资料：*
- Nginx 官方文档: ngx_http_proxy_module
- Nginx 源码 `src/http/modules/ngx_http_proxy_module.c`
- ngx_cache_purge: https://github.com/nginx-modules/ngx_cache_purge
