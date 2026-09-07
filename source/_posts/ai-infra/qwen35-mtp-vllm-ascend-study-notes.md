---
title: Qwen3.5 MTP 与 vLLM Ascend 投机解码：从启动调试到 KV Cache 的完整复习笔记
permalink: posts/qwen35-mtp-vllm-ascend-study-notes/
tags:
  - AI Infra
  - vLLM
  - Ascend
  - Qwen3.5
  - MTP
  - 投机解码
  - KV Cache
  - PyTorch
categories:
  - AI Infra
description: >-
  整理一次 Qwen3.5 MTP 源码调试会话的全部问题，串起启动配置、draft 与验证、随机拒绝采样、张量索引、请求重排，以及 full
  attention 缓存的分配、共享和生命周期，并附断点清单与复习自测。
toc: true
toc_number: false
katex: false
mathjax: false
mermaid: false
top_img: false
abbrlink: 18297
date: 2026-09-07 10:00:00
updated: 2026-09-07 10:00:00
---

这篇笔记来自一次围绕 **Qwen3.5-35B-A3B、vLLM V1 runner、Ascend NPU、TP=2、MTP K=3** 的源码阅读与调试。目标是把“一个 draft token 从哪里来、如何验证、缓存如何继续使用”串成完整流程，供以后反复复习。

文中将重复提问合并到同一主题，末尾保留问题索引、自测题和断点清单。代码片段分为两类：标为“教学代码”的片段用于说明算法；带具体函数名的片段对应会话中阅读的实现，省略了部分平台和异常分支。

**版本边界：** 会话中的 Ascend 分支包含本地适配，不能把所有行为当作任意 vLLM 版本的保证。整理时本地 checkout 的 HEAD 短标识为 vLLM `12b9573c98`、vLLM Ascend `d2e61d9ae`；工作区还可能有未提交修改，这两个标识不是完整的运行环境快照。特别是 `KVCacheTensor.shared_by` 与新版 `layers/stride` 已有接口差异，本文分别说明。实际 NPU 环境的安装路径、版本和运行时 metadata 优先。

<!-- more -->

<a id="quick-reference"></a>

## 1. 先记住这些结论

| 问题 | 复习答案 |
| --- | --- |
| `num_speculative_tokens=3` 一次验证最多输出多少？ | 标准链式投机最多 **4 个新 token**：3 个接受的草稿加 1 个 bonus。 |
| 中间拒绝后还能保留后面的草稿吗？ | 只能保留第一个拒绝之前的连续前缀，再输出 1 个 recovered token。 |
| 验证 d1 用哪一行概率？ | 用输入 d1 前一个 token 的 target logits；自回归 logits 预测的是下一 token。 |
| `draft_probs=None` 表示没有开启投机吗？ | 不表示。在本文路径里，它可以代表确定性 proposal，按 one-hot 分布处理。 |
| random sample 的 `argmax` 是贪心吗？ | 对原概率直接 argmax 是贪心；对“概率 / 指数随机噪声”argmax 是随机采样。 |
| 拒绝采样逐 token 执行 Python 循环吗？ | PyTorch 路径可并行计算，再通过第一个拒绝位置屏蔽后缀。 |
| MTP 的 3 个 draft token 对应 3 个 MTP 层吗？ | 本例只有 1 个 MTP 层，连续调用来生成 3 个草稿。 |
| 验证会清空 MTP KV cache 吗？ | 验证采样不清空整份缓存；有效前缀继续用，重算位置覆盖，无效后缀不应再被读取。 |
| MTP 与主模型 full attention 同组就共用 K/V 数值吗？ | 同组共用块管理规则，每层仍有自己的逻辑 K/V。 |
| MTP 的 `shared_by` 是什么？ | 本例旧版 11 full + 30 linear 分组推导下，MTP 那个 buffer 的列表只有自身。 |

统一记号：`B` 是请求数，`H` 是隐藏维度，`V` 是词表大小，`K` 是本轮草稿数；`p_i` 是 target 分布，`q_i` 是 draft 分布；`x` 是此前已经采样的 token，`d1…dK` 是本轮草稿，`r` 是拒绝后的替换 token，`b` 是全部接受后的 bonus token。

<a id="launch"></a>

## 2. VS Code 启动 MTP，以及“为什么没走投机”

### 2.1 在启动参数中加入 speculative-config

原有启动方式使用 `vllm.entrypoints.openai.api_server`，可在 `args` 中加入：

```json
"--speculative-config",
"{\"method\":\"mtp\",\"num_speculative_tokens\":3}"
```

下面是整理后的 `launch.json` 示例。模块名和环境变量里的下划线无需添加反斜杠；JSON 字符串内部的双引号需要转义。

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Qwen3.5 MTP K=3",
      "type": "debugpy",
      "request": "launch",
      "module": "vllm.entrypoints.openai.api_server",
      "console": "integratedTerminal",
      "subProcess": true,
      "env": {
        "VLLM_USE_V1": "1",
        "ASCEND_RT_VISIBLE_DEVICES": "0,1"
      },
      "cwd": "/",
      "args": [
        "--model", "/mnt/weight/Qwen3.5-35B-A3B",
        "--tensor-parallel-size", "2",
        "--speculative-config", "{\"method\":\"mtp\",\"num_speculative_tokens\":3}",
        "--no-enable-prefix-caching",
        "--enforce-eager",
        "--max-num-batched-tokens", "4096",
        "--port", "8555",
        "--max-model-len", "4096",
        "--no-enable-chunked-prefill"
      ],
      "justMyCode": false
    }
  ]
}
```

这里的 `/` 和模型目录指运行服务的 Linux/容器环境，不是保存这篇博客的 Windows 环境。`subProcess` 有助于跟踪子进程，但需要确认实际命中的是 engine/worker；不同启动器不一定都能自动附加。保留这些参数是为了对照本次调试，并非所有版本和模型的通用性能配置。

`--enforce-eager` 便于观察 Python 执行路径，它本身不代表开启 MTP。`--no-enable-prefix-caching` 关闭跨请求前缀缓存复用，也不代表取消当前请求 decode 所需的 KV cache。配置方式可对照 [Ascend 投机解码官方文档](https://docs.vllm.ai/projects/ascend/en/latest/user_guide/feature_guide/speculative_decoding.html)。

### 2.2 `spec_decode_metadata` 为空，不一定异常

这份 metadata 描述的是“当前这轮待验证的草稿”，不是全局的“已启用 MTP”开关。

```text
首次 prefill / 尚无上一轮草稿
    → 当前 spec_decode_metadata 可以是 None
    → target 采样出 x
    → MTP 首次生成 d1、d2、d3

后续调度真正带上这些草稿
    → 当前验证轮构建 SpecDecodeMetadata
```

动态 K=0、请求尚在 prefill、接近长度限制、没有为该请求调度草稿等，也可能让某一轮没有验证 metadata。不能仅在首次 forward 打断点，就断定投机没启动。

如果连续 decode 多轮都没有进入投机，可以按下面的顺序检查：

| 检查点 | 要确认的事实 |
| --- | --- |
| `self.speculative_config` | 是否存在，`method` 和 K 是否符合配置。 |
| `self.drafter` | MTP proposer 是否创建成功，权重是否加载。 |
| 运行文件和进程 | `vllm.__file__`、`vllm_ascend.__file__`、PID、解释器是否对应正在阅读的源码。 |
| `propose_draft_token_ids()` | 是否调用，返回草稿的形状与内容是什么。 |
| `self._draft_token_ids` | 本轮草稿是否保存，下一轮是否仍可取到。 |
| `scheduler_output.scheduled_spec_decode_tokens` | 当前请求是否实际调度了草稿。 |
| `_calc_spec_decode_metadata()` | 是否创建验证索引。 |
| `_sample()` | 是否进入 rejection sampler 分支。 |

本次讨论没有得到“某个特定参数就是未走投机的根因”的运行证据，因此这部分是排查路线，不应记成一个已经验证的故障结论。

<a id="timeline"></a>

## 3. 一轮 draft、验证、采样如何串起来

```text
target forward
    ↓
普通采样，或验证上一轮草稿并采样
    ↓
得到有效输出 + 最后一个有效 token
    ↓
准备 MTP 的 input_ids、positions、hidden_states、metadata
    ↓
MTP 第一次 forward → d1
    ↓
MTP 后续 forward   → d2 → d3
    ↓
保存 draft token IDs / 按需保存 draft probabilities
    ↓
下一轮 scheduler 调度草稿
    ↓
组装 target 输入 [x, d1, d2, d3]
    ↓
target 一次 forward 得到验证概率
    ↓
rejection sampler 输出接受前缀 + recovered 或 bonus
```

验证包含两个环节：**target forward 提供概率；rejection sampler 作出接受/拒绝决定。** 在代码中找“验证在哪调用”，需要把这两个位置都找到。草稿模型不会自行决定自己的 token 已经通过验证。

### 3.1 K=3 为什么最多输出 4 个，而不是 5 个

在不考虑停止条件截断的标准链式算法中：

| 验证结果 | 本轮新输出 | 数量 |
| --- | --- | ---: |
| d1 拒绝 | r | 1 |
| d1 接受，d2 拒绝 | d1、r | 2 |
| d1、d2 接受，d3 拒绝 | d1、d2、r | 3 |
| 全部接受 | d1、d2、d3、b | 4 |

`x` 是这轮验证开始前已经采样出来、作为输入送入 target 的 token。把 `x` 再加进去得到 5，是把不同轮次的输出重复计数。实际还可能因为 EOS、stop 或输出长度上限而少于表中的数量。

### 3.2 Bonus token 从哪里来

target 输入包含 K 个草稿以及前面的 `x`，所以可以产生 K+1 行下一 token 的 logits。全部草稿接受时，最后一行分布还可以采样一个新 token，这就是 bonus；不需要为这一个 bonus 再做一次 target forward。

只要出现第一次拒绝，就改用该拒绝位置的 recovered token，后面的 bonus 不再作为本轮有效输出。

<a id="logits-alignment"></a>

## 4. 验证 d1、d2、d3，究竟用哪几个概率

自回归模型的关键对齐关系是：**输入位置 i 的 logits，预测位置 i+1 的 token。**

| 本次 target 输入行 | 该行产生的分布 | 用途 |
| --- | --- | --- |
| x | p1 = P(next \| prefix, x) | 读取 p1(d1)，验证 d1 |
| d1 | p2 = P(next \| prefix, x, d1) | 读取 p2(d2)，验证 d2 |
| d2 | p3 = P(next \| prefix, x, d1, d2) | 读取 p3(d3)，验证 d3 |
| d3 | p4 = P(next \| prefix, x, d1, d2, d3) | 全接受时采样 bonus |

因果 attention 允许把这些位置打包进一次 forward，每个位置只读取自己的因果前缀。若 d2 被拒绝，p3、p4 是在包含错误草稿的条件下算出的，不能继续用于这条有效输出序列。

`SpecDecodeMetadata` 中几类索引的角色：

| 字段 | 含义 |
| --- | --- |
| `logits_indices` | 从打包的 target hidden states 中挑出需要计算 logits 的行。 |
| `target_logits_indices` | 在挑出的 logits 中，选出与各个 draft token 对应的验证行。 |
| `bonus_logits_indices` | 选出每个请求的 bonus 行。 |
| `cu_num_draft_tokens` | 每个请求草稿数的累积和，用于定位展平数组中的请求边界。 |
| `draft_token_ids` | 与这些验证概率对齐的候选 token ID。 |

例如两个请求的草稿长度为 `[3, 2]`：

```python
cu_num_draft_tokens = [3, 5]
# 请求 A 的草稿在展平下标 [0, 3)
# 请求 B 的草稿在展平下标 [3, 5)
```

<a id="hidden-states"></a>

## 5. hidden_states、sample_hidden_states 和 cat

### 5.1 两类 hidden states 的区别

`hidden_states` 保存 forward 计算出的多行结果，包括当前各请求需要计算的 token。`sample_hidden_states` 是其中挑选出来、用于计算采样 logits 的子集。

```python
# 示意，具体索引取决于 target 或 drafter 路径
sample_hidden_states = hidden_states[token_indices_to_sample]
logits = model.compute_logits(sample_hidden_states)
```

例如打包输入为 `[A1, A2, A3, B1, B2]`，若每个请求只取末尾一行，采样索引是 `[2, 4]`。在 target 投机验证中，一个请求可能需要 K+1 行 logits；在 MTP 的某一步，通常每个请求取一行。因此不要把 `sample_hidden_states` 的行数永远等同于请求数。

`aux_hidden_states` 则是某些 drafter 需要的额外层特征，不等同于最终 hidden states。是否使用和如何拼接，取决于模型与 proposer。

### 5.2 MTP 为什么做 torch.cat

会话中的 Qwen3.5 Ascend patch 使用以下融合过程：

```python
inputs_embeds = self.embed_input_ids(input_ids)
inputs_embeds = self.pre_fc_norm_embedding(inputs_embeds)
hidden_states = self.pre_fc_norm_hidden(hidden_states)
hidden_states = torch.cat([inputs_embeds, hidden_states], dim=-1)
hidden_states = self.fc(hidden_states)
residual = None
```

| 语句 | 作用与形状 |
| --- | --- |
| `embed_input_ids` | 将离散 token ID 转为 `[T, H]` 向量。 |
| 两个 norm | 分别规范 token embedding 与传入 hidden state 的数值尺度。 |
| `cat(..., dim=-1)` | 沿特征维拼接：`[T,H] + [T,H] → [T,2H]`，token 行数不变。 |
| `fc` | 学习两路特征的融合，将 `[T,2H]` 映射回 `[T,H]`。 |
| `residual=None` | 初始化此预测模块进入 decoder 时的 residual 状态，不是清除 KV cache。 |

两路输入分别提供“具体是哪一个 token”以及“此前网络已经提取了什么上下文”。直接相加会约束两路信息必须在相同特征坐标上融合；拼接再线性投影允许模型分别学习两路权重。实际架构由训练好的权重决定，不能仅凭形状相同就把 cat 改成相加。

本文引用的 patch 还调整了 PP 行为：本地 drafter 位于最后一个 PP stage 时仍融合 embedding 和传入 hidden states。上游其他版本可能保留不同的 `intermediate_tensors` 分支；这是适配差异，不是所有版本的统一 forward。

<a id="draft-inputs"></a>

## 6. 为什么第一次放末尾，后续放 input_ids 开头

### 6.1 第一次：处理一段 token，需要向左错位对齐

假设 target 这一段输入与输出是：

```text
target input_ids:     [x1, x2, x3]
target hidden:        [h1, h2, h3]
target positions:    [ 0,  1,  2]
新采样 token:         x4
```

MTP 用“上一位置的 hidden state + 下一 token 的 embedding”继续预测，因此准备成：

```text
MTP input_ids:        [x2, x3, x4]
MTP hidden:           [h1, h2, h3]
MTP positions:       [ 0,  1,  2]
用于生成 d1 的行：             ↑
```

核心操作是：

```python
self.input_ids[:num_tokens - 1] = target_token_ids[1:]
self.input_ids[token_indices_to_sample] = next_token_ids
```

新 token 补在**每个请求的有效采样行**。普通情况是该请求末尾；padded 情况则是排除拒绝后缀后选出的有效行，不能直接用整个张量的最后一格。跨请求的左移边界，也靠每个请求的对应位置写入来修正。

这里 positions 延续传入 hidden state 的对齐位置，所以不能看见 `x4` 就擅自把该行位置改成 3。

### 6.2 后续：每个请求只需继续计算一个 token

第一步生成 d1 后，各请求只保留一个继续预测所需的 hidden state。后续输入的有效行压到预分配 buffer 前部：

```python
self.input_ids[:batch_size] = previous_draft_ids
self.hidden_states[:batch_size] = selected_hidden_states
self._set_positions(batch_size, next_positions)
```

一个请求时看起来就是放到下标 0；多个请求时则是 `[d1_A, d1_B, ...]` 占据前 B 行。**内存行号 0 不表示序列位置 0。** 真实位置由 positions 和 attention metadata 描述；图执行还可能保留额外 padding 行。

### 6.3 positions 如何作用于输入

positions 不改变 token ID，也不负责 embedding 查表本身。Qwen 的相关 attention 路径用它索引或计算 RoPE，使 Q/K 带有位置信息：

```text
input_ids → token embedding → 融合/投影 → Q、K、V
positions ───────────────────→ 对 Q、K 应用 RoPE
```

缓存写入也要使用正确的绝对位置，典型槽位关系是：

```python
block_number = position // block_size
block_id = block_table[request_row, block_number]
slot = block_id * block_size + position % block_size
```

buffer 行号、序列 position、物理 slot 是三个不同的量。M-RoPE 的 positions 还可能是多轴张量；slot 通常依据实现指定的序列位置轴计算，不能不看代码就对所有轴套同一公式。

<a id="merged-draft"></a>

## 7. _run_merged_draft 与 K 等于 1 的分支

`_run_merged_draft()` 把一轮草稿的第一次 forward、后续自回归步骤和采样组织起来。函数名字中的 merged 不表示 d1、d2、d3 没有依赖、一次同时生成；普通自回归 MTP 仍然依次消费前一步的 token。

| 阶段 | 主要操作 | 为什么需要 |
| --- | --- | --- |
| 清理本轮输出槽 | `_last_draft_probs = None` | 防止上一轮的概率被误认为本轮结果。 |
| 准备 step 0 | 读取 input IDs、positions、hidden states | 第一遍可能覆盖一段有效输入。 |
| 第一次 forward | `self.model(**model_kwargs)` | 更新本轮涉及位置的 MTP K/V，得到 hidden states。 |
| 选择采样行 | `last_hidden_states[token_indices_to_sample]` | 每个请求从正确位置产生 d1。 |
| 生成 d1 | logits → draft sampling | 是否保留概率取决于配置。 |
| 单步提前返回 | `K == 1` | d1 已足够，无需准备后续循环。 |
| 准备后续步骤 | 收紧 hidden states，递增 positions | 每个请求只继续一条有效分支。 |
| 自回归循环 | 前一 draft ID 作为输入，切换对应 metadata | 依次生成 d2、d3。 |
| 整理输出 | token IDs 转为 `[B,K]`，概率按需堆叠 | 供下一轮验证使用。 |

简化后的教学代码：

```python
h = mtp(first_ids, first_positions, first_hidden)
h_selected = h[token_indices_to_sample]
d1 = sample(compute_logits(h_selected))

if K == 1:
    return d1[:, None]

drafts = [d1]
for step in range(1, K):
    positions = positions + 1
    # 实际实现还切换 slot_mapping、seq_lens 和 graph metadata
    h_selected = mtp(drafts[-1], positions, h_selected)
    drafts.append(sample(compute_logits(h_selected)))

return torch.stack(drafts, dim=1)
```

实际函数还有 TP gather、sequence parallel、graph padding、不同模型返回 tuple 等分支，阅读时先识别上面这条主线。`parallel_drafting` 也可能提前返回，但属于另一种生成方式，不能用它解释普通 MTP 的递归步骤。

<a id="prepare-next"></a>

## 8. prepare_next_token_ids_padded 逐句理解

这个函数从验证输出中，为每个请求选出**最后一个有效 token**，作为下一轮草稿的衔接输入，并返回有效 token 数。无有效采样的请求使用已有请求状态中的备用 token。

示例输入，假设 token ID 都小于词表大小：

```python
sampled_token_ids = torch.tensor([
    [11, 12, 13, 14],
    [21, 29, -1, -1],
    [-1, -1, -1, -1],
])
# 下一轮需要：[14, 29, backup_for_request_2]
# 有效数为： [ 4,  2, 0]
```

### 8.1 预先准备备用 token

```python
seq_lens_list = (gpu_input_batch.num_tokens_no_spec[:num_reqs] - 1).tolist()
self.backup_next_token_ids.np[:num_reqs] = np.array([
    requests[gpu_input_batch.req_ids[i]].get_token_id(seq_lens_list[i])
    for i in range(num_reqs)
])
self.backup_next_token_ids.copy_to_gpu(num_reqs)
```

这里通过请求 ID 和最后一个非投机 token 的位置读取备用值。`num_tokens_no_spec` 在这条路径是 CPU/NumPy 侧状态，不能因为变量名称里有 gpu 就认定 `.tolist()` 必然进行 D2H；若换成设备张量，情况就不同了。

### 8.2 将本来就不该采样的请求整行标为无效

```python
valid_sampled_token_ids_gpu = sampled_token_ids.clone()
valid_sampled_token_ids_gpu = DeviceOperator.index_fill(
    valid_sampled_token_ids_gpu, 0,
    discard_request_indices[:num_discarded_requests], -1,
)
```

例如 chunked prefill 尚未结束的请求，本轮不能把其暂时采出的 token 当作输出。clone 避免修改原始采样结果；index_fill 沿第 0 维处理整行。

### 8.3 统计有效数，选择最后一项

```python
valid_mask = (
    (valid_sampled_token_ids_gpu != -1)
    & (valid_sampled_token_ids_gpu < gpu_input_batch.vocab_size)
)
valid_count = valid_mask.sum(dim=1)
last_valid_indices = valid_count - 1
safe_indices = torch.clamp(last_valid_indices, min=0)
selected = torch.gather(
    valid_sampled_token_ids_gpu, 1, safe_indices.unsqueeze(1)
).squeeze(1)
```

`sum(dim=1)` 从 `[B,K+1]` 变成 `[B]`。`unsqueeze(1)` 把索引变为 `[B,1]`，使 gather 为每行选出一个值；squeeze 再恢复 `[B]`。

**这里依赖有效 token 连续位于行前部的契约。** `[21,29,-1,-1]` 可以用 `count-1` 定位末尾，`[21,-1,29,-1]` 则不可以。代码也不是任意负数输入的通用过滤器：该路径按约定使用 `-1` 作为无效标记。

### 8.4 没有有效 token 时采用备用值

```python
next_token_ids = torch.where(
    last_valid_indices != -1,
    selected,
    self.backup_next_token_ids.gpu[:batch_size],
)
return next_token_ids, valid_count
```

全无效行的 gather 仍会先安全地读取第 0 列，但 where 最终选的是 backup。无须逐请求 `.item()` 回到 CPU 决策。注意这里的有效数包含 recovered/bonus，不能直接当成“接受的 draft 数”。

<a id="sampling"></a>

## 9. random sample：为什么 argmax 也能实现随机抽样

### 9.1 从概率分布抽一个 token 的含义

假设词表只有 A、B、C，概率为 `[0.1, 0.6, 0.3]`。反复抽样时，各 token 的长期频率分别趋近 10%、60%、30%；单次并不一定选概率最大的 B。

教学上可将 `[0,1)` 分为三段：

```text
A: [0.0, 0.1)
B: [0.1, 0.7)
C: [0.7, 1.0)
```

均匀随机数 0.82 落在 C 的区间，所以本次选 C。这是 CDF 采样的直观解释，实际 vLLM/Ascend 路径不一定按区间扫描。

### 9.2 logits 先变成实际采样分布

采样前可能应用惩罚、词表限制、temperature、top-k/top-p 等操作，再形成实际分布。具体顺序由 sampler 实现决定。拒绝采样中使用的 p、q 应与各自**实际 proposal/采样机制**一致，不能混用处理前后的概率。

温度为 0 的贪心路径通常直接选择 logits 最大项；随机路径才需要随机数。

### 9.3 Ascend 的指数随机数方法

本次阅读的 `vllm_ascend/sample/sampler.py::random_sample()` 核心是：

```python
noise = torch.empty_like(probs)
noise.exponential_()           # 每个候选一个 Exp(1) 随机数
token_ids = (probs / noise).argmax(dim=-1)
```

实际实现把这个噪声张量命名为 `q`，并可为各请求使用独立 generator。**此处 q 是指数随机噪声，不是 draft distribution q_i。**

例如：

```text
概率 p：      [0.10, 0.60, 0.30]
指数噪声 E：  [0.30, 1.20, 0.10]
p / E：       [0.33, 0.50, 3.00]
本次 argmax： C
```

虽然最终调用 argmax，参与比较的是加了随机性的分数。其原理是：`E_v / p_v` 服从速率为 p_v 的指数分布，各候选竞争最小到达时间，v 胜出的概率为 `p_v / sum(p)`。选择最小 `E/p` 等价于选择最大 `p/E`。

它也可以写成 Gumbel-max 的形式：`argmax(log p - log E)`。同一行概率乘一个正的常数不会改变 argmax，所以 recovered sampling 可以直接使用未归一化的非负权重。

源码还使用辅助 NPU stream 生成噪声、等待 stream 并记录张量生命周期；这些操作是为了保证异步执行时数据就绪。原实现中的 `probs.div_(q)` 会原地修改概率张量，阅读调试值时要注意。其避免 `multinomial` 的原因是该实现针对当时 NPU 同步开销的取舍，不能概括成所有 PyTorch 后端的 multinomial 都会同步。

<a id="rejection"></a>

## 10. 接受概率、recovered 分布与 bonus

本节讨论标准拒绝采样；entropy verify、synthetic verify 或其他改变接受规则的模式，不自动具有相同的分布保证。

### 10.1 如何验证一个随机草稿 token

对第 i 个草稿 d_i：

```text
接受概率 α_i = min(1, p_i(d_i) / q_i(d_i))
生成 U_i ~ Uniform(0,1)
若 U_i < α_i，则接受
```

例如 draft 给某 token 0.5，target 给它 0.1，接受概率是 0.2；U=0.12 时接受，U=0.76 时拒绝。如果 p/q 大于 1，接受概率按 1 计算。

实现中直接比较 `p/q >= U` 也可以达到相同效果，因为 U 不超过 1；还需要处理零概率和 placeholder 等边界。

### 10.2 为什么拒绝后从 max(p-q,0) 抽样

接受分支已经贡献了两种分布重叠的概率质量。拒绝分支补回 target 尚缺的那部分，权重为：

```text
w_i(v) = max(p_i(v) - q_i(v), 0)
r_i(v) = w_i(v) / sum_u w_i(u)
```

举例：

| token | target p | draft q | max(p-q,0) | recovered r |
| --- | ---: | ---: | ---: | ---: |
| A | 0.10 | 0.50 | 0.00 | 0.00 |
| B | 0.60 | 0.30 | 0.30 | 0.75 |
| C | 0.30 | 0.20 | 0.10 | 0.25 |

若此位置拒绝，需要从 `[0,0.75,0.25]` 抽一个替换 token。CDF 随机数 0.80 会选 C；不是比较后永远选 B。

从单步概率质量看，接受分支对 v 贡献 `min(p(v),q(v))`，拒绝后补回 `max(p(v)-q(v),0)`，两者之和恰好是 p(v)。这解释了标准算法为什么能够保留 target 的采样分布。算法背景见 [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192)。

若 p=q，残差质量为 0，但标准算法也不会到达一个需要残差抽样的拒绝事件；实际实现仍要考虑数值误差或预先计算的未使用候选。

### 10.3 draft_probs 什么时候有值

本次分支的概率导出开关要求：

```python
rejection_sample_method == "standard"
draft_sample_method == "probabilistic"
```

还需要没有被 all-greedy 等路径提前返回。默认 `draft_sample_method` 在本次源码中是 `greedy`。

| 场景 | 本文所读路径中的 draft_probs |
| --- | --- |
| 确定性 argmax draft | 通常为 None。 |
| N-gram 等确定性 proposal | 通常为 None。 |
| 开启标准 probabilistic draft，实际走随机采样 | 返回与 proposal 对齐的概率。 |
| 请求全部 greedy | 可能直接 argmax 并返回 None。 |
| dummy/profile 调用没有 sampling metadata | 本地 fallback 可以返回 None。 |
| 不兼容的词表映射或本地 argmax 优化 | 可能配置拒绝或回退，需看实际分支。 |

同一个服务里，target 使用随机采样，drafter 仍可以使用确定性 argmax，两者不矛盾。

当 proposal 是确定性的 d，等价分布是 `q(d)=1`，其他 token 的 q 为 0。标准拒绝采样可据此简化：接受概率为 `p(d)`；拒绝时把 target 中 d 的权重清零，再从其他候选抽样。源码参数 `IS_NGRAM`/`NO_DRAFT_PROBS` 在这里也用于这种 None 情况，名称不代表调用者一定是 N-gram。

**不能把随机 draft 的真实 q 随意丢弃后仍声称标准算法等价。** None 路径成立的前提是 proposal 机制与该简化语义一致。

<a id="recovered-pytorch"></a>

## 11. sample_recovered_tokens 的 PyTorch 路径

这个函数准备的是“如果某位置第一次拒绝，应使用哪个替换 token”。它不运行 target 或 MTP forward，也不负责决定哪个位置被拒绝。

### 11.1 外层函数负责什么

| 输入/操作 | 含义 |
| --- | --- |
| `num_draft_tokens` | 每请求草稿数，比如 `[3,2]`。 |
| `cu_num_draft_tokens` | 展平草稿边界，比如 `[3,5]`。 |
| `draft_token_ids` | `[N]`，N 是整个 batch 的草稿总数。 |
| `draft_probs` / `target_probs` | 常规路径为 `[N,V]`，前者允许 None。 |
| `q.exponential_()` | 生成 `[B,V]` 的指数噪声；q 在此仍不是草稿概率。 |
| per-request generator | 为设置 seed 的请求使用相应随机数生成器。 |
| `recovered_token_ids` | `[N]`，为每个草稿位置预先准备一个候选替换 ID。 |
| 后端分支 | Triton、普通 PyTorch 或 blockwise PyTorch。 |

### 11.2 从展平 token 找到请求编号

普通 PyTorch 路径先计算：

```python
cu_start = [0, 3]
cu_end = [3, 5]
token_indices = [0, 1, 2, 3, 4]
```

然后用广播构造 `[N,B]` 的区间归属表：

```python
in_range = (
    (token_indices[:, None] >= cu_start[None, :])
    & (token_indices[:, None] < cu_end[None, :])
)
token_to_batch = in_range.int().argmax(dim=1)
# [0, 0, 0, 1, 1]
```

每个位置由此找到自己的请求噪声向量。

### 11.3 构造残差权重，再进行随机抽样

教学代码，保留核心张量路径：

```python
if draft_probs is None:
    # 这里假设是确定性 proposal
    weights = target_probs.clone()
    weights[torch.arange(N), draft_token_ids] = 0
else:
    weights = (target_probs - draft_probs).clamp_min(0)

noise_per_token = noise_per_request[token_to_batch]  # [N,V]
recovered_ids = (weights / noise_per_token).argmax(dim=1)
```

生产实现还屏蔽 placeholder，处理噪声的异常数值；若启用 reduce sampling，则需要在压缩词表中对齐 draft 概率，并将选出的局部索引通过 `target_indices` 映射回全局 token ID。

没有显式 `weights /= weights.sum()` 也能抽样，因为同一行的归一化常数不改变最终 argmax。同一请求的不同位置可能复用该请求的一组指数噪声，所以预备候选之间可能相关；标准链式验证最终只会消费**第一个拒绝位置**的 recovered token。

这也解释了为什么有些后缀已经注定不会输出，代码却仍为它们计算了 recovered 候选：GPU/NPU 上批量计算往往比逐 token 等待和分支更合适。

<a id="rejection-pytorch"></a>

## 12. rejection_random_sample_pytorch 如何并行模拟短路

这一函数拿到已经计算好的 p、q、均匀随机数和 recovered 候选后，组装最终输出。重点是“逻辑顺序”和“设备执行顺序”不同。

### 12.1 展平数据变为二维请求网格

```python
# K=3，两个请求各有 3、2 个草稿
pos = torch.arange(3)[None, :]            # [1,3]
starts = torch.tensor([0, 3])
lengths = torch.tensor([3, 2])
valid_mask = pos < lengths[:, None]      # [B,K]
global_indices = starts[:, None] + pos   # [B,K]
```

最后一个无效位置的 gather 索引会做安全 clamp，再由 valid_mask 排除。随后取出每格的 draft ID、`p_i(d_i)`、`q_i(d_i)`、U 和 recovered ID。

### 12.2 所有位置同时计算接受条件

```python
accept = (draft_p > 0) & (target_p / draft_p >= uniform)
reject = (~accept) & valid_mask
```

None proposal 路径的 `draft_p` 在候选 d 上为 1。混合 batch 还需要用 `is_greedy` 将随机处理限制在对应请求，greedy 请求由其他分支处理。

### 12.3 找第一个拒绝位置

```python
has_reject = reject.any(dim=1, keepdim=True)
first_reject = torch.where(
    has_reject,
    reject.int().argmax(dim=1, keepdim=True),
    K,
)
```

argmax 对 `[0,1,1]` 返回第一个 1 的下标 1。不能对全 False 行直接相信 argmax 的 0，必须用 `any()` 或其他哨兵机制区分“没有拒绝”。

### 12.4 屏蔽拒绝后缀并输出

```text
原始逐位置接受结果： [True, False, True]
有效接受前缀：       [True, False, False]
最终输出：           [d1, r2, -1, -1]
```

后面的 True 是在包含已拒绝 token 的草稿条件下得到的，不能恢复成有效接受。

源码通过 `pos >= first_reject` 形成 should_skip，再用 first_reject_mask 单独写入 recovered。全部接受时，把 bonus 写在该请求的草稿长度位置：

```text
3 个草稿全接受： [d1, d2, d3, bonus]
2 个草稿全接受： [d1, d2, bonus, -1]
```

回答“有一个被拒绝，后面是不是不用采样了”时，需要区分：

| 环节 | 顺序/并行关系 |
| --- | --- |
| 自回归 MTP draft | d2 依赖 d1，d3 依赖 d2。 |
| target 验证 forward | 打包多行，通过 causal mask 一次计算。 |
| PyTorch 接受判定与候选抽样 | 可以对所有位置并行计算。 |
| 有效输出 | 必须在第一次拒绝处终止草稿前缀。 |

这份标准算法之外的 entropy/synthetic/block verify 分支需要另看规则，不能把“允许更宽松接受”的分支与标准 p/q 拒绝采样混为一谈。

<a id="handoff"></a>

## 13. draft token 如何传入下一轮，batch 换行为什么不会串请求

### 13.1 草稿传递有设备侧和调度侧两条线

```text
MTP 返回 [B,K]
    ├─ runner._draft_token_ids：设备上保留真实 token IDs
    └─ take/copy draft IDs 等接口：提供 scheduler 所需信息

下一轮 scheduler_output
    → 确定各请求调度多少草稿
    → 组装 input_ids 中各请求的 [x, d1, d2, d3]
    → 计算 SpecDecodeMetadata
    → target forward + rejection sampler
```

同步/异步调度、PP/CP 路径可能使用 CPU 列表、设备缓存或占位数据，不能假设所有真实 token 都必须“先回 CPU，再传回 NPU”。概率若需要跨轮保留，也必须与 request ID 和位置对齐。

### 13.2 同一请求可以从上一轮第 2 行移到这一轮第 0 行

假设上一轮顺序是 `[A,B,C]`，当前顺序是 `[C,A]`，每请求缓存了 3 个草稿：

```python
previous_drafts = [
    [A1, A2, A3],
    [B1, B2, B3],
    [C1, C2, C3],
]
prev_req_id_to_index = {"A": 0, "B": 1, "C": 2}
current_req_ids = ["C", "A"]
```

请求 C 当前在第 0 行，通过稳定的 `req_id="C"` 找到旧行号 2。旧草稿展平后，起始地址是：

```python
source_start = previous_row * previous_K
# C: 2 * 3 = 6 → 源下标 [6,7,8]
```

当前若每个请求安排 `[x,d1,d2,d3]`，C 的目标草稿下标是 `[1,2,3]`，A 的是 `[5,6,7]`。因此：

```text
旧草稿的源下标：       [6,7,8, 0,1,2]
当前 input_ids 目标：  [1,2,3, 5,6,7]
```

设备端 gather + scatter 完成重排。源码中的 `prev_draft_token_indices` 管“从旧 buffer 哪取”，`spec_flattened_indices` 管“写入当前打包输入哪”；前面非草稿 x 的位置还有单独的采样 token 索引。

新请求没有上一轮映射，使用自身的输入准备路径。请求退出和本轮草稿长度缩短也需要跳过或裁剪。**行号会变，request ID 才是跨轮匹配依据。**

<a id="bookkeeping"></a>

## 14. _bookkeeping_sync 有什么用

它负责把采样结果纳入 runner 的请求状态和输出结构。名字里的 sync 不能理解为“只做一次设备同步”，也不能把它当成验证算法。

本次代码中主要做这些事：

| 工作 | 目的 |
| --- | --- |
| 处理 discard 请求及对应 generator offset | 避免本不该输出的采样污染状态；具体 offset 常数属于该实现。 |
| 复制 req_ids、req_id_to_index | 异步调度会修改 batch，返回结果仍需保持本轮映射。 |
| 同步路径解析 token 列表 | 去掉 `-1`/placeholder，整理 logprobs。 |
| 异步路径记录设备结果、无效请求及上一轮映射 | 尽量延迟 D2H，供下一轮直接接续。 |
| 更新 token_ids_cpu、num_tokens_no_spec 等 | 推进 runner 侧请求的有效输出状态。 |
| 更新 request.output_token_ids | 保存每个请求已得到的输出。 |
| 整理 prompt logprobs | 返回上层需要的结果。 |

注意 generator 的 `offset - 4` 只是这份代码的实现细节，不代表数学上每次采样都固定消耗 4 个可见随机数。同步解析可能触发等待；在热路径里逐个设备 `.item()` 也可能引入 CPU-NPU 同步。

<a id="architecture"></a>

## 15. Qwen3.5 MTP 为什么只有 full attention

主干按照配置中的 `layer_types` 构造层。Qwen3.5-35B-A3B 的官方配置为 40 个主干层，结构重复 `linear → linear → linear → full`，即 30 个 GDN linear attention、10 个 full attention；`mtp_num_hidden_layers=1`。[官方配置](https://huggingface.co/Qwen/Qwen3.5-35B-A3B/raw/main/config.json)

MTP 构造时明确指定：

```python
Qwen3_5DecoderLayer(
    vllm_config,
    layer_type="full_attention",
    prefix=f"{prefix}.layers.{idx}",
)
```

decoder 根据类型选择 `QwenGatedDeltaNetAttention` 或 `Qwen3NextAttention`。MTP 这条分支创建后者，没有创建 GDN 模块。

它是额外的预测模块，接收主模型已处理过的 hidden states，并不复制完整的 40 层混合网络。**K=3 表示调用预测过程生成三个草稿，不是把 `mtp_num_hidden_layers` 变成 3。**

这是当前模型结构与权重接口的事实。不能据此推断“vLLM 从原理上不允许 linear-attention drafter”，也不能没有资料就替模型作者确定设计动机。

<a id="kv-allocation"></a>

## 16. MTP full attention 的 KV cache 怎么分配、在哪写入

### 16.1 启动分配与运行时分块是两回事

```text
Attention 构造，按层名注册到 static_forward_context
    ↓
get_kv_cache_spec() 收集层的缓存规格
    ↓
引擎查询可用显存，生成 KVCacheConfig
    ↓
_allocate_kv_cache_tensors() 申请底层 buffer
    ↓
_reshape_kv_cache_tensors() 建立 dtype / shape / K-V 视图
    ↓
bind_kv_cache() 按层名绑定
```

MTP 层名类似 `mtp.layers.0.self_attn.attn`，可能有外层前缀。初始 `self.kv_cache` 可以只是占位张量，完整缓存稍后统一绑定。

| 层类型 | 缓存描述 |
| --- | --- |
| 主模型 full attention | `FullAttentionSpec`，按位置存 K/V。 |
| 主模型 GDN linear attention | `MambaSpec` 等，存卷积和递归/SSM 状态。 |
| MTP full attention | 自己的 `FullAttentionSpec`，按位置存 K/V。 |

linear attention 也有状态缓存，只是不能简单按标准逐 token K/V 理解。原始内存以 `int8` 字节 buffer 申请，也不等于模型使用 INT8 KV 量化；后面可能 view 成 BF16/FP16。

运行时 `kv_cache_manager.allocate_slots()` 从池中给请求分配 block，投机可能额外预留 lookahead token 所需空间。内存池不是每个 draft step 重新申请一次。

### 16.2 形状和一个估算

普通 full attention 的逻辑形状可以写成：

```text
K: [num_blocks, kernel_block_size, local_kv_heads, head_dim]
V: [num_blocks, kernel_block_size, local_kv_heads, head_dim]
```

管理器的 block_size 与 backend 的 kernel_block_size 在某些 Ascend 混合路径下可能不同，要确认所看的是哪一层规格。

该模型配置是 2 个 KV heads、head_dim=256。如果 drafter 也使用 TP=2、KV 为 BF16，则每卡 local KV heads 为 1，每位置这一层的有效 K/V 数据量为：

```text
2 (K,V) × 1 (head) × 256 × 2 bytes = 1 KiB
4096 个位置约为每卡 4 MiB
```

这是有效数据量估算，不是整个预分配缓存池大小；没有计入 padding、块取整、lookahead、空闲块和其他层。实际容量还受所有缓存组的共同规划影响。

### 16.3 真正写 K/V 的调用链

本次阅读的 Qwen3.5 普通 full attention 路径：

```text
MTP decoder
    → Qwen full attention 的 Q/K/V 投影与 RoPE
    → self.attn(q, k, v)
    → Ascend attention backend
    → reshape_and_cache()
    → DeviceOperator.reshape_and_cache()
    → torch_npu.npu_scatter_pa_kv_cache(...)
```

`slot_mapping` 指定每个输入行写入哪个物理槽位。写回的是**该次 forward 的输入对应的 K/V**，不是采样器刚选出但尚未再次送入模型的输出 token 的 K/V。

因此生成 d3 并不代表已经把 d3 作为输入计算过 MTP K/V。普通 K=3 自回归链中，第三次调用消费 d2、输出 d3。

<a id="kv-groups"></a>

## 17. KVCacheGroup、AttentionGroup、shared_by 分别是什么

### 17.1 同一个 KVCacheGroup 不等于同一份 K/V 数值

同一组的层需要兼容的缓存管理规则，共用该组的请求 block table。每层仍然存储自己投影出来的 K/V。若两个层用同一个 block ID=9，它们访问的是各自层区域里的第 9 块。

在前面模型和兼容缓存规格的常规路径下，主模型 10 个 full attention 与 MTP 1 个 full attention 归入一个组，30 个 linear 层再分到其他组。这个结果不能推广为“所有 MTP 都固定和主模型 full 同组”：规格、模型、布局、版本和分组策略变化都可能改变结果，gid 也不能硬编码。

Drafter 会根据自己的层名找到 `kv_cache_gid`，再构建草稿用的 attention group/metadata。即使 gid 相同，draft 与 target 执行时的 query 长度、seq_lens 和 slot_mapping 仍可不同。

### 17.2 旧 shared_by 接口如何理解

旧版通用混合缓存规划可从不同 group 的同一组内下标各取一层，组成一个 `shared_by`，复用一个底层 buffer。各组拥有不同的 block 分配；共享 buffer 不等于两个活跃组可以同时拥有并覆盖同一个物理 block。[vLLM v0.17.0 对应实现](https://github.com/vllm-project/vllm/blob/v0.17.0/vllm/v1/core/kv_cache_utils.py#L1030-L1055)

对本例推导，F/L 分别表示主模型 full/linear 层：

| 组内下标 | Full 组 | Linear A | Linear B | Linear C |
| ---: | --- | --- | --- | --- |
| 0 | F3 | L0 | L1 | L2 |
| 1 | F7 | L4 | L5 | L6 |
| … | … | … | … | … |
| 9 | F39 | L36 | L37 | L38 |
| 10 | MTP | 空 | 空 | 空 |

此布局中，每一行对应一个 buffer 的共享名单。最后一行只有 MTP，因此：

```python
shared_by = ["mtp.layers.0.self_attn.attn"]
```

它与其他 full 层同 KVCacheGroup，同时它自己的 buffer 又只由它使用，这两点可以同时成立。这个单元素列表是特定分组与顺序下的推导，应以运行时对象核实。

### 17.3 混合内存布局不代表 MTP 含有 linear attention

Ascend reshape 路径为了兼容 attention/Mamba 混合 buffer，可能让只有 MTP full attention 的 buffer 也采用相同布局并保留 padding。进入 `hybrid_with_attn_and_mamba` 相关分支，只说明使用了混合缓存布局，不表示 MTP 层内部创建了 GDN。

整理时发现，相邻 vLLM 源码的新 `KVCacheTensor` 用 `layers`、`layer_stride`、`block_stride`、`offset` 描述布局，而 Ascend runner 某些代码仍读取 `shared_by`。这两套字段不能直接互换。复习旧 shared_by 结论时，应先确认调试进程安装的 vLLM 版本；不能用当前工作区的一套接口猜另一套接口的运行结果。

<a id="kv-lifecycle"></a>

## 18. 验证后为什么还要 MTP 的历史 KV

### 18.1 验证采样不等于清除整个 MTP 缓存

| 操作 | 含义 | 典型发生时机 |
| --- | --- | --- |
| 逻辑失效 | 旧数值可能还在，但不能作为有效历史读取。 | 根据接受结果调整长度、索引和可见范围。 |
| 覆盖 | 新 K/V 写入相同槽位。 | 下一轮 MTP 重算相应位置。 |
| 回收 block | 请求放弃块的使用权，缓存池可重新分配。 | 完成、取消、抢占；可能等待在途写入结束。 |
| 清零 | 显式把内存写为零。 | 某些初始化、Mamba 或混合精度块复用路径，取决于版本。 |

主模型验证使用自己的 attention 层，rejection sampler 根据概率选 token；它们不会因为这一轮验证结束就清空整份 MTP K/V。采样后马上发起下一轮 proposer 时，MTP cache 会更新，容易被误看成“验证把 MTP 缓存改了”。

不能把“通常不靠清零实现拒绝回退”扩大成“任何版本任何缓存块都永远不清零”。新版本可能为了状态初始化、混合精度防 NaN 等，对重新分配的块执行清零。

### 18.2 一个精确到位置的覆盖例子

设此前已经采样出 x，MTP 第一遍的采样行位置为 p：

| MTP 调用 | 消费的 embedding | 写入位置 | 输出 |
| --- | --- | --- | --- |
| step 0 | x | p | d1 |
| step 1 | d1 | p+1 | d2 |
| step 2 | d2 | p+2 | d3 |

验证接受 d1、拒绝 d2，重新采样 r，下一轮 MTP 的有效第一遍输入使用 target 对 `[x,d1]` 的真实 hidden states，与 embedding `[d1,r]` 融合。按本例对齐，它会重算并覆盖 p+1、p+2；p 及更早的有效缓存仍供 attention 读取。

即使 d1 被接受，它相关的 MTP 位置也可能被重算，因为此时拿到了 target 的真实 hidden state，替换此前递归 draft 使用的表示。不能概括成“接受槽位永远保持原值，只覆盖拒绝槽位”。

### 18.3 已经有 target hidden，为什么仍然需要历史 K/V

MTP 对融合输入 z 还要执行自己的 full attention。省略 RoPE 和其他细节：

```text
Q_new = z_new Wq
K_new = z_new Wk
V_new = z_new Wv

output = softmax(Q_new × [K_history; K_new]^T / sqrt(d))
         × [V_history; V_new]
```

比如已缓存位置 0…99，新一轮只计算位置 100、101：位置 100 的 query 需要历史 0…99 和自己的 K/V；位置 101 还可以读取位置 100。清掉历史后，计算就只剩最近一两行，与原模型不同。若要保持结果，就得重算历史 K/V，这正是缓存要避免的开销。

target hidden 提供已经融合过的上下文表示，MTP 历史 K/V 提供 MTP 这一层 attention 具体要访问的内容。就像普通 Transformer 第 20 层仍有自己的 attention，不能因为第 19 层 hidden 已包含历史，就省掉第 20 层的历史 K/V。

### 18.4 padded 与非 padded 路径要分别观察

非 padded 的 `prepare_inputs()` 会计算拒绝后缀长度，裁剪有效输入并调整 seq_lens、query_start_loc 和 slot_mapping。K=3、输出 `[d1,r]` 时：

```text
拒绝后缀长度 = 3 + 1 - len([d1,r]) = 2
```

padded 的 `prepare_inputs_padded()` 暂时保留后缀行，通过 `query_end - 1 - num_rejected` 选出有效采样行；它本身不一定立即缩短 seq_lens。第一遍的因果可见范围与后续单 token draft 的长度处理都要正确。

会话中未通过运行时断点证明本地所有 padded 后续分支的长度修正都正确，因此不能把它写成已验证无误的事实。若拒绝后仍读到旧分支，应检查 metadata 的有效长度和写入位置，而不是直接给 KV cache 加清零。

### 18.5 请求结束与新请求复用

释放通常经过 scheduler → KVCacheManager → coordinator → BlockPool，减少引用计数并把可复用块放回队列。异步执行时，还要防止在途 kernel 正在写的块过早复用。

新请求可以覆盖**已经释放并重新分配给它的块**，不能覆盖另一个活跃请求仍拥有的有效缓存。启用前缀缓存时还可能保留可命中的内容；本次启动示例关闭了 prefix caching，但当前请求自己的 decode cache 仍然需要。

<a id="debug-checklist"></a>

## 19. 下次调试时，从这些断点开始

路径以对应仓库根目录为基准；下表强调函数名，避免版本更新后照搬行号。

| 目标 | 文件与函数 | 重点观察 |
| --- | --- | --- |
| 确认服务真的启用投机 | Ascend `worker/model_runner_v1.py` | speculative_config、drafter、执行进程。 |
| 确认草稿产生 | 同文件 `propose_draft_token_ids()` | K、`_draft_token_ids`、请求顺序。 |
| 确认衔接 token | Ascend `spec_decode/llm_base_proposer.py::prepare_next_token_ids_padded()` | sampled IDs、valid count、next token。 |
| 确认首次错位 | 同文件 `set_inputs_first_pass()` | target IDs、shifted IDs、采样行。 |
| 看三个递归步骤 | 同文件 `_run_merged_draft()` | 每步 input、position、hidden、metadata。 |
| 确认验证输入 | vLLM `v1/worker/gpu_model_runner.py::_prepare_input_ids()` 及 Ascend 对应路径 | 源/目标索引、prev row、打包后的 `[x,d1,d2,d3]`。 |
| 确认 logits 对齐 | Ascend `_calc_spec_decode_metadata()` | logits、target、bonus 三类索引。 |
| 找接受/拒绝入口 | Ascend `_sample()` 与 `sample/rejection_sampler.py` | draft_probs 是否 None，实际走哪个 verify 分支。 |
| 看 recovered | `sample_recovered_tokens_pytorch()` | weights、token_to_batch、noise、候选 IDs。 |
| 看最终输出屏蔽 | `rejection_random_sample_pytorch()` | first_reject、valid_mask、输出 `-1`。 |
| 看请求状态推进 | `_bookkeeping_sync()` | 有效输出和请求 ID 映射。 |
| 看 Qwen MTP 融合 | Ascend `patch/worker/patch_qwen3_5.py` | cat 前后形状、真实绑定的 forward。 |
| 看缓存规划 | `get_kv_cache_spec()`、`initialize_kv_cache()` | 每层 spec、groups、tensor 布局版本。 |
| 看物理写回 | Ascend `attention/attention_v1.py::reshape_and_cache()` | layer name、slot_mapping、K/V 视图。 |
| 看拒绝回退与释放 | vLLM scheduler、KVCacheManager、BlockPool | computed tokens、block ownership、延迟释放。 |

Debug Console 的只读表达式示例，变量必须在当前栈帧中存在：

```python
# 列出缓存组
[(i, g.layer_names) for i, g in enumerate(kv_cache_config.kv_cache_groups)]

# 仅适用于旧 shared_by 接口
[
    t.shared_by
    for t in kv_cache_config.kv_cache_tensors
    if any("mtp" in name for name in t.shared_by)
]

# 新布局接口应观察每层的 offset / stride
[
    (t.layers, t.offset, t.layer_stride, t.block_stride)
    for t in kv_cache_config.kv_cache_tensors
    if any("mtp" in name for name in t.layers)
]
```

Debug 时取 `.cpu()`、`.tolist()` 或 `.item()` 可以帮助观察，但可能同步设备、扰动性能。不要把临时观察代码留在正式热路径里，也不要用被断点打断的时间来衡量吞吐性能。

<a id="questions"></a>

## 20. 本次会话的问题索引

以下保留全部技术提问的主题；相同问题合并到同一节。

| 会话中的问题 | 对应章节 |
| --- | --- |
| VS Code 启动脚本如何新增 MTP 参数？ | [2. 启动与排查](#launch) |
| debug 没走投机、MTP decode metadata 为空？ | [2. 启动与排查](#launch) |
| hidden_states 与 sample_hidden_states 的区别？ | [5. Hidden states](#hidden-states) |
| _bookkeeping_sync 有什么用？ | [14. Bookkeeping](#bookkeeping) |
| bonus 是什么、如何验证、如何采样？ | [3. 时序](#timeline)、[10. 拒绝采样](#rejection) |
| K=3 是否能输出 5 个、一次验证最多几个？ | [3. 输出数量](#timeline) |
| prepare_next_token_ids_padded 做什么？ | [8. 逐句说明](#prepare-next) |
| _run_merged_draft 做什么？ | [7. Draft 循环](#merged-draft) |
| random sample 详细怎么做？ | [9. 指数采样](#sampling) |
| Qwen3.5 MTP 为什么 cat？ | [5. 特征融合](#hidden-states) |
| 从 max(p-q,0) 分布抽一个是什么意思？ | [10. 残差分布数值例子](#rejection) |
| 为什么区分 num_speculative_tokens == 1？ | [7. 提前返回](#merged-draft) |
| K=3 是否逐个采样，拒绝后是否停止后续计算？ | [12. 并行与逻辑短路](#rejection-pytorch) |
| 为什么后续 token 放到 input_ids 第一个位置？ | [6. 输入布局](#draft-inputs) |
| 为什么第一次放最后，后续放最前？ | [6. 输入错位与紧凑布局](#draft-inputs) |
| positions 怎么作用到 input？ | [6. RoPE 与 slot](#draft-inputs) |
| draft 结束后验证在哪里调用？ | [3. 时序](#timeline)、[19. 断点](#debug-checklist) |
| draft IDs 怎么传给下一轮验证？ | [13. 跨轮传递](#handoff) |
| sample_recovered_tokens 做什么、PyTorch 逐句怎么走？ | [11. Recovered 路径](#recovered-pytorch) |
| draft_probs 什么时候有值/没值？ | [10. Proposal 分布](#rejection) |
| rejection_random_sample_pytorch 做了什么？ | [12. 向量化验证](#rejection-pytorch) |
| 验证 d1、d2、d3 具体取哪行概率？ | [4. Logits 对齐](#logits-alignment) |
| Qwen MTP attention 在哪里更新 KV cache？ | [16. 写回调用链](#kv-allocation) |
| 请求从旧 batch 第 2 行移到第 0 行如何找回草稿？ | [13. 请求重排例子](#handoff) |
| MTP full cache 怎么分配，为什么没有 linear？ | [15. 结构](#architecture)、[16. 分配](#kv-allocation) |
| MTP 更新的缓存什么时候消除，验证会消除吗？ | [18. 生命周期](#kv-lifecycle) |
| MTP full 与普通 full 是否同 group？ | [17. 分组](#kv-groups) |
| 验证后 MTP cache 是否再也用不到，只等覆盖？ | [18. 有效历史](#kv-lifecycle) |
| MTP full 的 shared_by 是哪个？ | [17. 共享名单推导](#kv-groups) |
| 验证后的 MTP 为什么还需要历史 KV？ | [18. Attention 公式与位置例子](#kv-lifecycle) |

<a id="review"></a>

## 21. 复习自测：先回答，再展开

建议第一遍顺着数据流通读，以后每次只挑 3～5 题口述，并在调试器里核对一个张量。下一次复习优先看答错的题，不必每次从头阅读。

<details>
<summary>1. K=3，只有 d1 被接受，这轮输出什么？拒绝后缀有多长？</summary>

输出 `[d1,r]`，通常张量表现为 `[d1,r,-1,-1]`。有效输出数为 2，接受的 draft 数为 1，拒绝后缀长度为 2。这三个计数不同。

</details>

<details>
<summary>2. 输入 [x,d1,d2,d3]，验证 d2 用哪一行 logits？</summary>

输入 d1 对应的那一行。该行预测下一 token d2。输入 d2 对应的 logits 用于 d3。

</details>

<details>
<summary>3. 为什么 argmax(probs / Exp(1)) 不是贪心？</summary>

每次除以不同的随机噪声，胜出 token 的分布与 probs 成比例。直接 argmax(probs) 才是固定选最大概率项。

</details>

<details>
<summary>4. draft_probs=None，target temperature 又大于 0，可以正常投机吗？</summary>

可以。若 drafter 使用确定性 proposal，就按 one-hot q 做标准随机验证。target 随机与 draft 确定性可以共存。不能把一个原本随机 proposal 的 q 随意丢掉。

</details>

<details>
<summary>5. 接受条件为 [True,False,True]，最后一个 True 能输出吗？</summary>

不能。只保留第一次拒绝前的连续前缀；在拒绝位置输出 recovered，后缀全部作废。设备已经算过后缀，不等于后缀有效。

</details>

<details>
<summary>6. 为什么后续 draft 放 input_ids[0]，positions 却可能是 100？</summary>

0 是复用 buffer 的行号，100 是序列的逻辑位置。每请求仅一个有效输入时放到前 B 行，历史位置由 positions 和 metadata 表达。

</details>

<details>
<summary>7. MTP 与主干 full 同 KVCacheGroup，shared_by 能只有 MTP 自己吗？</summary>

能。旧版 shared_by 从不同组的相同下标取层；11 个 full 中的最后一个是 MTP，其他三个 linear 组各只有 10 层，最后一行就只有 MTP。

</details>

<details>
<summary>8. 验证后把 MTP 所有 KV 清空，为什么会改变下一次 draft？</summary>

下一轮通常只算少量新/重算位置，其 query 仍需访问前缀 K/V。清空后丢失 MTP 自己的历史 attention 内容；target hidden 不能替代这一层的全部历史 K/V。

</details>

<details>
<summary>9. 接受的 draft 对应 MTP 缓存一定不再改写吗？</summary>

不一定。下一轮拿到 target 真实 hidden states 后，会重算覆盖相应位置，即使相关 token 已被接受。更早的有效历史通常继续保留。

</details>

<details>
<summary>10. batch 重排时，防止草稿串请求的关键是什么？</summary>

用 request ID 查上一轮行号，再构造旧 draft buffer 的源索引和当前 input_ids 的目标索引。不能把当前行号直接当旧行号。

</details>

<details>
<summary>11. 看到 spec_decode_metadata=None，第一件事该确认什么？</summary>

确认是不是首次 prefill 或本轮根本没调度草稿，再检查 speculative_config、drafter、保存的 draft IDs 和 scheduler_output。单次 None 不是未开启投机的充分证据。

</details>

<details>
<summary>12. slot_mapping.fill_(-1) 清除的是 KV 数据吗？</summary>

它改的是写入地址索引，将相应输入标成不写入有效槽位。旧 KV 字节可能仍留在内存；正确性依赖有效长度、可见范围和块所有权，而不是必须全部归零。

</details>

## 22. 参考入口与阅读范围

- [vLLM Ascend：Speculative Decoding](https://docs.vllm.ai/projects/ascend/en/latest/user_guide/feature_guide/speculative_decoding.html)：启动方式和支持模式；latest 页面会随版本变化。
- [Qwen3.5-35B-A3B 官方配置](https://huggingface.co/Qwen/Qwen3.5-35B-A3B/raw/main/config.json)：主干层类型、MTP 层数和 attention 维度。
- [Speculative Decoding 论文](https://arxiv.org/abs/2211.17192)：标准接受概率和残差采样的算法背景。
- [vLLM v0.17.0 缓存规划源码](https://github.com/vllm-project/vllm/blob/v0.17.0/vllm/v1/core/kv_cache_utils.py)：用于解释旧 shared_by 分组，不能替代本地安装版本。
- [vLLM 仓库](https://github.com/vllm-project/vllm)、[vLLM Ascend 仓库](https://github.com/vllm-project/vllm-ascend)：按第 19 节函数名定位；本地 patch 的行为可能尚未出现在主分支。

本文的具体函数分析同时依据会话中粘贴的代码和本地源码阅读。数值例子是教学推导；没有把本次阅读写成已经在 NPU 上完成的性能或精度验证报告。
