# vLLM 系列 26 篇文章第五轮复查报告

复查日期：2026-07-20  
审校范围：`contents-en/LLM/vllm/01-*.md` 至 `13-*.md`，以及 `contents/LLM/vllm/01-*.md` 至 `13-*.md`；不含系列计划、配图计划、参考资料、`mt-*` 中间稿和英文第 01 篇另存的 natural edit。  
源码基线：本地 `/Users/bytedance/TikTok/vllm`，commit `6cf7b26bd4bff60bf378e1af14044280ac0d214c`。  
约束：本轮只新增本报告，没有修改任何文章或配图；第 06 篇拆分、P2-6 英文模板化和 P2-7 中文翻译腔继续按作者决定暂缓。

## 总结论

**第四轮列出的多数问题已经修好，但当前还不能判定为整套最终可发布。** 本轮确认 3 组发布前必须修复的问题，以及 2 组正文已改、配图或局部结论未同步的问题：

1. 第 02 篇主体已经正确说明默认 `SyncMPClient`，但结尾 full trace 又写回“`_add_request` 后尚未计算，计算发生在 `_run_engine`，没有 queue/background handler”；两张图也仍把离线默认画成 in-process。
2. 第 03 篇的 `03-05` HWM 图已经修好，中文正文也加了限定，但英文仍保留 “lost message ... is impossible / never dropped”；`03-03`、`03-12` 仍把整类边缘工作标成全程持有 GIL。
3. 第 11 篇多数 caveat 已经正确，但英文第 534、845 行、中文第 846 行和 `11-22` 图仍把 `VLLM_BATCH_INVARIANT` 写成固定归约顺序。
4. 第 08 篇正文已经改成静态 allocation compatibility，但 `08-09`、`08-25` 图仍写成可以无重新分配地切换 backend/transport。
5. 第 13 篇新增的第三方插件 caveat 是正确的，但开头、结尾和两张总览图仍把不同 plugin group 的加载时机、失败行为和惰性导入保证概括成一个统一不变量。

第 07、08 篇的重复小标题已经清理；第四轮 P2-9 可关闭。除下文列出的残留外，本轮没有发现新的坏源码链接、缺图、图片编号错配、中英文代码块漂移或标题与主体明显错位。

## 客观校验结果

- 26 篇目标正文均存在，共 68,682 行。
- 中英文每侧各有 1,312 个 fenced code block；逐篇数量与代码内容对应。本轮按 fence pair 重新计数，当前值是 1,312，而不是第四轮报告中的 1,314。
- 共检查 4,066 次固定 commit 的 GitHub 源码引用；源文件存在，显示路径与 URL 对应，行号未越界。
- 共检查 686 次图片引用，对应 684 个 SVG；图片文件全部存在，`href` 与 `src` 一致，中英文图片编号对应。
- 684 个 SVG 均通过 XML 解析，未发现重复 XML `id`，源文件中的最小文字字号为 14px。
- 当前 26 篇内没有重复 Markdown 标题。
- 对本轮高风险图 `02-03`、`02-12`、`03-03`、`03-05`、`03-12`、`08-09`、`08-25`、`11-22`、`13-01`、`13-11`、`13-13` 进行了文本级复核；其中 `03-03`、`03-05`、`03-12` 另做了渲染抽查。文件和排版正常，剩余问题是图内结论。

## 第四轮问题回归

| 第四轮问题 | 第五轮状态 | 结论 |
|---|---|---|
| P0-1 第 02 篇默认离线拓扑 | **大部分修复，仍有残留** | 开头和主体已改对；结尾 trace 与 `02-03`、`02-12` 仍保留旧模型 |
| P0-2 第 03 篇 GIL / HWM | **部分修复，仍阻断** | `03-05` 已修；英文 HWM 绝对句、`03-03`/`03-12` GIL 分类仍有问题 |
| P0-2 第 03 篇 InprocClient 一线程一把 GIL | **作者选择保留；技术限定仍未满足** | 不是普通漏改，但该结论还需要同时限定 executor；详见下文专节 |
| P0-3 第 11 篇 batch invariant | **大部分修复，仍有残留** | 主结论和后文 caveat 已修；3 处正文和 `11-22` 图仍写“固定归约顺序” |
| P1-4 第 08 篇 hot swap | **正文已修，配图未同步** | 两张图仍把静态兼容性写成运行时切换能力 |
| P1-5 第 13 篇惰性导入范围 | **部分修复，并发现同类残留** | 已承认第三方 callable 可有副作用；统一 loader、统一失败语义等概括仍不成立 |
| P2-6 英文模板化 | **按作者决定暂缓** | 本轮不计为修复失败；当前原版英文仍有 244 处 `Step-by-step read`、265 处 `the invariant` |
| P2-7 中文翻译腔 | **按作者决定暂缓** | 本轮不计为修复失败 |
| P2-9 重复小标题 | **已修复** | 当前未检出重复标题 |
| 第 06 篇拆分 | **按作者决定暂缓** | 本轮不计为修复失败 |

## P0：发布前必须修复

### 1. 第 02 篇结尾 trace 与已经修正的主体相互矛盾

涉及正文：

- `contents-en/LLM/vllm/02-entrypoints-llm-cli-openai-server.md:2478`
- `contents/LLM/vllm/02-entrypoints-llm-cli-openai-server.md:2473`

涉及配图：

- 中英文 `images/vllm-02-03-llm-construction.svg:19`
- 中英文 `images/vllm-02-12-construction-divergence-matrix.svg:19`

第 02 篇开头和构造章节已经正确写明：`LLM.generate()` 对调用者是同步、阻塞的 API，但默认 V1 离线实现使用 `SyncMPClient`，EngineCore 在后台进程中，输出由后台 collector thread 接收。这部分修改有效。

结尾却仍说：

> Nothing has run yet. `_add_request` only queued state. The compute happens in `_run_engine` ... no event loop, no queue, no background handler.

默认 `SyncMPClient.add_request()` 会把请求发给后台 EngineCore；当 frontend 继续渲染和提交后续 prompt 时，EngineCore 已可能开始调度和执行。`_run_engine()` 在默认路径上的职责是阻塞读取并排空输出，不是“计算开始发生”的位置，也不能用它的本地函数体推导整个系统没有 queue 或 background handler。

两张图也仍保留旧结论：

- `02-03` 写 `LLMEngine.__init__ (in-process, sync)` 和 `live in-process LLMEngine`；
- `02-12` 写离线 multiprocessing 是 `opt-in`、`same-process default`，中文版对应“按需开启 / 默认同进程”。

建议统一改为：离线 API 对调用者同步；EngineCore client 的实现由构造配置决定，固定基线默认是 multiprocess `SyncMPClient`。提交请求后后台 EngineCore 可以开始工作，`_run_engine()` 负责同步排空结果并恢复输入顺序。图中应把 `asyncio_mode=False` 与 `multiprocess_mode` 分成两个正交维度。

### 2. 第 03 篇仍把 HWM=0 和 GIL 行为写成绝对保证

涉及英文正文：

- `contents-en/LLM/vllm/03-v1-process-architecture.md:438`
- `contents-en/LLM/vllm/03-v1-process-architecture.md:1956`
- `contents-en/LLM/vllm/03-v1-process-architecture.md:2245`

涉及中文正文：

- `contents/LLM/vllm/03-v1-process-architecture.md:414`
- `contents/LLM/vllm/03-v1-process-architecture.md:437`
- `contents/LLM/vllm/03-v1-process-architecture.md:1955`
- `contents/LLM/vllm/03-v1-process-architecture.md:2244`

涉及配图：

- 中英文 `images/vllm-03-03-process-split.svg:19`
- 中英文 `images/vllm-03-12-gil-escape-hatches.svg:19`

`03-05` 已正确改成 “no configured limit; not end-to-end backpressure”，这一张图可以通过。中文第 437 行也补上了“不等于传输在任何情况下都不会阻塞或失败”，方向正确。

但英文第 438 行仍写 `a lost message under load is impossible because HWM is 0`，第 1956 行写 ready frame `never dropped`，第 2245 行写 `HWM=0 means no silent drop`。中文第 414、1955、2244 行仍有对应绝对表达。ZeroMQ 的 `SNDHWM/RCVHWM=0` 表示没有配置 HWM 上限；它只能排除“因为达到这个 HWM 阈值而触发的行为”，不能证明无限内存、peer 永不失败、shutdown 永不打断，也不是端到端交付保证。官方定义可参考 [`zmq_setsockopt`](https://zeromq.github.io/libzmq/zmq_setsockopt.html)。

`03-03`、`03-12` 仍把 tokenization、chat templating、多模态处理、detokenization 整组标成 “pure Python / holds GIL its whole run”，而正文第 26 行已经正确承认 Rust tokenizer、PyTorch 和媒体 codec 的 native 段可能释放 GIL。图还说独立进程中的 edge-work burst `cannot steal cycles` from forward pass；不同解释器避免共享同一把 GIL，但进程仍会争用 CPU core、内存带宽和 NUMA 资源。

建议：

- 把 HWM 结论限定为“在连接存活且资源可用时，消息不会因为达到已配置的 HWM 阈值而被丢弃/拒绝”；
- 把任务级 GIL 分类改成代码段级分类：Python glue 受 GIL 约束，native tokenizer/codec/PyTorch/CUDA 段可能释放 GIL；
- 把 `cannot steal cycles` 改为“removes same-interpreter GIL contention, while system-resource contention remains”；
- 正文第 46 行“开销会被隐藏在 forward pass 之后”也宜改为“可以与 forward pass 部分重叠”，避免暗示全部隐藏。

### 3. 第 11 篇仍有“固定归约顺序”的旧结论

涉及英文：

- `contents-en/LLM/vllm/11-distributed-inference-parallelism.md:534`
- `contents-en/LLM/vllm/11-distributed-inference-parallelism.md:845`

涉及中文：

- `contents/LLM/vllm/11-distributed-inference-parallelism.md:846`

涉及配图：

- 中英文 `images/vllm-11-22-size-banded-backends.svg:19`

第 1393/1394、1425/1426 和结尾已经采用正确口径：`VLLM_BATCH_INVARIANT` 会关闭若干与 batch shape/order 有关的特定 fast path，并只在文档列出的硬件、模型和版本范围内提供 batch invariance；它不是所有 collective/kernel 都采用一个固定求和顺序的证明。这些修改有效。

但英文第 534 行仍写 `forcing off any reduction-reordering path`，第 845 行仍写 `trading throughput for a fixed summation order`；中文第 846 行仍写“固定求和顺序”。`11-22` 图也写 `hard-off's symm-mem for a fixed reduction order` / “关闭 symm-mem 以固定归约顺序”。这些句子与同篇已经修正的 caveat 自相矛盾。

建议全部统一为：该 flag gate 掉实现中已识别的 batch-shape-dependent path（包括这里的 symmetric-memory 路径），从而在受支持配置上实现文档所述的 batch invariance；不要把机制概括为“固定所有归约顺序”。

## P1：正文与配图需同步、局部结论需收紧

### 4. 第 08 篇正文已修，两个 SVG 仍描述运行时切换

正文以下位置已经改对：

- 英文 `contents-en/LLM/vllm/08-pagedattention-kernels-attention-backends.md:1537,1707,2207`
- 中文 `contents/LLM/vllm/08-pagedattention-kernels-attention-backends.md:1545,1720,2223`

它们现在明确说 backend 在配置/初始化时选择，cache layout 的共同格式只表示静态 allocation compatibility，不表示受支持的 runtime hot swap。这部分可通过。

但配图仍写：

- 中英文 `images/vllm-08-09-backends-compare.svg:19`：`swap those backends with no reallocation` / “后端间切换无需重新分配”；
- 中英文 `images/vllm-08-25-scale-transport-matrix.svg:19`：`switching transport needs no cache reallocation` / “切换传输无需重新分配 cache”。

建议把 `swap/switching` 改成“在共同支持的 dtype、layout、stride 和 quantization 配置下，可采用兼容的静态 cache allocation”；不要暗示运行中的切换 API。图中若仍写 `shared dtypes`，也应加上 layout/stride 条件。

### 5. 第 13 篇把不同 plugin group 概括成统一 loader、统一失败语义

涉及正文：

- 中英文 `13-extending-vllm-models-plugins.md:5`
- 英文 `:70,2202,2252,2317`
- 中文 `:69,2219,2268,2332`

涉及配图：

- 中英文 `images/vllm-13-01-extension-surface.svg:19`
- 中英文 `images/vllm-13-11-extension-points.svg:19`
- 中英文 `images/vllm-13-13-resolve-obj-by-qualname.svg:19`

文章已经正确补充：第三方 entry-point callable 是任意 Python，可能在父进程 import torch 或触碰 CUDA；FQCN 字符串是插件作者应遵守的约定，不是框架可强制的保证。这部分修改有效。

剩余问题有三类：

1. 开头把 IO、logits、platform、stat、endpoint 和 general plugins 都写成“由 `load_general_plugins` 在每个进程执行”。固定源码 `vllm/plugins/__init__.py:16-30,77-88` 明确区分：`load_general_plugins()` 只加载 `vllm.general_plugins`；IO/stat 在 process 0，endpoint 在 API frontend，platform 在 `current_platform` 首次解析处加载。
2. “every lookup is total or loud / 整个扩展面唯一一个有意的 `None`”范围过大。`load_plugins_by_group()` 对 entry-point import 异常会记录日志并跳过；IO factory 可以返回 `None` 动态退出；platform plugin 也可返回 `None` 表示不适用。应只对文中逐个验证过的“显式请求某个名字的 lookup API”陈述 fail-loud 行为，不能推广到 discovery/factory 全流程。
3. `13-13` 图和结尾把 FQCN/lazy import 说成保证父进程不加载 CUDA、且是所有扩展点存在的“唯一原因”。这只适用于采用字符串注册并遵守惰性导入纪律的路径；任意第三方 entry point 仍可能破坏该假设。FQCN 还服务于序列化、解耦和按需解析，不宜压缩成唯一动机。

建议按 plugin group 分别列出“加载位置、是否调用 factory、返回值含义、失败行为”；把 CUDA/fork 安全限定到 vLLM 自有 lazy registry 和遵守字符串注册约定的第三方模型插件。

## 关于作者保留的 InprocClient 图注

这项不是“没有看到作者修改”，而是作者明确选择保留；但按固定源码复核后，当前原句仍需要增加 executor 前提。

`InprocClient` 的确保证：frontend 和 `EngineCore` 位于同一进程，调用者通过 `step_fn()` 驱动 EngineCore，没有 frontend → EngineCore 的 busy-loop/ZMQ 进程边界（`vllm/v1/engine/core_client.py:276-292`）。因此，“同进程”和“EngineCore step 由调用线程驱动”是准确的。

但 `InprocClient` 不决定 worker executor。`Executor.get_class()` 仍会根据 `distributed_executor_backend` 选择 `UniProcExecutor`、`MultiprocExecutor`、Ray 或其他实现（`vllm/v1/executor/abstract.py:48-76`）；只有 `world_size == 1` 且没有另行指定 backend 时才默认 `uni`（`vllm/config/parallel.py:915-916`）。所以关闭 `VLLM_ENABLE_V1_MULTIPROCESSING` 只折叠 frontend ↔ EngineCore 边界，并不普遍保证 tokenizer、scheduler、所有 worker 与 kernel launch 都落在同一线程。

另外，“kernel launch share ONE GIL”容易让读者误以为 CUDA/native 调用全程占有 GIL；实际应区分 Python launch glue 与可能释放 GIL 的 native 段。

建议把 `03-12` 底部改为：

> **InprocClient + UniProcExecutor（例如默认的单 GPU、`world_size=1` 形态）**：frontend 与 EngineCore 共享进程，EngineCore step 由调用线程驱动；Python glue 共享同一解释器 GIL，native tokenizer/PyTorch/CUDA 段可能释放 GIL。若 executor 为 `mp`/Ray，worker 仍在独立进程。

如果图只想描述单 GPU的典型 opt-out 形态，标题或脚注必须明确写出 `UniProcExecutor / world_size=1`；仅有 `InprocClient` 前提还不够。

## 逐篇发布状态

| 篇目 | 状态 | 第五轮结论 |
|---|---|---|
| 01 请求全链路 | **技术上可进入终稿** | 未发现新的明确技术错误；英文原版语言问题按作者决定暂缓 |
| 02 Entrypoints | **阻断** | 结尾 trace 与两张图仍把默认离线拓扑写错 |
| 03 V1 进程架构 | **阻断** | HWM 与 GIL 仍有绝对结论；Inproc 图还缺 executor 限定 |
| 04 EngineCore 生命周期 | **可进入终稿** | 未发现新的明确技术错误 |
| 05 Scheduler | **可进入终稿** | 未发现新的明确技术错误 |
| 06 KV Cache Manager | **内容可继续校对** | 未发现新的硬错误；拆分按作者决定暂缓 |
| 07 Prefix Caching | **可进入终稿** | 重复标题已修；本轮未发现新的硬错误 |
| 08 Attention Backends | **需同步两张图** | 正文已修，图仍暗示 runtime switch |
| 09 GPUModelRunner | **可进入终稿** | 未发现新的明确技术错误 |
| 10 Sampling | **可进入终稿** | 未发现新的明确技术错误 |
| 11 Distributed Inference | **阻断** | 旧的 fixed-reduction-order 表述仍存在 |
| 12 Speculative Decoding | **可进入终稿** | 未发现新的明确技术错误 |
| 13 Extending vLLM | **需局部修复** | plugin group、失败语义和 lazy-import 范围仍被统一化 |

## 最终修复清单

1. 第 02 篇改结尾 trace，并同步 `02-03`、`02-12`。
2. 第 03 篇收紧 3 处英文和 4 处中文 HWM 结论；同步 `03-03`、`03-12`；给 InprocClient 增加 `UniProcExecutor/world_size=1` 前提。
3. 第 11 篇删除 3 处“固定归约顺序”的旧句，并同步 `11-22`。
4. 第 08 篇把 `08-09`、`08-25` 从 runtime switching 改成 configuration-time allocation compatibility。
5. 第 13 篇按 plugin group 拆开 loader/进程/失败语义，收窄 `13-01`、`13-11`、`13-13` 和结尾的统一保证。
6. 完成后再跑一次全量链接、代码块、图片引用和 SVG 解析回归；技术内容通过后，再决定是否处理 P2-6/P2-7 的纯语言终校。

完成以上 5 组技术/图文同步修改后，这套文章才适合标记为最终技术版；P2-6/P2-7 如果继续暂缓，应把它们视为已知编辑债务，而不是技术发布阻断项。
