# CodeX 面向 RTL 仓库阅读与系统级激励生成的调研与方案

## 1. 结论先行

如果目标是让 CodeX 阅读一个很大的 RTL 仓库，并基于仓库级上下文生成系统级 `C`/裸机固件激励，那么最核心的设计原则不是“把更多 RTL 文本塞进上下文”，而是下面四点：

1. **把仓库变成可检索的结构化知识，而不是大段源码文本**
2. **把激励生成粒度提升到“软件动作/事务级”，而不是自由信号级**
3. **把“如何打到某模块”建模成可达性与路径分析问题**
4. **把覆盖率、波形、失败信息做成反馈闭环，而不是一次性生成**

换句话说，这个问题的本质不是 prompt engineering，而是：

- 仓库级知识建模
- 受约束生成
- 路径级检索
- 验证反馈闭环

如果系统设计得当，CodeX 不需要“记住整个仓库”；它只需要在每次任务里拿到一小包**高密度、任务相关、已结构化**的上下文。

## 2. 调研综述

### 2.1 UVM2：IP 级自动验证闭环的起点

`UVM2` 的核心思路是用多个任务型 LLM agent 配合模板，自动完成 test planning、UVM testbench 生成、仿真分析和 testcase supplement，并且用 coverage feedback 迭代补激励。[UVM2](https://arxiv.org/html/2504.19959v3)

它给你的主要启发有三点：

- **验证流程应拆成多个角色**，而不是一个万能 agent 一把梭
- **coverage-driven testcase supplement** 是必须的
- **模板与约束** 对生成质量非常关键

它的局限也很明显：

- 主要针对 IP 级或中等规模 RTL
- 更偏 UVM/testbench 自动化
- 对“仓库级设计理解”和“软件驱动的模块触达”支持有限

所以它适合借来做“多 agent + 覆盖闭环”的框架思路，但不够直接回答你“核级 C 激励如何打到深层模块”。

### 2.2 UVMarvel：子系统级结构化理解与路径追踪

`UVMarvel` 是更贴近你问题的一条路线。它的关键不是让 LLM 直接读完整个 subsystem RTL，而是先引入：

- 验证导向 `IR`
- `Bus Protocol Library`
- `Coverage Analyser`
- `Signal Tracker`
- `Verilog Patching / filtered DUT`

然后走一条明确链路：

- 先把规格和 RTL 压成验证所需的结构化表示
- 覆盖率空洞出来后，先做未覆盖点分解
- 再由 `Signal Tracker` 找相关信号路径
- 再生成局部精简上下文
- 最后才补激励或给出 waive 建议

这条路线对你最重要的价值在于两点：[UVMarvel](https://arxiv.org/html/2605.04704v2)

- **它证明了仓库/子系统级任务不能靠全量上下文硬喂**
- **它把“怎么打到某逻辑”变成路径追踪问题，而不是纯语言生成问题**

它的局限是：

- 主战场仍然是 UVM/testbench
- 主要面向子系统级验证，不完全等同于你关心的“软件式 C 激励”

但它几乎直接给出了你最需要迁移的两块：

- `IR / summary first`
- `coverage hole -> path -> sliced context -> regenerate`

### 2.3 HAVEN：合法性靠 DSL/模板，不靠模型自由发挥

`HAVEN` 的核心价值在于，它明确不相信 LLM 可以稳定、自由、正确地直接写复杂验证代码，而是采用：

- 结构化架构计划
- 协议相关模板
- `Protocol-Aware Sequence DSL`
- 规则生成器

再让 LLM 只在被约束的空间里工作。[HAVEN](https://arxiv.org/abs/2604.27643v1)

这对你的问题尤其关键，因为你已经明确提到：

- 你担心激励是否合法
- 你不知道生成的信号是否满足协议/时序/前置条件

HAVEN 给出的工程性答案就是：

- 不要把主生成对象设成“原始信号”
- 要把主生成对象设成“可检查的 DSL/模板参数”

对你的场景，可以把这个思想迁移成：

- LLM 负责生成**软件动作计划**
- 代码生成器负责生成最终 `C` 激励
- 合法性检查器负责过滤非法动作

### 2.4 FireBridge：软件激励与 RTL 协同验证的桥

你已经明确说更偏向系统级 `C` 激励，运行环境主要是裸机/固件。这时最相关的方向不是纯 UVM，而是 `HW/FW co-verification`。

`FireBridge` 的价值在于，它把生产级 firmware 与 RTL/gate-level 硬件桥接起来，支持：

- cycle-accurate co-verification
- register-level protocol testing
- memory bridge / congestion emulation
- firmware + RTL 联调

它的关键启发是：[FireBridge](https://arxiv.org/html/2603.25969)

- **软件动作本身就是合法激励的天然抽象**
- **真正的系统级行为应该从 CPU/firmware 入口去驱动**
- **寄存器、DMA、内存访问、中断、总线事务，可以成为“软件到内部模块”的统一桥梁**

这条路线特别适合你，因为它让“怎么激活某模块”不再是“如何构造底层信号”，而是：

- 哪段 firmware
- 哪些 MMIO 写
- 哪个 descriptor/buffer
- 哪个 doorbell/interrupt
- 哪种内存条件

### 2.5 HORIZON：仓库级 agent 的状态管理启发

`HORIZON` 虽然更偏 agentic hardware design，但它给了你一个很重要的工程启发：**把硬件任务当成 repository-level code evolution，而不是单次文本生成**。[HORIZON](https://arxiv.org/abs/2606.28279)

它强调：

- 用 repo/worktree 管理状态
- 用 executable evaluator 管正确性
- 用 acceptance predicate 管收敛
- 用 repository operations 管 tracing/replay

这对你的迁移价值不是“让 CodeX 去自动改 RTL”，而是：

- 你也应该把验证知识、激励、反馈、成功 recipe 当成 repo 资产
- 每一轮生成和执行都应该可追踪、可回放、可比较

### 2.6 Search-First / Knowledge-Base 路线

在大仓库场景下，单次上下文窗口永远不够，所以必须建设：

- repo 索引
- 模块卡片
- recipe 库
- bug/coverage 知识页
- token budget 管理

这类思路本身不是 RTL 专用，但对 RTL 尤其重要，因为芯片设计知识的上下文跨度比普通应用代码更大：

- 文件层
- 模块层
- 实例层
- 总线层
- 寄存器层
- 时钟/复位域
- 软件/硬件接口层

## 3. 问题重述：你真正要解决的是什么

你的问题可以拆成四个子问题：

1. **仓库级大上下文怎么管**
2. **系统级 C 激励怎么保证合法**
3. **怎么从核级动作触发到特定内部模块**
4. **怎么根据反馈不断补激励**

这四件事分别对应四层系统能力：

- 仓库理解层
- 合法性控制层
- 路径触达层
- 闭环优化层

## 4. 方案总览

建议把整个系统设计成下面这条主链路：

`RTL repo -> 结构化索引/知识库 -> 目标模块/覆盖点检索 -> 路径切片 -> 软件动作 DSL -> C 激励生成 -> 仿真/协同验证 -> 覆盖率/波形/失败摘要 -> 再检索/再补激励`

其中，真正进入 LLM 上下文的内容不应该是“整个 repo”，而应该是：

- 当前目标
- 相关模块卡片
- 相关寄存器和协议规则
- 一两条候选触达路径
- 相关 RTL 切片
- 最近一次执行反馈摘要

## 5. 仓库理解层

### 5.1 先建四层知识，而不是直接喂代码

建议离线构建四层知识表示。

**第一层：仓库级结构索引**

- module/interface/package 列表
- 文件到模块映射
- 实例化关系
- include/依赖关系
- 时钟/复位域
- 总线拓扑
- 中断/DMA/内存映射拓扑

**第二层：模块级摘要卡片**

每个关键模块维护一张卡片，至少包含：

- 模块职责
- 父实例路径
- 上下游模块
- 对外接口
- 关键寄存器
- 关键状态机
- 可观察信号
- 触发入口
- 前置条件
- 命中证据

**第三层：路径级 recipe / 切片**

这里是最关键的层。它回答：

- 从哪个软件动作可以打到这个模块
- 中间经过哪些寄存器/总线/状态机
- 哪些前置条件必不可少

**第四层：验证知识层**

- coverage point
- bug pattern
- known failure mode
- 成功激励案例
- wave/log signature

### 5.2 上下文检索单位要从“文件”升级成“对象”

在这个场景里，上下文检索单位不应该是文件，而应该是：

- 模块
- 接口/协议
- 寄存器块
- 路径片段
- coverage 点
- recipe

这样做的好处是：

- token 利用率高
- 更接近验证工程师的思维方式
- 更容易做可解释检索

### 5.3 每次任务只组装一个“小而准”的上下文包

建议每轮 prompt 的任务上下文包固定包含：

1. 当前目标
2. 目标模块卡片
3. 1-3 条候选触达路径
4. 相关寄存器/协议规则
5. 必要的 RTL 切片
6. 最近一次 feedback 摘要

不要把全量波形、全量覆盖报告、全量源码直接塞进上下文。

## 6. 合法性控制层

### 6.1 主生成粒度必须提升到“软件动作”

你的核心担忧是对的：如果让模型自由生成底层信号，它很难稳定保证合法性。

因此建议明确采用这条原则：

- **主路径：软件动作级刺激**
- **辅助路径：有限的信号级故障注入**

软件动作原语可以包括：

- `write_reg`
- `read_reg`
- `set_bitfield`
- `prepare_buffer`
- `build_descriptor`
- `flush_cache`
- `ring_doorbell`
- `enable_irq`
- `wait_irq`
- `poll_status`
- `inject_memory_pressure`

### 6.2 用 DSL 而不是直接自由写 C

推荐流程：

- LLM 先输出一个结构化 `stimulus DSL`
- 模板引擎把 DSL 编译成 `C`
- 校验器检查 DSL 和生成代码是否合法

一个最小 DSL 例子：

```text
init_platform()
reset_ip(dma0)
alloc_buffer(name=src, size=4096, align=64)
alloc_buffer(name=dst, size=4096, align=64)
write_reg(addr=DMA_CTRL, fields={EN:1, IRQ_EN:1})
program_desc(channel=0, src=src, dst=dst, len=1024)
ring_doorbell(channel=0)
wait_irq(name=DMA_DONE, timeout=1000)
check_reg(addr=DMA_STATUS, fields={DONE:1, ERR:0})
```

### 6.3 合法性靠四层保障

**第一层：寄存器/协议规则库**

- 地址合法范围
- bitfield 取值约束
- 写前依赖
- reset 后初值
- 状态机前置条件
- 资源互斥条件

**第二层：模板/DSL**

- 缩小生成空间
- 让 LLM 只选动作组合和参数

**第三层：静态检查**

- 顺序检查
- 参数边界检查
- buffer 对齐/大小检查
- 状态依赖检查

**第四层：动态检查**

- assertion/checker
- error code
- trap/exception
- protocol violation log
- timeout

## 7. 目标模块触达层

### 7.1 这是可达性分析，不是“灵感生成”

你问“给核的激励怎么触发到某个特定模块”，本质上是在问：

- 从软件可见入口到目标模块，有没有一条可达路径
- 这条路径的前置条件是什么
- 如何证明真的打到了

因此建议为每个关键模块维护一张 `activation recipe`。

### 7.2 Activation Recipe 模板

推荐字段：

- 模块名
- 实例路径
- 功能描述
- 软件入口
- 依赖资源
- 前置初始化
- 最短触达路径
- 增强触达路径
- 成功命中证据
- 常见失败原因

示例：

```text
模块: dma_engine.u_ch0
软件入口: DMA_CH0 MMIO + descriptor ring
前置条件:
1. 时钟稳定
2. reset 释放
3. global enable 打开
4. 中断使能
5. descriptor/buffer 初始化完成
最短路径:
CPU 写 DESC_BASE -> CPU 写 CTRL.EN -> CPU 写 DOORBELL
-> AXI/AHB/WB 事务到达 reg block
-> channel arbiter grant
-> descriptor fetch
-> DMA FSM 进入 ACTIVE
-> data mover 启动
命中证据:
1. code coverage 命中 ACTIVE 分支
2. waveform 命中 desc_fetch_valid / data_move_start
3. DONE interrupt 触发
4. STATUS.DONE=1 且 ERR=0
常见失败:
1. descriptor 未对齐
2. IRQ mask 未打开
3. reset 后未完成 global init
```

### 7.3 用路径切片控制上下文

对于复杂模块，不要把整片 RTL 都交给 CodeX。

只给它：

- 目标模块
- 上游关键寄存器块
- 总线/互连中相关节点
- 关键状态机
- 必要的跨模块信号锥

这就是把“仓库级超长上下文”转成“路径级小上下文”的关键技术点。

## 8. 闭环优化层

### 8.1 一轮生成不可能直接最优

建议把每次执行都看成一次采样，而不是最终答案。

每轮闭环：

1. 生成场景/DSL
2. 生成 C 激励
3. 执行仿真或协同验证
4. 收集反馈
5. 归因分析
6. 补充刺激或修正路径

### 8.2 反馈必须先摘要，再交给 CodeX

不要把原始数据直接扔给 LLM。

应该先提炼成结构化摘要：

- 未命中的 coverage 点
- 关键状态机停留状态
- 哪个路径节点没被满足
- 哪条 assertion 失败
- 哪个寄存器动作无效
- 哪个中断未发生

建议统一一个反馈 schema：

- `coverage_holes`
- `path_reached`
- `path_blocked_at`
- `assertions_failed`
- `timeout_stage`
- `unexpected_register_state`
- `waveform_signatures`

### 8.3 经验沉淀是长期价值来源

每次闭环后，把下面这些资产写回知识库：

- 成功 recipe
- 失败 pattern
- coverage 补洞经验
- 常见初始化序列
- 特定模块的高价值场景

长远看，这些资产比单轮 prompt 更值钱。

## 9. 推荐的 CodeX 系统分工

不要把 CodeX 当成一个单 agent。

建议拆成四个逻辑角色：

**1. Repo Indexer**

- 扫 RTL 仓库
- 建模块图/实例图/寄存器图/总线图

**2. Path Finder**

- 面向目标模块找“软件动作 -> 模块”路径
- 输出 activation recipe

**3. Stimulus Planner**

- 基于 coverage 目标和 recipe 生成 DSL 级场景

**4. Critic / Refiner**

- 读取覆盖率/波形/失败摘要
- 判断下一轮补什么

这种拆法的好处是：

- 每个 agent 的上下文都小
- 更容易做审计和可解释
- 更接近真实验证流水线

## 10. 推荐落地路线

### Phase 0：最小知识底座

先不要自动生成激励，先把以下资产做出来：

- 模块树
- 实例关系图
- 寄存器地图
- 总线/中断/DMA 拓扑
- 5-10 个关键模块的 activation recipe

这一步完成后，CodeX 至少可以做：

- 解释模块作用
- 解释从软件到模块的路径
- 起草测试意图

### Phase 1：受约束生成

引入最小 stimulus DSL 与模板化 `C` 生成。

此时 LLM 只负责：

- 测试目标
- 参数组合
- 场景意图
- 预期命中模块
- 预期 coverage 点

代码生成器负责：

- 生成 `C`
- 填模板
- 做静态检查

### Phase 2：反馈闭环

把下面三类数据接回来：

- coverage
- 波形摘要
- 失败信息

实现：

- gap triage
- path blockage analysis
- 有针对性的 testcase supplement

### Phase 3：仓库级知识化

建立可搜索的资产库：

- module cards
- recipe 库
- bug pattern 库
- scenario 库
- coverage remediation 库

这时 CodeX 的能力才会从“临时问答”进化成“持续积累”。

### Phase 4：局部自动化扩展

不要一开始就覆盖整个 SoC。

优先选择：

- 寄存器可控
- 路径清晰
- 命中证据明确
- 反馈容易量化

的模块先做，例如：

- DMA controller
- interrupt controller
- bus bridge
- power/control block
- config/register-heavy IP

## 11. 风险与边界

### 11.1 不要高估“纯自然语言理解 RTL”能力

即使模型很强，没有结构化索引、路径切片和规则库，它也很难可靠理解大规模 RTL 仓库。

### 11.2 不要高估“自由生成信号刺激”的可用性

自由信号生成很适合 demo，不适合做稳定、合法、可审计的系统级验证主路径。

### 11.3 波形分析一定要做前处理

如果把原始波形全文交给 LLM，不仅成本高，而且很难稳定得到有效结果。

### 11.4 先做高价值模块，不要追求全覆盖

一开始先做少量关键模块的 recipe 和闭环，比试图一次建全套平台更现实。

## 12. 最终建议

如果只给一句建议，那就是：

**把 CodeX 当成“基于知识检索与反馈闭环的验证规划器”，而不是“直接读完整仓库并自由编造激励的代码生成器”。**

更具体一点：

- 用结构化知识库解决大上下文问题
- 用软件动作 DSL 解决合法性问题
- 用 activation recipe 解决模块触达问题
- 用 coverage/波形/失败摘要闭环解决持续优化问题

## 13. 我建议你下一步先做的 5 件事

1. 选 5-10 个关键模块，手工整理第一版 `activation recipe`
2. 抽取寄存器地图和 bitfield 规则，形成可校验规则库
3. 设计一个最小的软件刺激 DSL
4. 统一 coverage/波形/失败摘要 schema
5. 让 CodeX 先只做“路径分析 + DSL 级测试规划”，不要直接自由写底层信号

## 14. 参考资料

- UVM2: [From Concept to Practice: an Automated LLM-aided UVM Machine for RTL Verification](https://arxiv.org/html/2504.19959v3)
- UVMarvel: [UVMarvel: an Automated LLM-aided UVM Machine for Subsystem-level RTL Verification](https://arxiv.org/html/2605.04704v2)
- HAVEN: [HAVEN: Hybrid Automated Verification ENgine for UVM Testbench Synthesis with LLMs](https://arxiv.org/abs/2604.27643v1)
- FireBridge: [FireBridge: Cycle-Accurate Hardware + Firmware Co-Verification for Modern Accelerators](https://arxiv.org/html/2603.25969)
- HORIZON: [Agentic Hardware Design as Repository-Level Code Evolution](https://arxiv.org/abs/2606.28279)
- Search-First Knowledge Design: [LLM Wiki × DV Agentic System Integration Design Specification](https://anlit75.github.io/dv-agentic-system/llm-wiki-dv-agentic-spec/)
