# 基于 Fizzle、Nyx 和 Python generator 的独立 fuzzing 引擎

## 目标与执行模型

采用“每个 case 独立执行、结束后恢复状态”的语义。宿主使用 LibAFL 调度和变异，通过 `libafl_nyx` 恢复 guest 快照；每轮不执行 `fork()`，也不调用 SUT 的 `__AFL_LOOP`。

已确定的首版范围：

- 新建独立引擎，保留旧 LibAFLstar 项目供参考。
- 一个 case 执行一次完整的 Python generator 交互，允许多次收发。
- Python worker、SUT、Fizzle 和内部 IPC 全部位于同一 guest，统一恢复快照。
- 首先支持单 TCP 信道，用受控的小型 C 服务验收。
- Nyx 端到端测试在另行准备的合适主机完成。

```text
宿主：LibAFL 参数变异、语料调度、覆盖反馈
                   │ case payload / 执行结果
                   ▼
Nyx guest：Python generator ⇄ 内部 IPC ⇄ Fizzle bridge ⇄ SUT
                   └──────── 同一个恢复快照 ────────┘
```

## 实现方案

**独立工程与输入契约**

- 在仓库新增 `fuzz/`，包含 Rust 引擎、guest harness、Python runtime、打包工具和测试目标。
- 提供 `pack`、`run`、`replay`、`doctor` 命令。配置描述 SUT、启动参数、固定 TCP endpoint、generator、依赖文件和超时。
- 保留 `{generator, params}` 输入形式；打包时将脚本路径转换为 guest 内已登记的 generator 标识，运行时只变异参数。
- 固定 LibAFL、Fizzle、libnyx、QEMU-Nyx 和 packer 版本。Fizzle 修改保存为仓库内可重放的补丁，避免改动只留在被忽略的第三方目录。

**Nyx guest 生命周期与覆盖率**

- 启动 Python worker、导入 generator、建立 IPC，再启动 SUT；在 SUT 已就绪且尚未接收 case 输入、Python 等待任务时建立唯一初始快照。
- 每轮恢复后读取新 payload，执行完整交互，提交结果并调用 Nyx release。正常、崩溃和超时之后都从同一快照恢复。
- Fizzle 不启用 `afl/pcr`；新增 guest harness 管理输入和轮次边界，绕开 Fizzle 原有 stdin 输入及自动退出逻辑。
- SUT 保留 AFL++ 源码覆盖率插桩，由 guest shim 初始化 bitmap 并注册给 Nyx，宿主使用 `NyxExecutor` 的 bitmap observer。首版不采用 Intel PT。
- 不叠加 stock Nyx fuzz preload：它也接管输入、进程和退出路径，需要提取所需机制接入自定义 harness。[上游实现](https://github.com/nyx-fuzz/packer/blob/main/packer/linux_x86_64-userspace/src/ld_preload_fuzz.c)
- SUT 的可写目录与临时文件放在 guest RAMFS，保证文件状态随快照恢复。

**Python 与 Fizzle 桥接**

- 保留同步 `generate(ctx, params)`，首版实现默认 TCP 连接、`send`、现有参数形式的 `recv`、关闭和超时；UDP、监听及第二条连接明确返回“不支持”。
- Python worker 不加载 Fizzle，通过预先创建的 guest 内 socketpair 发送逻辑 I/O 请求；首版不沿用 iceoryx2。
- Rust bridge 使用带长度前缀的 CBOR 请求与响应，维护输入队列、输出缓冲和待决读取。
- Python 请求尚无法满足的 `recv` 时，bridge 保存请求并返回 Fizzle 调度器；SUT 输出到达后再恢复 Python，避免插件回调等待 SUT 而死锁。
- 接收分隔符、长度、EOF 和超时均由虚拟信道处理；I/O 截止时间使用 Fizzle 虚拟时间。真实时间 watchdog 只用于终止卡死的执行。
- generator 返回后，先让已发送的数据被 SUT 处理、排空可运行任务，再关闭信道并结束 case，避免遗漏最后一次只发送的请求。

**Fizzle 调度扩展与结果分类**

- 新增 case begin、idle、完成和失败入口，替换基于“暂时没有工作”的隐式轮次判断。
- 当前时刻的桥接事件先处理，再推进到最近的 SUT timer 或 Python 接收截止时间。
- 显式实现 EOF、暂不可读和关闭状态，修复现有插件路径中相关 `unimplemented!()` 分支。
- 区分 SUT crash、执行 timeout、Python generator error 和基础设施错误；Python 或 bridge 失败不计为 SUT 漏洞。
- 保存可重放输入、覆盖摘要、逻辑 I/O transcript、结果与依赖版本。回放复用同一执行路径。

## 测试与验收

- **交互正确性：** 覆盖响应驱动的后续发送、拆包、粘包、分隔符、EOF、关闭和接收超时。
- **完成边界：** generator 最后只发送不读取时，仍能观察到 SUT 对该消息产生的覆盖率。
- **确定性：** 同一 case 重复执行 1,000 次，覆盖 bitmap、逻辑 transcript 和结果一致；不比较墙钟时间或运行日志。
- **恢复完整性：** 使用 `A → B → A` 验证 SUT 内存、Python globals、队列和文件没有跨 case 污染；crash、timeout、Python 异常后仍能执行正常 case。
- **覆盖反馈：** 受控目标包含可区分的分支和确定性崩溃输入，验证 Nyx 返回的是 SUT 覆盖率，LibAFL 能据此保留新语料。
- **执行成本：** 验证快照建立后每个 case 没有新增进程或重新导入 Python；记录吞吐量、恢复时间和桥接开销，再与基础 forkserver 基线比较。
- 本机先完成桥接和调度测试；缺少 Nyx 条件时明确跳过 VM 测试，不自动切换后端并宣称验收通过。

## 默认与后续边界

首版采用 Linux x86-64、单 worker、源码可编译目标和单一初始快照。Python generator 使用 `ctx` 通信；不承诺任意真实时钟、外部服务或后台线程具有确定性。

提供符合所固定 Nyx 版本要求的主机准备说明和环境检查，不在本次实现中修改当前主机内核或 KVM 配置。[Nyx 环境要求](https://github.com/AFLplusplus/AFLplusplus/blob/stable/nyx_mode/README.md)

动态多 TCP、FTP 控制／数据连接、UDP、监听、多协议状态快照及 RSC 模型到 generator 的生成链路，留到首版验收后扩展。
