# 使用 iostat 监控磁盘性能

`iostat` 是一个用于监控磁盘性能的工具，它可以显示磁盘设备的读写性能、IOPS、吞吐量、等待时间等信息。`iostat` 是 `Linux` 系统自带的工具，无需额外安装。

## 磁盘监控命令说明

```bash
iostat -d -x -m 1 30
```

- **-d**: 显示磁盘设备信息
- **-x**: 显示扩展统计信息（如队列长度、合并请求等）
- **-m**: 以 MB 为单位显示吞吐量
- **1 30**: 每秒采样一次，连续采集 30 次

> 注意：输出的某些字段如 `rareq-sz`、`wareq-sz`、`%rrqm`/`%wrqm` 等和 `sysstat` 版本有关。  
> 完整参数解释可参考 [iostat man page](https://man7.org/linux/man-pages/man1/iostat.1.html)。

## 内核 3.10

```bash
rpm -q sysstat
sysstat-10.1.5-20.el7_9.x86_64
```

### 内核 3.10 输出示例

```bash
iostat -d -x -m 1 10
Linux 3.10.0-1160.119.1.el7.x86_64 (tdc-55)  04/16/25  _x86_64_ (48 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.05    0.90   21.90     0.02     0.17    16.69     0.75   32.93    2.29   34.19   0.39   0.88
sdb               0.00     0.09    0.75   24.87     0.05     0.26    24.45     0.09    3.52   26.66    2.83   0.46   1.17
sdd               0.00     0.16    0.17    0.34     0.00     0.03   135.21     0.03   59.13    0.48   87.57   0.81   0.04
sdc               0.00     1.54    0.03    2.66     0.00     0.07    58.44     0.06   21.57   29.94   21.48   0.38   0.10
```

### 内核 3.10 iostat 各字段含义详解表格

> 以下健康范围为常见设备的经验值，实际分析时应结合具体设备与业务负载综合判断。

| **字段** | **计算原理** | **健康范围** | **异常场景分析** |
|----------|--------------|--------------|------------------|
| **r/s** | 每秒原始读请求数（不包括合并），对应内核字段 `rio`，是反映读 IOPS 的关键指标之一 | `HDD`：`< 150`<br>`SATA SSD`：`< 20k`<br>`NVMe`：`< 80% 设备标称读 IOPS` | 持续升高说明读压力加大。<br>若 `await` 同时升高，可能存在 I/O 瓶颈。|
| **w/s** | 每秒原始写请求数（不包括合并后的请求），对应内核字段 `wio`，是反映写 IOPS 的关键指标之一 | 同上 | 高 `w/s` 结合 `wareq-sz`（写请求大小）判断：若 `wareq-sz` 较小，可能是小文件写入密集，通常会导致设备负载过高，需要优化写入方式或合并请求 |
| **rMB/s** | 每秒读取的数据量（单位：MB） | `HDD`：`< 200 MB/s`<br>`SATA SSD`：`< 550 MB/s`<br>`NVMe`：`数 GB/s` | 持续高吞吐量且 `await` 低 → 正常；若 `await` 高，需排查延迟问题，可能涉及磁盘寻址、硬件退化或 I/O 调度问题 |
| **wMB/s** | 每秒写入的数据量（单位：MB） | 同上 | 同上 |
| **avgrq-sz** | 每次 I/O 请求的数据量（单位：扇区，`1` 扇区=`512`字节） | 通常与文件系统块大小对齐（如 `4K=8` 扇区） | 远小于块大小：可能是碎片化读写，影响性能；<br>远大于块大小：可能是 `RAID` 条带化影响，跨多个磁盘的写入导致请求更大 |
| **avgqu-sz** | 平均队列长度，即单位时间内平均处于等待队列的 `I/O` 请求数 | `HDD`：`< 32`<br>`SATA SSD`：`< 256`<br>`NVMe`：`< 设备队列深度 × 80%` | 持续高于经验值说明存在明显排队<br>配合高 `%util` 或 `await` 说明磁盘 `I/O` 饱和<br>若低负载下该值仍持续非零，可能是应用 `I/O` 零散、调度器不合并请求，或存在慢设备影响 |
| **rrqm/s** | 每秒合并的读请求数 | 无固定范围（依赖调度策略和负载） | 合并多表示顺序读请求占主导，合并少可能是随机 `I/O` 操作较多 |
| **wrqm/s** | 每秒合并的写请求数 | 同上 | 类似于 `rrqm/s`，写 `I/O` 调度对合并策略影响更大 |
| **%rrqm**  | `rrqm/(r/s+rrqm)*100%` | `HDD`：`70-95%`<br>`SSD`：`50-80%`<br>`NVMe`：`0-30%` | `< 50%`：合并效果差，可能有过多随机 I/O；`> 95%`：可能因队列堆积导致合并延迟 |
| **%wrqm**  | `wrqm/(w/s+wrqm)*100%` | 同上 | 类似 `%rrqm`，对写 `I/O` 合并效果的评价 |
| **r_await** | 单个读请求平均总延迟（ms），包括**排队**时间和**服务**时间 | `HDD`：`< 15ms`<br>`SATA SSD`：`< 1ms`<br>`NVMe`：`< 0.1ms` | `> 50ms`：可能是 I/O 拥堵或设备性能退化，需排查磁盘负载或队列问题 |
| **w_await** | 单个写请求平均总延迟（ms），包括**排队**时间和**服务**时间 | 同上 | 高写延迟常见于刷盘或 `fsync` 密集型应用，可能影响应用性能 |
| **await** | 平均每个 I/O 请求的总延迟时间（单位：ms），包括**排队**时间和**服务**时间。 | `HDD`：`< 10ms`<br>`SATA SSD`：`< 0.5ms`<br>`NVMe`：`< 0.1ms` | `await` 高说明请求处理慢，可能是磁盘性能瓶颈、I/O 调度延迟或应用写入阻塞（如 `fsync`） |
| **svctm** | 平均服务时间（单位：ms），即请求从设备开始处理到完成所用时间，不包括排队时间 | `HDD`：`< 5ms`<br>`SATA SSD`：`< 0.2ms`<br>`NVMe`：`< 0.05ms` | 该指标在多队列或异步调度场景中不准确，已在新版本中弃用。应结合 `await` 和 `%util` 综合判断 `I/O` 状况 |
| **%util** | 设备 I/O 繁忙时间百分比，表示设备在采样间隔内有多大比例处于处理请求状态。| `HDD`：`< 70%`<br>`SSD`：`< 90%`<br>`NVMe`：`< 95%` | 持续高 `%util` 表示设备繁忙。<br>若 `await` 低但 `%util` 高 → 表示设备高效运行；<br>若 `%util` 高而 `await` 也高 → 存在瓶颈。<br>若 `%util` 长时间 <10%，而 `await` 却升高，则可能是调度器或上层问题导致 `IO` 发不出去 |

## 内核 4.18

```bash
rpm -q sysstat
sysstat-11.7.3-2.el8.x86_64
```

### 内核 4.18 iostat 输出示例

```bash
iostat -d -x -m 1 30

Linux 4.18.0-193.el8.x86_64 (qaperf147)  04/16/25  _x86_64_ (104 CPU)

Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sda             27.52  128.43      1.65      1.96     1.48     4.59   5.11   3.45    2.19    0.50   0.04    61.50    15.66   0.02   0.31
sdb             43.31  311.60      2.98     35.42     1.10     8.94   2.48   2.79    0.94    0.02   0.01    70.50   116.41   0.15   5.42
sdc             28.58   67.94      2.80      7.12     1.23     8.33   4.11  10.92    1.79    0.77   0.02   100.26   107.27   0.39   3.79
sdd             28.12   65.57      2.98      6.77     1.20     7.72   4.10  10.54    1.45    0.58   0.06   108.50   105.75   0.40   3.71
sde             26.74   65.45      2.80      7.00     1.12     7.49   4.03  10.27    2.06    0.89   0.03   107.14   109.59   0.40   3.66
```

### 内核 4.18 iostat 各字段含义详解表格

> 只输出变化的字段

| **字段**     | **计算原理** | **健康范围** | **异常场景分析** |
|--------------|--------------|--------------|------------------|
| **aqu-sz**   | 监控区间内设备队列中处于活动状态的 I/O 请求数的平均值 | 应远低于设备的最大队列深度。<br>例如：<br>- `SAS/HDD`：建议保持在最大队列深度的 `30%～70%` 范围内。<br>- `NVMe`：依据较大队列深度实际评估 | 当 `aqu-sz` 值持续接近或超过设备队列深度时，表明 `I/O` 请求堆积过多，可能存在瓶颈，应检查 `I/O` 调度及存储子系统配置 |
| **rareq-sz** | 平均每个读请求的数据大小（`KiB`）；计算公式：`rMB/s × 1024 ÷ r/s` | 应与系统或存储设备使用的块大小接近（如 4K/16K/64K） | 如果平均读请求大小远低于块大小，表明存在大量随机小读，可能导致 `I/O` 效率下降 |
| **wareq-sz** | 平均每个写请求的数据大小（`KiB`）；计算公式：`wMB/s × 1024 ÷ w/s` | 应与系统块大小匹配；不合适的较小写请求易引起写放大问题 | 当小写请求占比过大时，易触发写放大效应，增加设备写入负担，应评估是否需要优化写请求合并策略 |


## 性能指标使用说明

## 如何查看块设备信息

```bash
ls -l /sys/block/
total 0
lrwxrwxrwx 1 root root 0 Apr  3 10:45 sda -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/host0/target0:3:107/0:3:107:0/block/sda
lrwxrwxrwx 1 root root 0 Apr  3 10:45 sdb -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/host0/target0:3:108/0:3:108:0/block/sdb
lrwxrwxrwx 1 root root 0 Apr  3 10:45 sdc -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/host0/target0:3:109/0:3:109:0/block/sdc
lrwxrwxrwx 1 root root 0 Apr  3 10:45 sdd -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/host0/target0:3:110/0:3:110:0/block/sdd
lrwxrwxrwx 1 root root 0 Apr  3 10:45 sde -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/host0/target0:3:111/0:3:111:0/block/sde
```

## 查看设备参数

| **参数** | **查看方式** |
|---------------|---------------------------------------------|
| IO 调度器      | `cat /sys/block/sdX/queue/scheduler`        |
| 合并策略       | `cat /sys/block/sdX/queue/nomerges`         |
| 队列深度       | `cat /sys/block/sdX/queue/nr_requests`      |
| 多队列 CPU 分布 | `cat /sys/block/sdX/mq/0/cpu_list`          |

## 性能关键指标说明

**性能关键指标**：

- **%util**：设备是否饱和。
- **await**：IO 响应是否延迟。
  - **`r_await`/`w_await`**：分别表示读/写请求的延迟情况（在 `sysstat v11.0+` 开始独立统计）。
- **aqu-sz**：是否存在排队堵塞。
- **rareq-sz / wareq-sz**：分别表示每次读 / 写请求的平均大小，单位为 `KB`。可用来判断是大块顺序 `IO` 还是大量小块随机 `IO`。

## 常见异常场景分析

### 1. **%util 为 100% 不一定是异常**

`%util = 100%` 并不意味着性能瓶颈，需结合其他字段综合判断：

- **机械硬盘（HDD）**：
  - 若 `%util` 持续超过 `80%`，说明设备接近极限负载，需关注潜在瓶颈。
  
- **SSD / NVMe**：
  - 即使 `%util` 达到 `100%`，只要 `await` 和 `aqu-sz` 保持在正常（较低）水平，说明设备正在高效运行，通常不构成异常。

### 2. **await 偏高，说明存在 IO 响应延迟**

`await` 值偏高通常说明 I/O 子系统存在响应延迟。可能原因包括：

- 存储后端响应慢（如磁盘、`RAID` 或 `SAN` 设备）
- `I/O` 随机性高，命中率低，难以顺序优化
- `RAID`、虚拟化层或云平台底层引入额外延迟
- 存在写放大或日志频繁刷盘等行为

**排查建议：**

- 使用 `cat /sys/block/sdX/queue/nomerges` 查看是否启用了 I/O 合并（值为 `0` 表示启用，`1` 表示禁用）
- 使用 `iostat -t` 观察延迟指标随时间变化是否存在波动
- 使用 `blktrace`、`bpftrace` 等工具进一步分析 I/O 路径、请求类型与响应耗时

### 3. **%util 高但 await 低：设备高效运行**

当 `%util` 持续接近 `100%`，但 `await` 保持较低（如 `< 1ms` 或 `< 5ms`），通常意味着设备正在高效处理请求，**并未出现明显排队或延迟**。

**可能原因包括：**

- 设备吞吐能力强，能够快速处理大量并发 `I/O` 请求；
- 是 `SSD`、尤其是 `NVMe` 等高性能设备的常见表现；
- 操作系统或 `I/O` 调度策略配合良好，没有形成队列堆积。

**处理建议：**

- 属于正常现象，一般不需干预；
- 可结合 `aqu-sz` 观察是否存在 `I/O` 积压（若也低，则更能确认设备高效运行）；
- 若性能仍有需求，可从应用侧或并发模型优化着手，而非硬件侧。

### 4. **%util 高 + await 高 + aqu-sz 高：存在严重瓶颈**

此组合指标说明 I/O 请求在设备层大量堆积，响应延迟显著上升，**设备已成为性能瓶颈点**。

**典型现象：**

- `%util ≈ 100%`：设备已满负荷运转；
- `await 明显升高`（如 HDD > 30ms，SSD > 10ms，NVMe > 5ms）；
- `aqu-sz` 持续较高，说明请求排队严重。

**优化建议：**

- **增加磁盘并发能力**：
  - 添加磁盘、构建 RAID0/RAID10；
  - 或在云环境中绑定更多磁盘并行使用。
- **调整 I/O 调度器策略**：
  - 对于 SSD/NVMe，可考虑使用 `noop` 或 `none`；
  - 对于 HDD，使用 `deadline` 或 `bfq` 更利于减少延迟。
- **分析访问模式并优化应用逻辑**：
  - 识别热点数据，合理预读或缓存；
  - 减少频繁小 I/O 或随机写的场景；
  - 使用合并写、异步 I/O 提高效率。

### 5. **rareq-sz / wareq-sz 很小：大量小 I/O 操作**

`rareq-sz` 和 `wareq-sz` 分别表示平均读写请求大小（单位 KB），若该值明显偏小（如 < 4KB 或远小于块设备/文件系统的优化块大小），说明系统正在处理大量小 I/O 请求，效率通常较低。

**可能原因：**

- 小文件频繁读写（日志文件、配置文件等）；
- 日志系统频繁落盘（如数据库频繁刷日志页）；
- 应用访问模式为高随机性且未对齐；
- 应用未使用 `O_DIRECT`，存在频繁 `fsync` / `cache flush` 行为。

**优化建议：**

- 增大 I/O 请求的逻辑块大小（例如合并写、批量读）；
- 调整文件系统挂载参数：
  - 如 `noatime` 关闭访问时间更新；
  - 合理设置 `commit=秒数` 降低落盘频率；
- 应用侧优化：
  - 使用内存缓冲、写入聚合；
  - 启用预写日志（如 WAL）或顺序日志合并机制；
  - 在适用场景中使用 `O_DIRECT` + 缓存策略控制。

### 6. **%util 低但 await 高：存在“隐性 I/O 延迟”**

当 `%util` 显示较低（如 < 30%），但 `await` 明显升高（如 > 20ms），说明虽然设备总体忙碌度不高，但部分 `I/O` 请求处理时间异常，**存在潜在 I/O 路径或后端瓶颈**。

**常见原因包括：**

- 后端存储链路延迟高（如 `RAID`、`SAN`、虚拟磁盘）；
- 云平台或虚拟化环境中存在资源抢占、`QoS` 限流；
- 单线程阻塞等待特定 `I/O` 完成（虽总量不大，但响应慢）；
- 存储路径配置不当，如 `NUMA` 不匹配、`PCIe` 带宽受限；
- 文件系统层缓存失效，导致直接落盘延迟。

**排查建议：**

- 使用 `iostat -xtd 1` 观察设备响应趋势；
- 配合 `blktrace`、`bpftrace` 等工具分析具体 `I/O` 请求；
- 检查虚拟化层配置（如 `virtio-scsi` vs `virtio-blk`、`IO throttle`）；
- 查看是否存在跨 `NUMA` 节点访问、`I/O` 中断绑定错误等问题；
- 若为 `SAN`/`NAS` 设备，检查网络传输瓶颈或存储网关负载；
- 若使用了 `RAID` 卡，请重点排查 `RAID` 卡缓存设置。


### 总结判断思路

在实际排查中，建议综合分析以下几个核心指标：

| **指标** | **反映的问题** |
|------------|--------------------|
| `%util`    | 是否达到设备负载上限 |
| `await`    | 响应是否延迟       |
| `aqu-sz`   | 是否存在排队堵塞   |
| `rareq/wareq-sz` | I/O 请求是大是小 |

并结合以下上下文进一步判断：

- **存储类型**（`HDD` / `SSD` / `NVMe`）
- **I/O 模式**（顺序 / 随机）
- **应用访问模型**（数据库 / 日志 / 文件系统）
- **是否存在 RAID / 虚拟化 / 云平台抽象层**

### 判断建议：

- **三者同时高**（`%util`、`await`、`aqu-sz`）通常为瓶颈，表明系统 `I/O` 请求排队严重，设备处理响应缓慢，需考虑优化存储架构或增加资源。
- **任一异常**需结合上下文判断：
  - 如果 `%util` 显示较低，仍出现 `await` 高，可能是后端存储或 `RAID` 卡的问题。
  - 如果 `aqu-sz` 较高，可能存在请求排队或并发不足。
  - 如果 `rareq/wareq-sz` 很小，说明系统正处理大量小 `I/O` 请求，可能影响性能。

**注意**：这些指标相互影响，需要综合判断，不宜孤立解读。实际应用中，可以通过调整存储配置、优化应用访问模式、增加硬件资源等方式改善 `I/O` 性能。
