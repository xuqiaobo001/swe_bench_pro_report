# SWE-bench_Pro 测试用例分类分析报告

> 数据来源：[ScaleAI/SWE-bench_Pro](https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro)
>
> 分析日期：2026-04-01

---

## 概述

SWE-bench_Pro 是一个用于评估大模型编码能力的测试用例集，共包含 **731 个**测试用例，覆盖 **11 个**真实的开源项目仓库。

---

## 1. 语言覆盖情况

共覆盖 **4 种**主流编程语言：

| 语言 | 用例数 | 占比 | 趋势 |
|------|--------|------|------|
| **Go** | 280 | 38.3% | ████████████████ |
| **Python** | 266 | 36.4% | ███████████████▌ |
| **JavaScript** | 165 | 22.6% | █████████▌ |
| **TypeScript** | 20 | 2.7% | █ |

**分析：**
- Go 和 Python 占据主导地位，合计占比 **74.7%**
- JavaScript 作为前端主流语言也有相当比重
- TypeScript 用例较少，可能是因为 TS 项目通常也包含大量 JS 代码

---

## 2. 每种语言涉及的场景分布

### 场景定义

| 场景 | 说明 |
|------|------|
| **0~1项目构建** | 从零开始实现新功能、新模块 |
| **代码/特性续写** | 扩展现有功能、增强已有特性 |
| **重构** | 代码重组、架构优化、迁移 |
| **Bug Fix** | 修复错误、解决缺陷 |

### 各语言场景分布详情

#### Go（280 个用例）

| 场景 | 数量 | 占比 |
|------|------|------|
| Bug Fix | 176 | 62.9% |
| 代码/特性续写 | 55 | 19.6% |
| 0~1项目构建 | 37 | 13.2% |
| 重构 | 12 | 4.3% |

#### Python（266 个用例）

| 场景 | 数量 | 占比 |
|------|------|------|
| Bug Fix | 172 | 64.7% |
| 代码/特性续写 | 50 | 18.8% |
| 0~1项目构建 | 28 | 10.5% |
| 重构 | 16 | 6.0% |

#### JavaScript（165 个用例）

| 场景 | 数量 | 占比 |
|------|------|------|
| Bug Fix | 71 | 43.0% |
| 代码/特性续写 | 43 | 26.1% |
| 0~1项目构建 | 38 | 23.0% |
| 重构 | 13 | 7.9% |

#### TypeScript（20 个用例）

| 场景 | 数量 | 占比 |
|------|------|------|
| Bug Fix | 16 | 80.0% |
| 0~1项目构建 | 3 | 15.0% |
| 代码/特性续写 | 1 | 5.0% |
| 重构 | 0 | 0.0% |

### 场景分布汇总

```
          Go    Python  JavaScript  TypeScript
Bug Fix   ████████████ ████████████ ████████    ████████████████
续写      ███         ███          █████       █
构建      ██          ██           ████        ███
重构      █           █            █           -
```

**关键发现：**
- 所有语言都以 **Bug Fix** 场景为主导（43%~80%）
- JavaScript 在新功能构建方面比例最高（23%）
- TypeScript 样本量较小，场景分布可能不够代表性
- 重构场景在各语言中占比都较低（<8%）

---

## 3. 每种场景下的基础代码仓代码量

### 代码量估算方法

基于 GitHub Languages API 返回的代码字节数，除以平均每行代码字节数（约 50 字节）进行估算。

### 场景代码量汇总

| 场景 | 用例数 | 涉及仓库数 | 平均代码量 | 中位数代码量 |
|------|--------|------------|------------|--------------|
| **0~1项目构建** | 87 | 10 | ~35.6万行 | ~11.0万行 |
| **代码/特性续写** | 213 | 11 | ~38.4万行 | ~11.0万行 |
| **重构** | 40 | 10 | ~35.6万行 | ~11.0万行 |
| **Bug Fix** | 391 | 11 | ~38.4万行 | ~11.0万行 |

### 各仓库代码量详情

| 仓库 | 估算代码量 | 仓库大小 | 主要语言 | 用例数 |
|------|------------|----------|----------|--------|
| gravitational/teleport | ~132万行 | 1.3 GB | Go | 76 |
| protonmail/webclients | ~118万行 | 3.0 GB | JavaScript | 65 |
| tutao/tutanota | ~66万行 | 262 MB | TypeScript | 20 |
| element-hq/element-web | ~32万行 | 503 MB | JavaScript | 56 |
| ansible/ansible | ~22万行 | 258 MB | Python | 96 |
| qutebrowser/qutebrowser | ~11万行 | 63 MB | Python | 79 |
| navidrome/navidrome | ~11万行 | 51 MB | Go | 57 |
| internetarchive/openlibrary | ~10万行 | 108 MB | Python | 91 |
| NodeBB/NodeBB | ~10万行 | 102 MB | JavaScript | 44 |
| future-architect/vuls | ~6万行 | 31 MB | Go | 62 |
| flipt-io/flipt | ~5万行 | 102 MB | Go | 85 |

### 各场景涉及的仓库分布

#### 0~1项目构建（87 个用例）

| 仓库 | 估算代码量 | 用例数 |
|------|------------|--------|
| gravitational/teleport | ~132万行 | 11 |
| protonmail/webclients | ~118万行 | 5 |
| element-hq/element-web | ~32万行 | 16 |
| ansible/ansible | ~22万行 | 9 |
| qutebrowser/qutebrowser | ~11万行 | 2 |
| navidrome/navidrome | ~11万行 | 6 |
| internetarchive/openlibrary | ~10万行 | 12 |
| NodeBB/NodeBB | ~10万行 | 9 |
| future-architect/vuls | ~6万行 | 2 |
| flipt-io/flipt | ~5万行 | 15 |

#### 代码/特性续写（213 个用例）

| 仓库 | 估算代码量 | 用例数 |
|------|------------|--------|
| gravitational/teleport | ~132万行 | 16 |
| protonmail/webclients | ~118万行 | 24 |
| tutao/tutanota | ~66万行 | 6 |
| element-hq/element-web | ~32万行 | 24 |
| ansible/ansible | ~22万行 | 13 |
| qutebrowser/qutebrowser | ~11万行 | 27 |
| navidrome/navidrome | ~11万行 | 20 |
| internetarchive/openlibrary | ~10万行 | 31 |
| NodeBB/NodeBB | ~10万行 | 11 |
| future-architect/vuls | ~6万行 | 22 |
| flipt-io/flipt | ~5万行 | 19 |

#### 重构（40 个用例）

| 仓库 | 估算代码量 | 用例数 |
|------|------------|--------|
| gravitational/teleport | ~132万行 | 1 |
| protonmail/webclients | ~118万行 | 8 |
| element-hq/element-web | ~32万行 | 1 |
| ansible/ansible | ~22万行 | 2 |
| qutebrowser/qutebrowser | ~11万行 | 3 |
| navidrome/navidrome | ~11万行 | 6 |
| internetarchive/openlibrary | ~10万行 | 7 |
| NodeBB/NodeBB | ~10万行 | 8 |
| future-architect/vuls | ~6万行 | 1 |
| flipt-io/flipt | ~5万行 | 3 |

#### Bug Fix（391 个用例）

| 仓库 | 估算代码量 | 用例数 |
|------|------------|--------|
| gravitational/teleport | ~132万行 | 48 |
| protonmail/webclients | ~118万行 | 28 |
| tutao/tutanota | ~66万行 | 14 |
| element-hq/element-web | ~32万行 | 15 |
| ansible/ansible | ~22万行 | 72 |
| qutebrowser/qutebrowser | ~11万行 | 47 |
| navidrome/navidrome | ~11万行 | 25 |
| internetarchive/openlibrary | ~10万行 | 41 |
| NodeBB/NodeBB | ~10万行 | 16 |
| future-architect/vuls | ~6万行 | 37 |
| flipt-io/flipt | ~5万行 | 48 |

---

## 4. 知识领域覆盖

测试用例还涉及多个知识领域（issue_categories）：

| 知识领域 | 出现次数 |
|----------|----------|
| back_end_knowledge | 560 |
| api_knowledge | 238 |
| devops_knowledge | 156 |
| ui_ux_knowledge | 137 |
| security_knowledge | 127 |
| front_end_knowledge | 123 |
| infrastructure_knowledge | 122 |
| web_knowledge | 118 |
| database_knowledge | 87 |
| authentication_authorization_knowledge | 76 |
| performance_knowledge | 67 |
| desktop_knowledge | 59 |
| full_stack_knowledge | 31 |
| networking_knowledge | 31 |
| cloud_knowledge | 23 |
| accessibility_knowledge | 10 |
| ds_knowledge | 6 |
| mobile_knowledge | 2 |
| ml_ai_knowledge | 2 |
| iot_embed_knowledge | 1 |
| blockchain_knowledge | 1 |

---

## 5. 关键结论

### 数据集特点

1. **语言覆盖全面**：涵盖 Go、Python、JavaScript、TypeScript 四种主流语言
2. **场景真实**：Bug Fix 占比最高（53.5%），符合真实开发场景
3. **项目规模多样**：从 5 万行到 132 万行，涵盖中小型到大型项目
4. **领域广泛**：涉及后端、前端、DevOps、安全等 21 个知识领域

### 对模型能力的挑战

1. **代码理解深度**：需要在大型代码仓中定位问题
2. **上下文长度**：基础仓库动辄数十万行代码
3. **多领域知识**：单个问题可能涉及多个知识领域
4. **真实场景复杂性**：基于真实 GitHub Issue，问题描述可能不够精确

### 建议

- 对于 Bug Fix 场景，模型需要具备较强的代码调试和问题定位能力
- 对于 0~1 项目构建，模型需要理解项目架构和编码规范
- 建议按语言和场景分批次评估，以获得更细粒度的能力画像

---

## 附录：数据文件

- `test-00000-of-00001.parquet` - 测试数据集（731 条记录）
- `eval.yaml` - 评估配置文件
- `README.md` - 数据集说明文档

---

*报告生成工具：Claude Code*
