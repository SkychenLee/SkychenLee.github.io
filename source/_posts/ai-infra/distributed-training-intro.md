---
title: 分布式训练入门：从 DataParallel 到 DeepSpeed
tags:
  - AI Infra
  - 分布式训练
  - PyTorch
  - DeepSpeed
categories:
  - AI Infra
cover: https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1280
description: 本文介绍大模型训练中常用的分布式训练方案，从 PyTorch 原生的 DataParallel/DistributedDataParallel 到 DeepSpeed ZeRO 优化。
katex: false
mermaid: false
mathjax: false
abbrlink: 40991
date: 2025-05-24 10:00:00
top_img:
---

# 分布式训练入门：从 DataParallel 到 DeepSpeed

在大模型时代，单卡训练往往无法满足需求。本文介绍主流的分布式训练方案。

## 1. PyTorch DataParallel (DP)

最简单的多卡方案，但存在 GIL 限制和负载不均衡问题。

```python
import torch.nn as nn

model = nn.Module(...)
model = nn.DataParallel(model)
model = model.cuda()
```

**缺点**：
- 单进程多线程，受 GIL 限制
- GPU 0 负载过重（梯度汇总）
- 扩展性差

## 2. DistributedDataParallel (DDP)

PyTorch 官方推荐方案，多进程架构，性能更优。

```python
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

dist.init_process_group(backend='nccl')
model = DDP(model, device_ids=[local_rank])
```

**优势**：
- 多进程，无 GIL 限制
- 负载均衡，每张卡处理相同 batch
- 通信效率高（Ring-AllReduce）

## 3. DeepSpeed ZeRO

微软开源方案，通过优化器状态分片大幅降低显存占用。

### ZeRO 三阶段

| 阶段 | 分片内容 | 显存节省 |
|------|----------|----------|
| ZeRO-1 | 优化器状态 | 4x |
| ZeRO-2 | + 梯度 | 8x |
| ZeRO-3 | + 模型参数 | $N$ 倍（$N$ 为 GPU 数） |

```json
{
  "zero_optimization": {
    "stage": 3,
    "offload_optimizer": {
      "device": "cpu"
    },
    "offload_param": {
      "device": "cpu"
    }
  }
}
```

```python
import deepspeed

model, optimizer, _, _ = deepspeed.initialize(
    model=model,
    model_parameters=model.parameters(),
    config="ds_config.json"
)
```

## 4. 方案对比

| 特性 | DP | DDP | DeepSpeed ZeRO-3 | FSDP |
|------|-----|-----|------------------|------|
| 架构 | 单进程 | 多进程 | 多进程 | 多进程 |
| 显存效率 | 低 | 中 | 高 | 高 |
| 扩展性 | 差 | 好 | 极好 | 好 |
| 易用性 | 简单 | 中等 | 中等 | 中等 |
| 适用场景 | 小模型 | 中等模型 | 大模型 | 大模型 |

## 5. 选型建议

- **小模型**（< 1B 参数）：DDP 足够
- **中等模型**（1B - 10B）：DDP + Gradient Checkpointing
- **大模型**（> 10B）：DeepSpeed ZeRO-3 或 FSDP

## 总结

分布式训练是大模型训练的基石。选择合适的方案需要综合考虑模型大小、硬件资源和开发效率。DeepSpeed 和 FSDP 在超大模型上表现出色，是目前的主流选择。

---

> 参考资料：
> - [PyTorch DDP 文档](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)
> - [DeepSpeed 官方文档](https://www.deepspeed.ai/)
> - [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](https://arxiv.org/abs/1910.02054)
