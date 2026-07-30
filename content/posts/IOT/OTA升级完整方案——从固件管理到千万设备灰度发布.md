# OTA 升级完整方案——从固件管理到千万设备灰度发布

> OTA (Over-The-Air) 升级是物联网平台的核心能力，也是面试中最能区分"做过"和"用过"的考题。本文从固件管理的全生命周期讲起，拆解差分升级、断点续传、签名校验、灰度策略、速率控制、异常回滚六大核心模块的完整设计方案。

---

## 一、OTA 的整体架构

一次完整的 OTA 升级涉及四个角色：

```
┌─────────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────┐
│  运营后台    │ →  │  云平台 OTA   │ →  │  MQTT Broker  │ →  │  设备端  │
│             │    │  升级服务     │    │               │    │  OTA     │
│ 上传固件    │    │ 版本管理     │    │ 指令下发      │    │ 下载     │
│ 创建升级任务│    │ 灰度策略     │    │ 进度上报      │    │ 校验     │
│ 监控进度    │    │ 速率控制     │    │               │    │ 安装     │
│             │    │ 回滚管理     │    │               │    │ 重启     │
└─────────────┘    └──────────────┘    └───────────────┘    └──────────┘
```

**为什么不直接从设备端拉取固件？** HTTP 下载固件包很简单，但 IoT 设备不一定有公网 IP，且直接暴露固件下载地址有安全隐患。标准做法是：设备收到 OTA 通知 → 设备主动向平台发起 HTTPS 下载请求（带 token 认证）→ 平台校验设备身份和升级权限 → 返回固件流。

---

## 二、固件管理

### 2.1 固件的生命周期

```
创建 → 验证 → 就绪 → 灰度中 → 全量 → 归档
  ↓      ↓      ↓               ↓
 废弃   废弃   暂停             紧急回滚
```

每个状态的流转条件：
- **验证**：固件上传后自动做完整性校验（MD5）、签名验证、在测试设备池自动刷入验证
- **就绪**：验证通过，可以创建升级任务
- **灰度中**：任务创建后按策略逐步放开
- **全量**：灰度完成后覆盖所有目标设备
- **归档**：新版固件发布后，旧版本标记为归档，保留 30 天后清理

### 2.2 固件存储

```sql
CREATE TABLE firmware (
    id            BIGINT PRIMARY KEY,
    product_key  VARCHAR(32)  NOT NULL,     -- 产品标识
    version       VARCHAR(32)  NOT NULL,     -- 版本号，如 "2.3.1"
    file_name     VARCHAR(256) NOT NULL,     -- 原始文件名
    file_size     BIGINT       NOT NULL,     -- 文件大小（字节）
    file_md5      VARCHAR(32)  NOT NULL,     -- 整包 MD5
    file_sign     TEXT         NOT NULL,     -- RSA 签名
    oss_url       VARCHAR(512) NOT NULL,     -- OSS 存储地址
    diff_url      VARCHAR(512),              -- 差分包地址（可选）
    diff_size     BIGINT,                    -- 差分包大小
    diff_md5      VARCHAR(32),               -- 差分包 MD5
    firmware_type TINYINT      NOT NULL,     -- 1=MCU 2=模组 3=应用
    upgrade_mode  TINYINT      NOT NULL,     -- 1=全量 2=差分
    release_note  TEXT,                      -- 版本说明
    status        TINYINT      NOT NULL,     -- 1=验证中 2=就绪 3=已归档 4=已废弃
    created_at    DATETIME     NOT NULL,
    UNIQUE KEY uk_product_version (product_key, version)
);
```

**签名机制**：固件上传后，平台用私钥对固件做 RSA-SHA256 签名，签名值存入 `file_sign`。设备下载固件后用对应公钥验签——确认固件未被篡改且确实来自平台。公钥烧录在设备固件中，出厂后不可变更。

### 2.3 差分升级

全量升级每次下载完整固件（几 MB 到几十 MB），对流量和下载时间都是浪费。**差分升级**只下发新旧版本的差异部分。

```
差分算法流程:
  V1 固件 (旧) + V2 固件 (新) → bsdiff/hdiffpatch → 差分包 (patch)
  
设备端流程:
  下载 patch → 本地 V1 固件 + patch → 合成 V2 固件 → 校验签名 → 刷写
```

差分升级的代价：
- 服务端需要为每个旧版本生成对应的差分包。10 个历史版本 × 每个新版本 = O(n) 存储成本
- 设备端需要额外的闪存空间：当前固件 + patch + 新固件（合成中）三者同时存在时需要 2.5 倍固件大小的空间
- 差分算法 (bsdiff/hdiffpatch) 在 MCU 上运行时 CPU 和 RAM 占用显著——需要在设备端实测确认可行性

**实用建议**：固件 < 1MB 直接用全量，简单可靠。固件 > 10MB 再考虑差分——投入产出比才划算。

---

## 三、升级任务管理

### 3.1 任务模型

```sql
CREATE TABLE ota_task (
    id              BIGINT PRIMARY KEY,
    task_name       VARCHAR(128) NOT NULL,
    product_key    VARCHAR(32)  NOT NULL,
    firmware_id     BIGINT       NOT NULL,     -- 目标固件
    target_type     TINYINT      NOT NULL,     -- 1=全量 2=标签 3=设备列表
    target_tags     JSON,                      -- 标签条件 {"region":"east"}
    target_devices  TEXT,                      -- 设备 ID 列表（JSON 数组）
    target_count    INT          NOT NULL,     -- 目标设备总数
    strategy        JSON         NOT NULL,     -- 灰度策略
    status          TINYINT      NOT NULL,     -- 1=待开始 2=灰度中 3=全量 4=已完成 5=已终止
    created_at      DATETIME     NOT NULL,
    started_at      DATETIME,
    finished_at     DATETIME
);
```

### 3.2 目标设备筛选

三种筛选方式：

**全量推送**：该产品下所有设备。风险最大，必须经过充分灰度。

**标签筛选**：设备注册时打标签（地域、批次、渠道）。如 `region=guangdong AND batch=2026Q1`。

**设备列表**：手动指定设备 ID 列表。适合定向修复某批已知设备的问题。

标签筛选的实现用 bitmap 或倒排索引：`tag:region=guangdong → Set<deviceId>`，多个标签取交集。

---

## 四、灰度策略

### 4.1 分批灰度模型

```json
{
  "batches": [
    {"ratio": 0.01,  "min_devices": 10,   "pause_hours": 4},
    {"ratio": 0.05,  "min_devices": 100,  "pause_hours": 8},
    {"ratio": 0.20,  "min_devices": 1000, "pause_hours": 24},
    {"ratio": 1.0,   "min_devices": 0,    "pause_hours": 0}
  ],
  "auto_pause_rules": {
    "max_failure_rate": 0.02,
    "max_offline_rate": 0.05
  },
  "push_window": {
    "start": "02:00",      -- 凌晨 2 点开始推送
    "end":   "06:00"        -- 6 点前必须停，避免影响白天业务
  }
}
```

**分批执行流程**：

```
批次 1 (1%): 下发 → LED 屏/智能音箱用户静默 → 各等 4 小时观察失败率
   ↓ 异常率 < 2%，自动进入下一批
批次 2 (5%): 下发 → 观察 8 小时
   ↓
批次 3 (20%): 下发 → 观察 24 小时（覆盖一整天的使用场景）
   ↓ 确认所有指标正常
全量 (100%): 大规模铺开
```

**自动熔断条件**：
- 升级失败率 > 2% → 自动暂停任务，通知运维
- 设备升级后离线率异常升高（升级后 1 小时内离线占比 > 5%）→ 自动暂停
- 设备升级后 CPU/内存异常 → 自动暂停

### 4.2 灰度引擎的 Java 实现

```java
public class OtaGrayEngine {
    
    private OtaTaskMapper taskMapper;
    private DeviceMapper deviceMapper;
    private MqttPublisher mqttPublisher;
    
    /**
     * 每次调度触发：检查是否有待推送的批次
     */
    @Scheduled(fixedDelay = 60000)  // 每分钟一次
    public void schedule() {
        List<OtaTask> runningTasks = taskMapper.findByStatus(GRAYING);
        for (OtaTask task : runningTasks) {
            GrayBatch currentBatch = task.getCurrentBatch();
            
            // 1. 检查暂停窗口：是否过了观察时间
            if (!currentBatch.isPauseExpired()) {
                continue;
            }
            
            // 2. 检查是否在推送时间窗口内
            if (!task.getPushWindow().isInWindow(LocalTime.now())) {
                continue;
            }
            
            // 3. 检查自动熔断条件
            TaskStats stats = calculateStats(task.getId());
            if (stats.getFailureRate() > task.getAutoPauseRules().getMaxFailureRate()) {
                taskMapper.pauseTask(task.getId(), "失败率超阈值: " + stats.getFailureRate());
                notifyOps(task, stats);
                continue;
            }
            
            // 4. 推送下一批设备
            int batchSize = currentBatch.calculateSize(task.getTargetCount());
            List<String> deviceIds = selectNextBatch(task, batchSize);
            for (String deviceId : deviceIds) {
                OtaCommand cmd = OtaCommand.builder()
                    .deviceId(deviceId)
                    .firmwareUrl(task.getFirmware().getOssUrl())
                    .firmwareMd5(task.getFirmware().getFileMd5())
                    .firmwareSign(task.getFirmware().getFileSign())
                    .firmwareSize(task.getFirmware().getFileSize())
                    .build();
                mqttPublisher.send(deviceId, cmd);
            }
            currentBatch.markDispatched(deviceIds);
        }
    }
    
    private List<String> selectNextBatch(OtaTask task, int batchSize) {
        // 排除已推送的设备
        // 确保设备在线（查设备状态缓存）
        // 随机选取，避免都选到同批次设备
        List<String> onlineDevices = deviceMapper.findOnlineByProduct(task.getProductKey());
        Set<String> alreadyPushed = taskMapper.getPushedDeviceIds(task.getId());
        onlineDevices.removeAll(alreadyPushed);
        Collections.shuffle(onlineDevices);
        return onlineDevices.subList(0, Math.min(batchSize, onlineDevices.size()));
    }
}
```

---

## 五、速率控制 — 千万设备的核心挑战

### 5.1 为什么需要速率控制

1000 万设备瞬间开始下载固件：
- OSS 带宽被打爆 → 其他业务受影响
- 下载流量费用飙升（CDN 回源请求）
- 蜂窝网络设备集中下载 → 基站拥塞 → 连带影响通话和短信
- MQTT Broker 下发通知瞬间海量 → Broker 内存飙升

**速率控制的本质**：用时间换稳定性——把瞬时压力摊到更长时间窗口。

### 5.2 三层限速

```
第一层：任务级限速 → 控制整体并发下载设备数
第二层：区域级限速 → 避免单一基站/数据中心过载
第三层：设备级限速 → 控制单设备下载速率
```

**任务级限速**：

```java
public class OtaRateController {
    
    // 令牌桶：控制全局并发下载设备数
    private final RateLimiter globalRateLimiter;
    
    // 信号量：控制 OTA 通知下发速率
    private final Semaphore notifySemaphore;
    
    public OtaRateController(OtaTask task) {
        // 每批下发间隔，避免瞬间涌入
        long batchIntervalMs = 10_000;  // 10 秒发一批
        
        // 每批大小 = 全局最大并发 / (60秒 / 批次间隔)
        // 例: 50000 / (60/10) = 8333 个设备/批
        int batchSize = task.getMaxConcurrentDownloads() / 
                        (int)(60_000 / batchIntervalMs);
        
        this.globalRateLimiter = RateLimiter.create(batchSize / 10.0);  // 每秒允许多少
        this.notifySemaphore = new Semaphore(batchSize);
    }
    
    public void dispatchBatch(List<String> deviceIds) {
        for (String deviceId : deviceIds) {
            globalRateLimiter.acquire();  // 等待令牌
            notifySemaphore.acquire();
            
            try {
                sendOtaNotify(deviceId);
            } finally {
                // 设备下载完成（或超时）后释放信号量
                registerTimeoutCallback(deviceId, () -> {
                    notifySemaphore.release();
                });
            }
        }
    }
}
```

**区域级限速**：按设备所在地域（IP 归属/基站 LAC）分组，每组独立限速。主要针对蜂窝设备——防止同一基站下的几千台 NB-IoT 设备同时下载。

### 5.3 设备侧防重入

每个批次下发时打上批次版本号，设备端决策权在上层应用或 MCU 侧，收到 OTA 通知后：
- 新版本号 = 当前版本 → 忽略
- 新版本号 < 当前版本 → 拒绝，上报 `VERSION_ROLLBACK_DENIED`（防回滚攻击）
- 新版本号 > 当前版本 → 执行升级
- 已在下载中 → 忽略（不重复下载同一版本）
- 蜂窝网络 → 等待 Wi-Fi 可用（可配置策略，允许用户选择"流量也下"）

---

## 六、设备端 OTA 流程

### 6.1 双区备份 (A/B 分区)

设备闪存划分为两个等大的分区：

```
┌──────────────────────────────────────────┐
│  Bootloader  │  Flag 分区  │  A 区  │  B 区  │
│  (不可变)    │  (1KB)      │ (2MB)  │ (2MB)  │
└──────────────────────────────────────────┘

Flag 分区记录:
  running_slot = A           # 当前从 A 区启动
  slot_A_status = VALID      # A 区固件校验状态
  slot_B_status = UPDATING   # B 区固件校验状态
  boot_attempt = 0           # 当前分区的启动尝试次数
```

**升级流程**：

```
1. 设备在线运行在 A 区
2. 收到 OTA 通知 → 下载新固件到 B 区
3. 写入完成后校验 B 区 MD5 + RSA 签名
4. 校验通过 → 写 Flag 分区: slot_B_status = VALID, running_slot = B
5. 软件复位 (或调用 bootloader 跳转)
6. Bootloader 检查 Flag 分区 → 发现 running_slot = B
7. 校验 B 区签名 → 通过 → 跳转到 B 区执行
8. 新固件启动成功 → 上报 "升级成功" → 确认 Flag 分区写入最终状态
```

**回滚流程**：

```
如果 B 区固件启动后 3 分钟内未上报"升级成功":
  → 平台标记该设备 "升级超时"
  
设备端的看门狗保护:
  → B 区固件启动后连续 3 次启动失败
  → Bootloader 检测到 boot_attempt >= 3
  → 自动切回 A 区: running_slot = A
  → 上报: "升级失败，已回滚到版本 X"
```

这个回滚对 MCU 设备最关键的保障是：**bootloader 的逻辑必须写在 ROM 里或硬件写保护，不能依赖固件自身**。如果固件已经跑飞，它连"判断自己是否需要回滚"都做不到，必须由独立的引导代码来做这个判断——也就是在 bootloader 里检测 Flag 分区 + 启动计数器，固件自身不参与回滚决策。

### 6.2 下载与校验

```
设备端下载流程:
  1. 解析 OTA 通知: {url, md5, sign, size, chunk_count}
  2. 检查 B 区剩余空间 >= size
  3. 分片下载: HTTP Range 请求，每片 256KB
     GET /firmware/v2.3.1.bin HTTP/1.1
     Range: bytes=262144-524287
  4. 每片落地: 写入 B 区对应偏移，记录已下载分片索引
  5. 断线重连: 从最后未完成分片继续（HTTP Range 天然支持）
  6. 全部分片下载完成 → 全量 MD5 校验
  7. RSA 签名验签
  8. 校验通过 → 更新 Flag 分区 → 重启
```

**为什么不每个分片都做 MD5？** 

每个分片做 MD5 在工程上有两个问题：
- HTTP Range 请求用的是 TCP，TCP 本身已经保证数据不会在传输中损坏（有 checksum）。分片级别的损坏几乎是 0
- 在低功耗 MCU 上对着 256KB 数据跑一次 MD5 耗时几十到几百毫秒——N 个分片就是 N 倍的浪费。只在最终全量做一次 MD5 + 一次 RSA 验签就够了

但一种极端场景：Flash 写入过程中发生掉电，某分片写了一半。这个在设备端做"写入后回读比对"可以解决——写入后把该分片内容读出来跟下载缓冲区逐字节比对，不一致就说明写入失败，重新写。这比 MD5 便宜得多（只做一次 memcmp，不需要算 hash）。

---

## 七、升级进度监控

### 7.1 统计维度

```sql
-- 实时统计（Redis + 定时刷 MySQL）
ota_task_stats:{taskId}
  total:      100000       # 目标设备总数
  notified:   50000        # 已下发通知
  downloading: 15000       # 下载中
  downloaded:  34800       # 下载完成
  installing:  12000       # 刷写中
  success:     22000       # 升级成功
  failed:      800          # 升级失败
  offline:     700          # 离线未响应
  timeout:     300          # 已通知但超时未响应
```

### 7.2 设备上报的状态机

```
NOTIFIED → DOWNLOADING → DOWNLOADED → INSTALLING → SUCCESS
    ↓          ↓            ↓            ↓            ↓
  OFFLINE    FAILED       FAILED       FAILED       FAILED
                ↓
          (可重试，最多 3 次)
```

设备在每个状态变更时通过 MQTT 上报。特殊情况处理：
- 设备从 NOTIFIED 后 30 分钟无任何上报 → 标记为 TIMEOUT，可重推
- 设备从 DOWNLOADING 后 2 小时未完成 → 标记为 TIMEOUT
- 设备上报 FAILED 时携带错误码：`DOWNLOAD_FAILED` / `MD5_MISMATCH` / `SIGN_INVALID` / `FLASH_WRITE_ERROR` / `BOOT_FAILED`

### 7.3 异常设备处理

```java
public enum OtaErrorCode {
    DOWNLOAD_FAILED,     // 下载失败 → 可重试（可能是网络问题）
    MD5_MISMATCH,        // MD5 不匹配 → 可能是传输损坏，允许重试一次
    SIGN_INVALID,        // 签名无效 → 固件被篡改！立即告警安全团队
    FLASH_WRITE_ERROR,   // 闪存写入失败 → 硬件可能损坏
    BOOT_FAILED,         // 启动失败 → 回滚到旧版本
    VERSION_ROLLBACK,    // 新版本号低于当前版本 → 拒绝
    INSUFFICIENT_SPACE   // 空间不足
}

// 不同错误码的处理策略
public RetryStrategy getRetryStrategy(OtaErrorCode code) {
    return switch (code) {
        case DOWNLOAD_FAILED -> RetryStrategy.retry(3, Duration.ofMinutes(5));
        case MD5_MISMATCH    -> RetryStrategy.retry(1, Duration.ofMinutes(1));
        case SIGN_INVALID    -> RetryStrategy.NEVER;  // 立即告警
        case BOOT_FAILED     -> RetryStrategy.ROLLBACK; // 回滚
        case FLASH_WRITE_ERROR -> RetryStrategy.NEVER; // 硬件故障
    };
}
```

---

## 八、回滚方案

### 8.1 触发回滚的条件

- **手动回滚**：运维发现严重 Bug → 在控制台点击"终止任务并回滚"
- **自动回滚**：失败率 > 2% 或 离线率异常 > 5% → 自动触发
- **单设备回滚**：某设备升级后 3 次启动失败 → 设备端自动回滚

### 8.2 回滚执行流程

```
平台端:
  1. 管理员点击"回滚"
  2. 创建回滚任务: target_version = 上一稳定版本
  3. 对"已升级到问题版本的设备"触发回滚 OTA
  4. 回滚任务走同样的灰度? → 通常不走灰度，尽快全量

设备端:
  1. 收到回滚指令（带目标版本号）
  2. 校验目标版本号 < 当前版本号 → 合法
  3. 下载目标版本固件到备用分区
  4. 校验 → 切换分区 → 重启
  
自动回滚（设备端自主）:
  1. 新固件启动后 3 次 watchdog 复位
  2. Bootloader 检查 boot_attempt >= 3
  3. 自动切换回旧分区
  4. 旧固件正常启动后上报"已自动回滚"
```

### 8.3 回滚后的数据兼容

OTA 不能光考虑固件，还要考虑**固件升级后产生的数据**——数据版本可能跟旧固件不兼容：

- 新旧固件的数据格式兼容性必须在发版前验证
- 如果新固件改了数据结构（如新增必填字段），回滚后旧固件读到新格式数据会解析失败
- 对策：
  - **向前兼容**：新字段加默认值，旧固件读到忽略未知字段
  - **数据版本号**：数据结构带版本号，启动时检测到不兼容 → 使用备份数据或恢复出厂设置
  - **升级前数据备份**：升级时把关键配置备份到独立分区，回滚时恢复

---

## 九、安全设计

**身份认证**：设备下载固件时携带 OTA Token（一次性或短有效期），平台验证 Token 后才返回文件流。防止未授权设备下载固件逆向。

**传输安全**：固件下载必须走 HTTPS。固件包在 OSS 上加密存储（AES-256），下载时解密传输或由设备端解密。

**防回滚攻击**：设备端检查固件版本号必须单调递增，拒绝刷入低于当前版本的固件。版本号比较不是简单的字符串比较，而是语义版本号的严格递增判断（major.minor.patch）。

**签名验证不可跳过**：Bootloader 中验签逻辑必须在 ROM 中或硬件写保护区域，固件自身不能绕过。任何进设备 Flash 的固件在刷写前必须验签通过，否则不写入。

**防降级**：即使版本号递增，攻击者可能构造一个"版本号很大但存在已知漏洞"的假固件——防降级的关键不在版本号大小，而在于**签名校验**。没有开发者私钥签名的固件，版本号再大也会在验签阶段被拦截。

---

OTA 不是"把文件发过去"这么简单。从固件管理的全生命周期，到灰度策略的精细控制，再到设备端的双区备份与自愈回滚——每个环节的容错设计和安全防护，决定了千万设备升级时是"一切正常"还是"大规模变砖"。
