# 运行时状态聚类：自动发现故障模式的技术综述

> 研究日期：2026-04-27
> 项目：agent-cognitive-scaffolding
> 作者：Research Agent
> 关键词：在线聚类、异常检测、概念漂移、日志挖掘、表示学习、主动学习

---

## 摘要

本文面向 agent-cognitive-scaffolding 项目的"运行时状态聚类"需求，系统调研了从流式状态向量中自动发现新故障模式所涉及的关键技术领域。我们的 Skill 系统在 Phase 0 阶段产生六维状态向量（CLI 可用性、Gateway 进程、端口绑定、版本一致性、包完整性、域注册状态），目前的 Playbook 决策树依赖人工枚举所有状态组合。本研究旨在回答：**如何让系统自动从运行时数据中发现未被枚举的故障模式，并辅助人类专家创建新的 Playbook？**

本文依次调研在线聚类算法、时序异常检测、概念漂移检测、日志挖掘与模式发现、系统状态表示学习、主动学习六大技术领域，最后提出面向本项目的具体架构方案。

---

## 一、在线聚类算法

### 1.1 问题定义

我们的状态向量是离散的、流式到达的。每次 Agent 运行 Phase 0 诊断时，产生一个六维向量，如 `(CLI_OK, GATEWAY_STOPPED, PORT_FREE, VERSION_MISMATCH, PKG_INTACT, DOMAIN_UNLOADED)`。我们需要一种算法，能够：

- **增量处理**：无需存储全部历史数据即可更新聚类结果
- **自动发现新簇**：不预设簇数量 k
- **识别噪声/异常**：区分真正的新模式与偶发噪声
- **适应时间演化**：旧模式消亡、新模式出现

### 1.2 DenStream

DenStream（Cao et al., 2006）是密度聚类算法 DBSCAN 的流式扩展，堪称本领域奠基之作。其核心思想是维护两类微簇（micro-cluster）：

- **潜在微簇（p-micro-cluster）**：密度足够高，可能成为真正簇的候选
- **离群微簇（o-micro-cluster）**：密度不足但尚未被丢弃的观测集合

算法设计了一种"衰减函数"（fading function），使旧数据点的权重随时间指数衰减。当一个离群微簇累积到足够的权重时，它会被提升为潜在微簇——这恰好对应我们"发现新故障模式"的需求。在离线阶段，算法对潜在微簇运行变体 DBSCAN 得到最终聚类。

**适用性分析**：DenStream 天然支持新簇发现且不需预设 k。衰减机制适合处理系统演化（老版本故障模式自然消亡）。River 框架（riverml.xyz）提供了可直接使用的 Python 实现。

**局限**：需要调参（epsilon 邻域半径、最小权重阈值等），对高维数据效果下降。但我们的六维离散空间维度不高，问题不大。

### 1.3 BIRCH

BIRCH（Balanced Iterative Reducing and Clustering using Hierarchies）（Zhang et al., 1996）通过 CF-Tree（Clustering Feature Tree）实现增量聚类。每个叶节点存储一个子簇的统计摘要（数据点数量 N、线性和 LS、平方和 SS），而非原始数据。新数据点到达时，从根沿路径下降到最近的叶节点：

- 若加入后子簇半径不超过阈值 T，则吸收
- 否则分裂叶节点

**适用性分析**：BIRCH 的 CF-Tree 是天然的增量数据结构，内存受限且不需扫描全部数据。scikit-learn 提供成熟实现。但 BIRCH 本质上倾向于发现球形簇，且需要预设簇数量（或配合后续聚类步骤）。

### 1.4 HDBSCAN 与 Incremental HDBSCAN

HDBSCAN（Campello et al., 2013）扩展 DBSCAN 为层次化版本，通过构建最小生成树和压缩聚类层次树，自动选择不同密度层级的最优簇划分。它完全不需要预设 k，且能识别噪声点。

**增量挑战**：原始 HDBSCAN 是离线算法。近期研究（如 2025 年 arxiv:2601.20680）探索用 Online HDBSCAN 替代离线版本处理实时数据流。策略包括：

- 对新数据点增量更新最小生成树
- 批处理模式：积累一批数据后重建聚类层次
- 近似增量：仅更新受影响的局部子树

**推荐策略**：对我们的场景（数据量不大，每天可能几十到几百次 Phase 0 运行），采用"批次化 HDBSCAN"即可——每积累 N 个新向量后重新运行 HDBSCAN，计算开销在毫秒级别。

### 1.5 Mini-Batch K-Means 与 Growing Neural Gas

Mini-Batch K-Means（Sculley, 2010）是在线 K-Means 的变体，每次处理一个小批量。缺点是必须预设 k。

Growing Neural Gas（Fritzke, 1995）是一种自适应拓扑结构的增量聚类方法，可根据数据分布动态增减节点。理论上可用于发现新簇，但实现复杂度较高。

### 1.6 本节结论

| 算法 | 不需预设k | 增量更新 | 噪声处理 | 实现成熟度 | 推荐度 |
|------|-----------|----------|----------|-----------|--------|
| DenStream | ✅ | ✅ | ✅ | 中（River） | ⭐⭐⭐⭐ |
| BIRCH | ❌ | ✅ | ❌ | 高（sklearn） | ⭐⭐ |
| HDBSCAN（批次化） | ✅ | 半增量 | ✅ | 高（hdbscan lib） | ⭐⭐⭐⭐⭐ |
| Mini-Batch KMeans | ❌ | ✅ | ❌ | 高（sklearn） | ⭐ |
| Growing Neural Gas | ✅ | ✅ | 中 | 低 | ⭐⭐ |

**推荐方案**：以批次化 HDBSCAN 为主聚类算法，辅以 DenStream 做实时新簇候选监测。

---

## 二、时序状态数据的异常检测

### 2.1 问题场景

我们的状态向量虽然是离散的，但具有时序性质——相邻时间点的状态向量之间存在转移关系。异常检测的目标是识别"从未见过的状态组合"或"罕见的状态转移序列"。

### 2.2 Facebook Prophet

Facebook Prophet（Taylor & Letham, 2018）是一个针对时间序列预测的加法模型，擅长处理趋势、季节性和假日效应。其异常检测思路是：训练模型后，实际值超出预测置信区间即为异常。

**适用性**：Prophet 面向连续数值型时序数据（如 QPS、延迟），对我们的离散状态向量不直接适用。但可以将状态向量映射为一个"健康分数"（如已知 Playbook 能匹配则为 1.0，不能匹配则为 0.0），然后对该分数序列做 Prophet 预测。当"无法匹配"的频率异常升高时，提示可能出现了新故障模式。

### 2.3 Netflix 异常检测体系

Netflix 的异常检测实践包括多个系统：

- **RAD（Robust Anomaly Detection）**：基于鲁棒 PCA 的异常检测，将正常行为建模为低秩矩阵，异常为稀疏矩阵
- **Automated Canary Analysis**：A/B 测试式的异常检测，比较 canary 与 baseline 的指标分布差异
- **Spectator/Atlas**：实时指标收集与统计异常检测

**可借鉴思路**：Robust PCA 的思想可以应用——将历史状态矩阵分解为"正常模式"（低秩部分）和"异常事件"（稀疏部分），后者就是我们需要关注的新故障模式候选。

### 2.4 AWS CloudWatch Anomaly Detection

AWS CloudWatch 使用机器学习模型（基于 Random Cut Forest 算法）为每个指标建立正常行为基线，支持自动调整季节性模式。Random Cut Forest（Guha et al., 2016）是 Amazon 开发的在线异常检测算法，通过随机切割树来估计每个点的异常分数。

**适用性**：Random Cut Forest 支持流式更新，对我们的场景有参考价值。可以为每个新到达的状态向量计算异常分数，高异常分数的向量优先进入聚类分析。

### 2.5 状态转移异常

除了单点异常外，状态转移异常同样重要。例如，从 `HEALTHY` 直接跳到 `(CLI_BROKEN, PKG_MISSING)` 而没有经过 `(CLI_OK, PKG_PARTIAL)` 的中间状态，可能暗示一种新的破坏性故障路径。

建模方法：
- **马尔可夫链**：建立状态转移概率矩阵，低概率转移为异常
- **LSTM 序列预测**：类似 DeepLog 的思路，用 LSTM 预测下一个状态，预测失败为异常

### 2.6 本节结论

对于我们的离散状态向量场景，推荐组合方案：
1. **Random Cut Forest** 做实时异常分数计算（每个新向量）
2. **马尔可夫链** 检测异常状态转移
3. **频率统计** 追踪"无法匹配已知 Playbook"的比率变化

---

## 三、概念漂移检测

### 3.1 为什么需要概念漂移检测

系统随时间演化：新版本发布改变了正常状态表现、新的配置选项引入新的状态维度、基础设施变更使旧的故障模式消失。如果聚类模型不能检测到分布变化，它会产生两类错误：

- **虚警**：将新的正常模式误报为异常
- **漏报**：将真正的新故障混入已知簇中

### 3.2 ADWIN（ADaptive WINdowing）

ADWIN（Bifet & Gavalda, 2007）是最广泛使用的概念漂移检测器之一。核心思想：

1. 维护一个变长滑动窗口
2. 在窗口内寻找切分点：如果窗口的前半部分和后半部分的统计量差异超过阈值（基于 Hoeffding 不等式），则判定发生漂移
3. 发生漂移后，丢弃窗口前半部分（旧分布数据）

**适用性**：ADWIN 可以监控状态向量的各维度分布。例如，监控 `DOMAIN_UNLOADED` 状态出现频率——如果某次版本升级后该状态突然变得常见，ADWIN 会检测到分布变化，提示我们可能需要重新聚类或更新 Playbook。River 框架提供了 `river.drift.ADWIN` 的生产就绪实现。

### 3.3 Page-Hinkley Test

Page-Hinkley Test（Page, 1954; Hinkley, 1971）是一种序贯假设检验方法，检测均值的变化：

- 维护累积偏差 $U_T = \sum_{t=1}^{T}(x_t - \bar{x}_T - \delta)$
- 当 $\max(U_t) - U_T > \lambda$ 时触发漂移警报

其中 $\delta$ 是最小可接受变化量，$\lambda$ 是检测阈值。

**适用性**：适合监控连续指标（如"每日新簇数量"、"异常分数均值"），当这些宏观指标发生突变时触发模型重建。

### 3.4 DDM（Drift Detection Method）

DDM（Gama et al., 2004）基于分类器错误率的统计监控：

- 若错误率 $p_i + s_i$ 达到 $p_{min} + 2s_{min}$：进入警告状态
- 若错误率 $p_i + s_i$ 达到 $p_{min} + 3s_{min}$：确认漂移

**适用性**：如果我们用分类器做 Playbook 匹配，DDM 可以监控匹配错误率。当错误率上升，说明现有 Playbook 覆盖不足，需要发现新模式。

### 3.5 概念漂移响应策略

检测到漂移后如何响应？

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| 窗口重置 | 丢弃旧数据，仅用新窗口数据重新聚类 | 突变型漂移（如大版本升级） |
| 渐进更新 | 提高新数据权重，逐步淡化旧数据 | 渐变型漂移（如配置逐步迁移） |
| 集成方法 | 维护多个时期的模型，加权投票 | 周期性漂移（如工作日/周末模式） |
| 显式触发 | 外部事件（版本发布）主动重置 | 已知变更点 |

**推荐**：对我们的场景，采用"ADWIN + 显式触发"的组合策略。ADWIN 自动检测未知漂移，同时在已知事件（如 `openclaw update` 执行后）主动标记变更点。

---

## 四、日志挖掘与模式发现

### 4.1 从日志解析到状态模式

日志挖掘领域解决的核心问题——从海量非结构化文本中自动发现重复模式——与我们从结构化状态向量中发现新故障模式的需求高度同构。

### 4.2 Drain

Drain（He et al., 2017）是目前最受认可的在线日志解析算法。其核心数据结构是一棵固定深度的解析树（Fixed Depth Tree）：

- **第1层**：按日志长度分类
- **第2层**：按首词分类
- **后续层**：按 token 匹配度逐步细分
- **叶节点**：存储日志模板（template）

新日志到达时从根遍历到叶节点。如果与已有模板匹配度足够，合并；否则创建新模板。

**对本项目的启示**：我们可以借鉴 Drain 的"固定深度树"思想来组织状态向量的匹配逻辑。目前的 Playbook 决策树本质上就是一棵手工构建的 Drain 树。差异在于：Drain 自动从数据中学习树结构，而我们当前人工编写。

### 4.3 Spell

Spell（Du & Li, 2016）使用最长公共子序列（LCS）来匹配日志模板。新日志与现有模板集合计算 LCS，如果相似度超过阈值则归入该模板，否则创建新模板。

### 4.4 LogMine

LogMine（Hamooni et al., 2016）是一种分层聚类方法，自底向上合并相似日志行。其关键贡献是"模式层次化"——同一故障在不同抽象层级可以有不同的表示粒度。

**对本项目的启示**：我们的故障模式同样存在层次性。例如：
- 高层："安装相关故障"
- 中层："包文件不完整"
- 低层：`(CLI_OK, GATEWAY_STOPPED, PORT_FREE, VERSION_MISMATCH, PKG_PARTIAL, DOMAIN_UNLOADED)`

层次化组织有助于人类专家理解和审核。

### 4.5 DeepLog

DeepLog（Du et al., 2017, CCS）使用 LSTM 神经网络建模日志序列的正常执行路径：

1. 训练阶段：用正常运行日志训练 LSTM 预测下一条日志的 key
2. 检测阶段：如果实际日志 key 不在 LSTM 预测的 top-k 候选中，判为异常
3. 诊断阶段：用异常日志序列构建工作流图，定位根因

**对本项目的启示**：我们可以用类似的序列模型来学习"正常操作序列"（如 diagnose → match_playbook → execute → verify），当 Agent 的实际行为序列偏离预测时，可能发现了新的故障处理路径。

### 4.6 本节结论

日志挖掘的核心方法论——"先解析为结构化表示，再发现重复模式，最后识别新模式"——可以直接指导我们的系统设计。不同之处在于，我们的输入已经是结构化的状态向量，可以跳过解析步骤，直接进入模式发现阶段。

---

## 五、系统状态的表示学习

### 5.1 为什么需要表示学习

虽然我们的六维状态向量已经是结构化的，但直接在原始空间做聚类存在问题：

- **维度间相关性**：某些维度组合有特殊语义（如 PORT_BOUND + GATEWAY_STOPPED = "端口被其他进程占用"），直接欧氏距离无法捕捉
- **状态值的非均匀性**：CLI_BROKEN 和 CLI_OK 的"距离"不应与 PKG_INTACT 和 PKG_PARTIAL 相同
- **上下文依赖**：同一状态向量在升级前后可能有不同含义

### 5.2 自编码器（Autoencoder）

自编码器通过编码器-解码器结构学习数据的压缩表示。对运维数据的应用：

- **训练**：用正常状态向量训练自编码器
- **异常检测**：重建误差大的样本为异常
- **表示学习**：编码器输出的隐空间向量用于后续聚类

清华 NetMan 实验室的多项工作（Pei et al., 2020; Zhao et al., 2021）展示了自编码器在 AIOps 场景中的表示学习效果。

**对本项目的适用性**：当状态维度扩展到更多维度（日志特征、性能指标等）时，自编码器可以将高维特征压缩到低维隐空间，使聚类更加有效。当前六维可能过于简单，暂不需要。

### 5.3 变分自编码器（VAE）

VAE 的优势在于隐空间具有连续性和正则性，新样本在隐空间中的位置有概率解释。VAE 的 ELBO 损失中包含 KL 散度项，使得正常数据被约束在先验分布附近，异常数据自然偏离——这提供了一种优雅的异常检测机制。

### 5.4 图神经网络（GNN）表示

系统状态之间存在因果关系和依赖关系：

- Gateway 进程依赖 npm 包完整
- 端口绑定依赖 Gateway 进程
- 版本一致性依赖 CLI 和 plist 两个组件

这种依赖结构可以用图来表示。图自编码器（Graph Autoencoder, GAE）（Kipf & Welling, 2016）可以将带有结构信息的状态编码为向量：

```
状态图：
  CLI_OK ──depends──> PKG_INTACT
  GATEWAY_RUNNING ──depends──> PKG_INTACT
  PORT_BOUND ──depends──> GATEWAY_RUNNING
  VERSIONS_MATCH ──depends──> CLI_OK
  DOMAIN_LOADED ──depends──> GATEWAY_RUNNING
```

**对本项目的启示**：当系统复杂度增加（多个 Agent 并行操作、多服务依赖），图表示学习将成为必要。当前阶段可先用简单的 one-hot 编码 + 手工特征工程。

### 5.5 度量学习（Metric Learning）

传统欧氏距离对离散状态向量不太合理。度量学习（如 Siamese Network、Triplet Loss）可以学习一个距离函数，使得"语义相似的故障"在嵌入空间中更近。

训练信号来源：
- 相同 Playbook 处理的故障应该距离近
- 不同 Playbook 处理的故障应该距离远

这种方法需要历史标注数据，属于后期优化方向。

### 5.6 本节结论

对当前阶段（六维离散状态），推荐使用简单的编码方案：
- 每个维度 one-hot 编码，拼接为二进制向量
- 使用 Hamming 距离作为聚类度量

未来扩展路径：
- 维度 > 20 时引入自编码器
- 多服务场景引入图表示
- 积累足够标注后引入度量学习

---

## 六、主动学习与 Playbook 生成

### 6.1 问题定义

当聚类算法发现一个新簇时，它只知道"这组状态向量彼此相似且不属于已知模式"。我们需要：

1. 判断这是否是一个**真正的新故障模式**（而非噪声或偶发事件）
2. 如果是，提取其**核心特征和处理逻辑**
3. 生成**新的 Playbook 草案**供人类专家审核

主动学习（Active Learning）正是解决"如何最少地请求人类标注"的技术。

### 6.2 查询策略（Query Strategy）

主动学习的核心问题是"选哪些样本去问人"。常用策略包括：

**不确定性采样（Uncertainty Sampling）**：选择模型最不确定的样本。对我们来说：
- 选择与所有已知簇中心距离相近的状态向量
- 选择被不同聚类运行反复重新分配的状态向量

**多样性采样（Diversity Sampling）**：选择能代表新簇多样性的样本。具体方法：
- 在新簇中选择相互距离最远的 k 个代表性向量
- 确保呈现给专家的不是一堆几乎相同的案例

**信息增益（Expected Model Change）**：选择标注后对模型改变最大的样本。

**推荐**：对我们的场景，采用"多样性采样 + 时间覆盖"策略——从新簇中选择在不同时间点出现的、相互有差异的代表性样本，呈现给专家。

### 6.3 Few-Shot Learning for Playbook Generation

新簇可能只有少量样本（few-shot 场景）。如何从几个案例中生成 Playbook？

**基于模板的方法**：
- 从已有 Playbook 中提取结构模板（诊断 → 决策 → 执行 → 验证）
- 用新簇的特征填充模板变量

**基于 LLM 的方法**（2024-2026 前沿）：
- 将新簇的状态向量 + 时间上下文 + 系统日志片段作为 prompt
- 让 LLM 生成 Playbook 草案
- 人类专家审核修改

近期工作如 LLM-IRAgent（2025）探索了用 LLM 将安全事件响应手册自动转化为可执行 Playbook 的方法，证明了 LLM 在此类结构化知识生成任务中的有效性。

### 6.4 Human-in-the-Loop 工作流

完整的工作流应该是：

```
新簇发现 → 置信度评估 → [高置信度] → 自动生成草案 → 人工审核
                       → [低置信度] → 主动学习查询 → 人工标注 → 生成草案 → 人工审核
```

关键设计原则：
- **最小化人类负担**：每次只呈现最重要的候选
- **保留否决权**：所有自动生成的 Playbook 必须经过审核才能激活
- **反馈闭环**：人类的审核决策（接受/拒绝/修改）反馈回模型

### 6.5 本节结论

主动学习的引入将我们的系统从"完全人工枚举"进化为"机器提议 + 人类审核"。关键技术选择：
- 查询策略：多样性采样
- 生成方法：LLM 基于模板生成
- 工作流：分层置信度 + 人类审核闸门

---

## 七、面向本框架的架构方案

### 7.1 总体架构

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Runtime                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐       │
│  │  Phase 0  │───▶│State Vec │───▶│ Playbook     │       │
│  │ Diagnosis │    │ Producer │    │ Matcher      │       │
│  └──────────┘    └────┬─────┘    └──────┬───────┘       │
│                       │                  │               │
│                       │ emit             │ match_result  │
│                       ▼                  ▼               │
│              ┌────────────────────────────────┐          │
│              │     State Event Bus             │          │
│              └────────┬───────────────────────┘          │
└───────────────────────┼──────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│              State Clustering Daemon                      │
│                                                          │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐     │
│  │ Collector │─▶│ Anomaly   │─▶│ Cluster Engine   │     │
│  │ & Store   │  │ Scorer    │  │ (HDBSCAN/DenStr) │     │
│  └──────────┘  └───────────┘  └────────┬─────────┘     │
│                                          │               │
│  ┌──────────────┐  ┌───────────────────┐│               │
│  │ Drift        │  │ New Cluster       │◀┘               │
│  │ Detector     │  │ Evaluator         │                 │
│  │ (ADWIN)      │  │                   │                 │
│  └──────┬───────┘  └────────┬──────────┘                │
│         │                    │                           │
│         │ drift_alert        │ candidate_playbook        │
│         ▼                    ▼                           │
│  ┌────────────────────────────────────┐                  │
│  │        Review Queue                 │                  │
│  │  (Human Expert Interface)           │                  │
│  └────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 7.2 数据收集层

**收集内容**：

| 数据项 | 类型 | 来源 |
|--------|------|------|
| 六维状态向量 | 离散向量 | Phase 0 诊断输出 |
| 时间戳 | datetime | 系统时钟 |
| 触发上下文 | 枚举 | 用户请求类型（升级/启动/诊断/…） |
| Playbook 匹配结果 | string/null | 匹配到的 Playbook ID 或 null |
| 执行结果 | 枚举 | 成功/失败/部分成功 |
| Agent ID | string | 执行 Agent 标识 |
| 系统版本 | string | openclaw --version 输出 |

**存储方案**：
- 近期数据（30天）：SQLite 本地文件，支持快速查询
- 长期归档：append-only JSON Lines 文件

```python
# 数据记录结构示例
@dataclass
class StateEvent:
    timestamp: datetime
    state_vector: Tuple[str, str, str, str, str, str]  # 六维
    trigger_context: str
    matched_playbook: Optional[str]
    execution_result: str
    agent_id: str
    system_version: str
```

### 7.3 异常评分层

每个新到达的状态向量经过两级评分：

1. **点异常分数**（Random Cut Forest）：该向量在历史数据中的罕见程度
2. **序列异常分数**（马尔可夫转移概率）：从前一个状态到当前状态的转移是否罕见

评分结果决定后续处理路径：
- 分数 < 阈值低：正常，仅记录
- 阈值低 ≤ 分数 < 阈值高：可疑，加入聚类缓冲区
- 分数 ≥ 阈值高：高度异常，立即标记为关注候选

### 7.4 聚类引擎

**主算法：批次化 HDBSCAN**

```python
from hdbscan import HDBSCAN
import numpy as np

class StateClusterEngine:
    def __init__(self, min_cluster_size=3, min_samples=2):
        self.buffer = []
        self.known_clusters = {}  # cluster_id -> centroid + metadata
        self.clusterer = HDBSCAN(
            min_cluster_size=min_cluster_size,
            min_samples=min_samples,
            metric='hamming'  # 适合离散状态向量
        )
    
    def add_event(self, state_event: StateEvent):
        self.buffer.append(state_event)
        if len(self.buffer) >= self.batch_threshold:
            self._run_clustering()
    
    def _run_clustering(self):
        vectors = self._encode_vectors(self.buffer)
        labels = self.clusterer.fit_predict(vectors)
        new_clusters = self._identify_new_clusters(labels)
        for cluster in new_clusters:
            self._evaluate_and_enqueue(cluster)
```

**辅助算法：DenStream 实时监测**

```python
from river.cluster import DenStream

denstream = DenStream(
    decaying_factor=0.01,
    beta=0.5,
    mu=2.0,
    epsilon=0.3
)

# 每个新事件实时更新
for event in state_event_stream:
    vector = encode(event.state_vector)
    denstream.learn_one(vector)
    # 检查是否形成了新的潜在微簇
```

### 7.5 漂移检测层

```python
from river.drift import ADWIN

# 为每个状态维度维护一个 ADWIN 检测器
drift_detectors = {
    'cli': ADWIN(delta=0.002),
    'gateway': ADWIN(delta=0.002),
    'port': ADWIN(delta=0.002),
    'version': ADWIN(delta=0.002),
    'package': ADWIN(delta=0.002),
    'domain': ADWIN(delta=0.002),
}

# 也为 "无法匹配 Playbook" 比率维护检测器
unmatched_rate_detector = ADWIN(delta=0.001)

def on_state_event(event: StateEvent):
    for dim, detector in drift_detectors.items():
        value = encode_dim(event.state_vector, dim)
        detector.update(value)
        if detector.drift_detected:
            trigger_recluster(reason=f"drift in {dim}")
    
    unmatched = 1.0 if event.matched_playbook is None else 0.0
    unmatched_rate_detector.update(unmatched)
    if unmatched_rate_detector.drift_detected:
        alert_new_pattern_emergence()
```

### 7.6 新簇评估与 Playbook 候选生成

当聚类引擎发现新簇时，执行以下评估流程：

```
1. 簇稳定性检验
   - 簇在连续 3 次聚类运行中都出现？
   - 簇大小 ≥ 最小阈值（如 3 个事件）？
   
2. 新颖性检验
   - 与所有已知 Playbook 的状态模式计算距离
   - 最近距离 > 阈值 → 确认为新模式
   
3. 可操作性评估
   - 簇中事件是否有共同的失败结果？
   - 是否存在自然的处理方向？
   
4. 候选 Playbook 生成
   - 提取簇的核心状态模式（多数投票）
   - 提取关联的上下文信息
   - 用 LLM 生成 Playbook 草案
```

### 7.7 人类审核界面

审核界面需要向专家展示：

```markdown
## 新发现的故障模式候选 #42

**状态模式**: (CLI_OK, GATEWAY_RUNNING, PORT_BOUND, VERSION_MISMATCH, PKG_INTACT, DOMAIN_LOADED)
**首次出现**: 2026-04-20 14:32:00
**出现次数**: 7 次（过去 7 天）
**触发上下文**: 主要在 "upgrade" 操作后出现
**现有最近 Playbook**: P1a（HEALTHY + upgrade），距离 = 1（仅 VERSION_MISMATCH 不同）

**代表性事件**:
1. 2026-04-20 14:32 — Agent-3 — upgrade 后 CLI 报告新版本但 plist 未更新
2. 2026-04-22 09:15 — Agent-1 — 同上，重启后自愈
3. ...

**LLM 生成的草案 Playbook**:
> Label: POST_UPGRADE_VERSION_DESYNC
> 症状: 升级完成但版本号不一致，Gateway 仍在运行旧版本
> 处理: openclaw gateway install && openclaw gateway restart
> 验证: openclaw --version 与 Gateway 日志版本一致

**操作**: [接受并编辑] [标记为已知模式的变体] [标记为噪声] [需要更多数据]
```

### 7.8 实施路线图

| 阶段 | 目标 | 技术选型 | 预计工作量 |
|------|------|---------|-----------|
| Phase 1 | 数据收集基础设施 | SQLite + JSON Lines | 2-3 天 |
| Phase 2 | 离线聚类分析 | HDBSCAN + Hamming | 2-3 天 |
| Phase 3 | 漂移检测 | ADWIN (River) | 1-2 天 |
| Phase 4 | 异常评分 | Random Cut Forest | 2-3 天 |
| Phase 5 | 新簇评估流程 | 规则引擎 + LLM | 3-5 天 |
| Phase 6 | 人类审核界面 | CLI/Markdown 报告 | 2-3 天 |
| Phase 7 | 闭环反馈 | 标注数据回流 | 1-2 天 |

---

## 八、总结与展望

### 8.1 核心结论

1. **算法选型**：批次化 HDBSCAN + DenStream 的组合方案最适合我们的场景。HDBSCAN 提供准确的簇结构发现，DenStream 提供实时监测能力。

2. **表示方案**：当前六维离散空间使用 Hamming 距离即可。未来维度扩展后考虑自编码器或图表示。

3. **漂移适应**：ADWIN 作为主要漂移检测器，配合显式版本变更触发器。

4. **模式发现**：借鉴 Drain 的分层匹配思想，但利用我们输入已结构化的优势直接进入模式聚类。

5. **人类协作**：主动学习的多样性采样策略 + LLM 辅助 Playbook 生成，最大限度减少人类标注负担。

### 8.2 预期收益

- **覆盖率提升**：从人工枚举的有限 Playbook 扩展到数据驱动的自动发现
- **响应速度**：新故障模式从"多次发生后人类注意到"变为"自动检测并提议"
- **知识积累**：每个被确认的新簇都丰富了 Skill 的决策树
- **版本适应**：概念漂移检测使系统自动适应软件演化

### 8.3 开放问题

- **冷启动**：系统初期数据量不足以进行有意义的聚类，需要设计冷启动策略
- **评估指标**：如何衡量"新模式发现"的质量？需要定义 precision/recall 的操作化指标
- **隐私与安全**：状态向量是否包含敏感信息？聚类过程是否需要差分隐私保护？
- **多 Agent 协作**：多个 Agent 的状态数据如何安全合并？

---

## 参考文献

1. Cao, F., Ester, M., Qian, W., & Zhou, A. (2006). Density-Based Clustering over an Evolving Data Stream with Noise. *SDM 2006*.
2. Zhang, T., Ramakrishnan, R., & Livny, M. (1996). BIRCH: An Efficient Data Clustering Method for Very Large Databases. *SIGMOD 1996*.
3. Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). Density-Based Clustering Based on Hierarchical Density Estimates. *PAKDD 2013*.
4. Sculley, D. (2010). Web-Scale K-Means Clustering. *WWW 2010*.
5. Fritzke, B. (1995). A Growing Neural Gas Network Learns Topologies. *NIPS 1995*.
6. Taylor, S. J., & Letham, B. (2018). Forecasting at Scale. *The American Statistician, 72*(1), 37-45.
7. Guha, S., Mishra, N., Roy, G., & Schrijvers, O. (2016). Robust Random Cut Forest Based Anomaly Detection on Streams. *ICML 2016*.
8. Bifet, A., & Gavalda, R. (2007). Learning from Time-Changing Data with Adaptive Windowing. *SDM 2007*.
9. Page, E. S. (1954). Continuous Inspection Schemes. *Biometrika, 41*(1/2), 100-115.
10. Gama, J., Medas, P., Castillo, G., & Rodrigues, P. (2004). Learning with Drift Detection. *SBIA 2004*.
11. He, P., Zhu, J., Zheng, Z., & Lyu, M. R. (2017). Drain: An Online Log Parsing Approach with Fixed Depth Tree. *ICWS 2017*.
12. Du, M., & Li, F. (2016). Spell: Streaming Parsing of System Event Logs. *ICDM 2016*.
13. Hamooni, H., Debnath, B., Mueen, A., Xu, J., & Zhang, K. (2016). LogMine: Fast Pattern Recognition for Log Analytics. *CIKM 2016*.
14. Du, M., Li, F., Zheng, G., & Srikumar, V. (2017). DeepLog: Anomaly Detection and Diagnosis from System Logs through Deep Learning. *CCS 2017*.
15. Kipf, T. N., & Welling, M. (2016). Variational Graph Auto-Encoders. *NeurIPS Workshop on Bayesian Deep Learning*.
16. Settles, B. (2009). Active Learning Literature Survey. *Computer Sciences Technical Report 1648, University of Wisconsin-Madison*.
17. Montiel, J., Halford, M., Mastelini, S. M., et al. (2021). River: Machine Learning for Streaming Data in Python. *JMLR, 22*(110), 1-8.
18. Pei, D., et al. (2020). AIOps: Artificial Intelligence for IT Operations. *Tsinghua NetMan Lab Technical Report*.
19. Zhao, N., et al. (2021). Identifying Bad Software Changes via Multimodal Anomaly Detection for Online Service Systems. *ESEC/FSE 2021*.
20. Gong, D., Zhou, H., & Zhang, Y. (2018). Clustering Stream Data by Exploring the Evolution of Density Mountain. *VLDB 2018*.

---

> 本报告为 agent-cognitive-scaffolding 项目研究系列第 02 篇。
> 前序：`docs/04-cross-review-report.md`（交叉评审报告）
> 后续规划：实现 Phase 1 数据收集原型
