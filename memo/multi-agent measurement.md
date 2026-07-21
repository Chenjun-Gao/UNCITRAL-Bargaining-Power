# D-MATI v3：Agent 标注 + 极简数学的多边谈判文本影响力测量

> 版本关系：v1（`memo/多边谈判文本影响力测量研究设计_2026-07-12.md`）建立了三个量的框架；
> v2（`D-MATI_v2_简化标注_加强数学`）把复杂度从标注搬到数学（OT + Sinkhorn + 多状态生存 + 蒙特卡洛 Shapley）。
> **用户对 v2 的核心不满**：(a) 数学太重、太难；(b) 标注仍含主观项——`legal_importance∈{1,2,3}`
> 标注员之间无法一致，`survives` 要人工对照终稿太麻烦；(c) LLM 用得不够、agent 完全没用。
>
> **v3 的一句话**：**把"判断"从人搬到 agent（可复现、带证据、带不确定性），把"数学"降到三件本科工具，
> 人只做二元核查。** 主观评分改为 agent 的成对比较（更可靠）；存活状态由确定性谱系脚本自动算；
> 归因由"多智能体辩证 + 引用回溯"产出带校准概率的分配，天然保留 secretariat / no-country sink。

---

## 0. 为什么 agent 能同时让标注更简单、结果更好

三条来自方法论文献的依据（不是"因为 LLM 强"）：

1. **LLM 已被证明在文本标注上可达到或超过众包一致性**（Gilardi, Alizadeh & Kubli 2023, *PNAS*；
   Törnberg 2023；Chae & Davidson 2023）。所以把"是不是提案""朝哪个方向"交给模型有据可依，
   但**必须用人工金标准报告 agreement/precision**，不能盲信。
2. **成对比较（comparative judgment）比绝对打分更可靠、更易达成一致**（Thurstone 1927；
   Bradley & Terry 1952；教育测量中大量复现）。这直接解决 v2 的 `legal_importance∈{1,2,3}`：
   不再让任何人（或模型）打 1/2/3 绝对分，而是问"A、B 两条编辑哪条更改变权利义务？"，
   再用**你们 Round2 已经在用的 Bradley–Terry** 把成对结果标度成连续重要度。零新数学栈。
3. **多智能体分工/辩证 + LLM-as-judge 能产出带校准与不确定性的裁决**（Du et al. 2023 debate；
   Zheng et al. 2023 LLM-as-judge/MT-Bench；Wang et al. 2023 self-consistency）。归因这一最难、
   最易泄漏"强国先验"的环节，用**正方 agent（此编辑来自提案 P）× 反方 agent（这是秘书处综合/无来源）
   × 裁判 agent（出概率分布）** 来做，sink 与不确定性自然落地，且每步都有证据 span 可审计。

**agent 相对"单次 LLM 调用"多出的、本问题真正需要的能力**：
- **工具化多跳检索**：秘书处注释里写着 `reflects a suggestion … (A/CN.9/1160, para. 21)`。
  agent 能**顺着这个引用去读该届会议纪要**，找出秘书处自己承认的来源方——这是**联合国官方记录的
  客观 provenance 通道**，不是模型意见。这是全语料里最被低估的资产（见 §3.1）。
- **角色隔离的集成**：K 个独立裁判 → 一致=高置信、分歧=转人工，把不确定性做成可复现的采样量。
- **分步留痕**：每个 agent 输出都带 `evidence_span + confidence`，校验 agent 检查"引用的 span
  是否真的支持结论"，杜绝幻觉式归因。

> 边界纪律（贯穿全文）：**agent 只产出模型输入（provenance 概率、联盟、重要度成对、存活），
> 三个"量"仍由透明统计模型估计**。绝不让 agent 直接吐出"哪国影响力最大"的名次——那会把结论
> 变成模型意见而非可识别参数。国名在归因阶段一律遮蔽，防止强国先验写进匹配。

---

## 一、标注怎么再简化：人只做"二元核查"，agent 做"判断 + 结构"

### 三层责任划分（这是 v3 的骨架）

| 谁 | 做什么 | 产物 | 为什么它做最合适 |
|---|---|---|---|
| **确定性脚本**（无模型，可复现） | 切版本、对齐条款、算跨版存活、过滤体例改动、抽取秘书处脚注里的 `A/CN.9/xxx` 引用 | `edits_master`、`clause_genealogy`、`footnote_citations` | 纯规则，零主观，100% 复现 |
| **Agent 群**（固定 model+temp=0+seed，留全轨迹） | 提案抽取、目标条款接地、联盟立场、provenance 辩证裁决、重要度成对比较 | 带 `evidence_span+confidence` 的结构化标签 | 判断类、需检索/多跳、易主观——正是模型强于人处 |
| **人工**（唯一任务：核查，二元） | 对 agent 输出的**分层抽样**做 `agree∈{0,1}`；对 agent `confidence=low / 裁判分歧` 的样本做仲裁 | 金标准验证集、agreement/precision 报告 | 二元核查一致性远高于让人产出结构；成本降一个数量级 |

**关键转变**：v2 让人**产出**（打 1/2/3、判存活、连链接、编联盟）；v3 让人**核查**（agent 说这条是提案、
朝放宽、来自 P，人只点"同意/不同意"）。核查是二元、单样本自足的判断，Kappa 天然高、可外包、可复用。

### 人工到底还标什么（全部是"看一眼点是/否"）

- **验证集 V1（提案检测）**：分层抽 ~300–500 个 turn，人核查 agent 的
  `has_proposal / operation / target_article` 是否正确。→ 报 proposal detector 的 P/R/F1。
- **验证集 V2（provenance 链接）**：抽 ~150 对 (edit, top-1 候选提案)，人核查"这条编辑是否真由该提案促成"
  ∈{是 / 部分 / 否-秘书处综合 / 否-无关}。→ 校准裁判 agent 的概率、报 top-k precision 与 Brier。
- **验证集 V3（重要度成对）**：抽 ~100 对编辑，人做"哪条更改变权利义务"的成对判断（**只比大小，不打分**）。
  → 与 agent 成对结果算 agreement，验证 BT 重要度标度可信。
- **仲裁队列**：所有 `confidence=low` 或 K 裁判不一致的样本进人工，人给最终标签。

> 没有任何一处要人打 1/2/3 绝对分，也没有任何一处要人从头连链接或编联盟。存活对照终稿由脚本做。


---

## 二、多智能体架构（可复现、带证据、可审计）

所有 agent 共享**复现契约**：固定 `model_id`、`temperature=0`（集成除外，见下）、固定 `seed`、
`prompt_hash`；每步写 `trace.jsonl`（输入 span、检索命中、输出、confidence）；
输出若无法引用支持性 `evidence_span` 一律判 `abstain`。国名在 §2.3 归因中一律用占位符替换。

### 2.1 Genealogy Agent（谱系 + 存活，替代 v2 人工谱系）
输入：相邻两版 ADV 文本 + 确定性 diff。任务：把每版切成"原子法律命题"，跨版对齐（条款号 + 语义），
分类 `unchanged/added/deleted/narrowed/broadened/replaced/bracketed/resolved`，**并自动追踪每命题是否留到
WP.238（= `survives`，脚本级确定，不要人标）**。配一个 **Adversary Agent** 专挑"漏对齐/错并"的命题，
分歧进仲裁。→ 产出 `clause_genealogy.csv`。

### 2.2 Proposal Agent（提案抽取 + 接地）
输入：一个 `relevance=1` 的 turn + 前后文（消解指代）+ 被引条款的草案文本（检索接地）。
任务：抽原子提案，填 `operation / target_article / stance_sign(-1,0,+1，只标符号)`，给 `proposal_span`。
**集成**：跑 K=5 次（变 seed / 轻微改写 prompt）→ 多数票定标签、票型离散度 = `confidence`（self-consistency）。
→ 产出 `proposals.csv`。人工只在 V1 上核查。

### 2.3 Provenance Court（核心创新：辩证 + 引用回溯，替代 OT）
对每条**存活的实质编辑** e，脚本先用 bi-encoder 在**同主题、同条款、时间早于 e** 的提案里召回候选集
C_e（防未来信息泄漏）。然后三类 agent（**国名全遮蔽**）：
- **Citation-Follower（客观通道，优先）**：解析 e 所在秘书处注释里的 `A/CN.9/xxx, para.` 引用，
  **去读那届会议纪要**，抽取秘书处自己写明的来源方/来源提案。命中即为强证据（官方自述，非模型猜测）。
- **Advocate**：论证"e 实现了候选 p"，必须引 e 与 p 的对应 span。
- **Null-Advocate**：论证"e 是秘书处综合 / 整体共识 / 无可观测来源"（= sink），必须给理由。
- **Judge（K 个独立实例）**：读三方证据，输出 e 在 `{p1..pk, sink}` 上的**概率分布** `q_{pe}`（Σ=1）。
  K 个裁判的均值 = 归因概率，方差 = 不确定性；分歧大 → 进 V2 人工仲裁。
→ 产出 `provenance.csv`（每条 e 的软分配 + sink 概率 + 证据链）。**质量守恒与 sink 由"概率分布 + 显式 sink 类"
直接保证，无需 Sinkhorn。**

### 2.4 Coalition Agent（联盟由读取产生，替代人工联盟编码）
对提案 p，agent 读此后同 (主题, 条款) 的发言，抽每个发言方的 `stance∈{support/oppose/modify/neutral}`
及证据 span。联盟 `S_p` = 明确 support/modify 且方向一致者。→ 产出 `coalitions.csv`。
联盟特征（规模、跨地区广度用发言方区域表、θ 立场跨度、桥接中心性）由脚本算。

### 2.5 Importance via Pairwise（替代主观 1/2/3）
不打绝对分。**Judge agent 做成对比较**：随机配对存活编辑，问"哪条更改变权利义务/准入门槛？"，
每对多裁判投票。把成对胜负喂 **Bradley–Terry（复用 Round2 代码）** → 每条编辑一个连续 `importance_e`（带 SE）。
人工只在 V3 上核查成对方向。**这是把主观评分变客观标度的关键一招，且不引入新数学。**

---

## 三、数学层：只用三件本科工具（对比 v2 砍掉 OT / Sinkhorn / 多状态生存 / 采样 Shapley）

记号：编辑 $e$，候选提案 $p$，国家 $i$，会议届次 $t$。所有解释变量只用 $\le t$ 的信息。

### 3.1 归因概率（agent 直接给，不解 OT）
Provenance Court 的 K 个裁判对编辑 $e$ 给出分布，取均值：
$$q_{pe}=\frac{1}{K}\sum_{k=1}^{K} \text{Judge}_k(p\mid e),\qquad \sum_{p\in C_e} q_{pe} + q_{\text{sink},e}=1 .$$
$q_{\text{sink},e}$ 就是"秘书处综合/无国家来源"的质量，**天然保留 no-country 类别**。
不确定性 $\;\widehat{\text{sd}}(q_{pe})=\text{sd}_k \text{Judge}_k(p\mid e)$。若 Citation-Follower 命中官方来源，
则对该 $p$ 施加强先验（在 prompt 中作为证据给裁判，并在敏感性分析里报告"去掉官方引用后"的名次变化）。

### 3.2 提案转化权：一个 logistic + 国家随机效应（替代五状态生存）
把生命周期**压成一个客观二元结局**：提案 $p$ 的诉求是否**进入某后续草案并留到 WP.238**（脚本从谱系算，记 $y_p\in\{0,1\}$）。
$$\text{logit}\,\Pr(y_p=1)=\alpha + b_{c(p)} + X_p^\top\gamma + g(S_p)^\top\delta + \text{article FE} + \text{session FE},\quad b_i\sim\mathcal N(0,\tau^2).$$
- $b_i$：国家 $i$ 的**层级随机效应（强收缩先验）**——数据少的国家自动得宽后验，不硬排名次。
- $X_p$：具体性、是否给出可直接起草文字、与现状距离、提出时点、主题显著性（全部 $\le t$，agent/脚本可得）。
- **反泄漏铁律**：Round2 时间坍缩的静态 $\theta$ **禁止进 $X_p$**；只可作**第二阶段**解释变量（解释为何 $b_i$ 不同）。
- **提案转化权 = $b_i$ 的后验**（不是发言数、不是终稿相似度）。用 `brms`/`PyMC` 一行分层 logit 即可，无需自写 MCMC。

### 3.3 边际文本影响：等分信用规则（替代蒙特卡洛 Shapley）
对每条存活编辑 $e$，其可分配总量 $=\text{importance}_e$（§2.5 的 BT 标度）。按归因概率 $q_{pe}$ 落到提案 $p$，
再在"提案者 + 联盟"间分配。用**预注册的等分规则**：提案者得固定份额 $\rho$，其余 $(1-\rho)$ 在
支持联盟 $S_p$ 成员间**等分**：
$$\phi_{ie}=\sum_{p}\ q_{pe}\Big[\rho\,\mathbb 1\{i=\text{proposer}(p)\}+\frac{1-\rho}{|S_p|}\,\mathbb 1\{i\in S_p\}\Big],\qquad
\text{MI}_i=\sum_e \phi_{ie}.$$
> **为什么这等价于 Shapley 又更简单**：当联盟成员对结局的贡献可交换（对称）且价值可加时，
> Shapley 值就退化为**等分**。所以等分规则是 Shapley 在"对称+可加"假设下的闭式解，
> 无需指数级枚举或蒙特卡洛采样——把假设写明即可，$\rho$ 作敏感性参数。
> 不确定性：对 $(q_{pe},\,\text{importance}_e,\,S_p)$ 做 bootstrap，报 $\text{MI}_i$ 的区间。

### 3.4 动态立场（防泄漏，替代静态 BT 直接入模）
仍复用 Round2 的成对数据，但让立场随会议游走：
$$\Pr(y_{ijkt}=1)=\text{logistic}(\theta_{ikt}-\theta_{jkt}+\text{order}+\text{annotator}),\quad
\theta_{ikt}\sim\mathcal N(\theta_{i,k,t-1},\sigma_k^2).$$
解释 $t\!\to\!t\!+\!1$ 版只用 $\theta_{i,k,\le t}$。这是 §3.2 里"与现状距离"等特征的合法来源。

### 3.5 三个量、三种报告（与 v1/v2 一致，不退化）
| 量 | 定义 | 来源 |
|---|---|---|
| 偏好实现 preference attainment | 终稿离谁 $\theta$ 近 | §3.4 + 终稿位置；**结果接近度，非因果** |
| 提案转化权 proposal-conversion | $b_i$ 后验 | §3.2，控制内容/机会/联盟后的身份效应 |
| 总边际文本影响 $\text{MI}_i$ | $\sum_e\phi_{ie}$ | §3.3，实际产生的文本影响总量 |

三者一致时"该国影响大"最可信；同时报 $b_i$、$\text{MI}_i$ 及反事实删除
$\Delta_i=D(\mathbb E[\text{Final}\mid\text{all}],\ \mathbb E[\text{Final}\mid\text{remove }i])$。

---

## 四、可复现性与可追溯性（用 LLM/agent 的硬约束）

1. **冻结配置**：记录 `model_id + provider + temperature + top_p + seed + prompt_hash + tool_versions`，
   随论文附录与代码仓一并发布。所有确定性步骤（diff、谱系存活、引用抽取、召回）用固定随机种子的脚本。
2. **全轨迹留痕**：每个 agent 每步写 `trace.jsonl`（输入 span、检索命中、中间推理摘要、输出、confidence）。
   任一归因结论都能回放到"哪条证据 + 哪个裁判"。
3. **接地约束**：agent 输出必须引用可定位的 `evidence_span`（turn 内字符区间 / 文件+段落）；无法引用 → `abstain` 进人工。
4. **人工金标准锚定**：V1/V2/V3 三个验证集给出 agent-vs-human 的 agreement / precision / Brier；
   论文主张"LLM 达到可用一致性"必须由这些数字支撑（Gilardi 2023 的报告范式）。
5. **集成量化不确定性**：K 次独立运行（变 seed）→ 用分歧度当置信；分歧样本转人工，既省人力又诚实报错。
6. **防泄漏审计**：静态 θ 不进预测特征；归因只用 $\le t$ 候选；国名遮蔽。附"关掉官方引用通道"的稳健性对照。

## 五、基线（先跑，D-MATI 必须超过才算成功）
- **B1 Jaccard**：提案与编辑 n-gram 重叠 → argmax 归因。
- **B2 嵌入相似**：SBERT 余弦 → argmax。
- **B3 单次 LLM 蕴含**：遮蔽国名的一次性 NLI 打分 → argmax（**无辩证、无 sink、无集成**——用来证明"辩证+回溯"的增量）。
- **B4 发言次数**：国家在该主题 `relevance=1` turn 数排名。
- **B5 提案采纳率**：国家提案被后续编辑匹配上的简单比值（无生命周期控制）。
- **B6 终稿—立场距离**：preference attainment 排名（Berge & Stiansen 型）。

D-MATI 若不能在 provenance precision / 终稿存活预测上稳定超过 B1–B3、B5，或时间负对照仍给同样名次 → 核心方法失败。
**特别地**：Provenance Court 必须显著超过 B3（单次 LLM），否则"多智能体"这层没有增量，应回退到单次 LLM + 校准。

## 六、识别边界（不因简化而放松）
用 **revealed textual influence**，非无条件因果。把 $b_i$ 当因果 bargaining power 需假设：控制提案内容、
条款机会、时点、联盟、会议 FE 后，无同时影响国家身份与采纳概率的未观测因素。至少控制：发言/提案机会、
提案质量、是否给可直接起草语言、初始 status-quo 距离、主题与条款 FE、会议阶段、联盟规模、
secretariat/Chair uptake、是否议题首倡者。GDP 等权力变量**不进影响力得分**，只在第二阶段解释 $b_i$ 差异。
**agent 引入的新识别威胁**：LLM 的训练先验可能偏向著名国家 → 用 §四的国名遮蔽 + 身份置换检验 + "关官方引用"稳健性对照防御。

## 七、验证与证伪
- **时间负对照**：用草案发布之后的发言匹配此前编辑，归因准确率应降到接近随机。
- **身份置换**：同主题同会议内随机置换国名，$b_i$ 排名应消失（也检验 agent 是否偷用国名先验）。
- **无实质内容负对照**：致谢/程序复述/泛目标不应预测条款进入。
- **预测检验**：只用 WP.230 及以前预测 WP.236/238 修改，报 top-k precision、Brier、log score。
- **sink 校准**：秘书处独立起草的已知案例应主要落到 sink。
- **agent 一致性**：报三个验证集的 human agreement；裁判间一致性（Fleiss' κ）；集成分歧率。
- **人工链条审计**：每个高影响国家抽 10–20 条 proposal→edit→final 链，法律专家盲审。
- **留一条款/留一会议 + 敏感性**：变 $\rho$、BT 重要度、召回阈值、K、"官方引用开关"，查名次稳定性。

**最小证伪实验**：前四版训练，只预测最后一次实质草案修订；D-MATI 不稳定超过"最近文本相似度"与"发言次数"，
或时间负对照给同样名次，或 Provenance Court 不超过单次 LLM（B3）→ 方法失败。

---

## 八、数据现实核对（为什么这套在你们语料上可行）
- ADV 共 6 版（WP.168→212→212/Add.1→230→236→238）。**早期三次转换是文档体例切换**（`mode_shift=true`，
  整块 added/deleted），**非实质修改**——Genealogy Agent + `is_substantive` 过滤掉，避免把排版当法律变化。
  真正实质编辑集中在 WP.230→236→238（`changed` 9、12 条），确定性 `ADV_clauses_clean` 约 57 条。
- 秘书处注释确含官方 provenance 线索（如 `reflects a suggestion … (A/CN.9/1160, para. 21)`），
  当前仅 1/57 被朴素正则命中——**正是 Citation-Follower agent 要系统化榨取的资产**（去读被引会议纪要）。
- Round1：7,512 turns，`relevance=1` 2,046 条，覆盖第 42–48 届、147 个发言方——Proposal Agent 的输入池。
- Round2：222 个"发言方×主题"BT 坐标，代码可直接复用于 §2.5 重要度成对与 §3.4 动态立场。
- **样本量诚实结论**：57 条 clean edits 不足以给几十国精确全局排名。主文聚焦 2–3 个高争议主题/条款
  （如 Art.4 会员资格、Art.8 融资的争议括号项），国家全局排名放探索性附录。

## 九、实施顺序
1. 确定性层：跑 `build_dataset.py`（切编辑、过滤体例、谱系存活、抽 `A/CN.9` 引用、拼 turn↔draft 时间链、稳定 ID）。
2. Proposal Agent（集成）跑全 `relevance=1`；人工在 V1（~300–500）核查 → 报 P/R/F1。
3. 跑基线 B1–B6，出对照表。
4. Provenance Court（Citation-Follower + Advocate + Null + K 裁判）跑存活编辑；人工在 V2（~150 对）校准 → 报 top-k/Brier。
5. Importance 成对 + Round2-BT 标度；人工在 V3（~100 对）核查。
6. 估 §3.2 分层 logit（`brms`/`PyMC` 一行）→ $b_i$ 后验；加 §3.3 等分信用 + bootstrap + 反事实删除。
7. 全套证伪与敏感性；不足则收敛到 2–3 主题，全局排名入附录。

## 十、创新性结论（scoop-check 复核）
**实质创新未变，仍 Level 3 — Medium Overlap**（最近对手 Lie 2023 *Treaty Influencers* 只做条约级文本演化、
无联盟归因、无 sink、无不确定性；Berge & Stiansen 2023 只做双边 preference attainment）。三个承重件不变：
**(1) 提案级动态 provenance；(2) secretariat/no-country sink；(3) 联盟反事实/信用分配**。缺一即退化为既有文本重用或 preference-attainment 研究。

**v3 新增的、可辩护的方法学增量**（相对 v1/v2 是"how"的升级，不改"what"的 delta）：
> 首次在**同一多边起草记录内**用**多智能体辩证 + 官方引用回溯**产出带校准与不确定性的**条款级 provenance**，
> 并把主观强度/存活判断替换为**成对比较标度 + 确定性谱系**，使影响力测量在**可复现、可追溯、低人工**的前提下成立。

一句话 delta（承 v2）：Unlike Lie 2023, which traces treaty-level text development without assigning actor credit,
D-MATI attributes each surviving clause edit to countries, coalitions, and a secretariat/no-country sink within one
multilateral drafting process — now via **reproducible multi-agent adjudication grounded in the UN's own documentary
citations**, a shrinkage logit for proposal-conversion power, and a symmetric equal-split (closed-form Shapley) credit
rule — yielding actor-level conversion power and marginal textual influence with posterior uncertainty, distinct from
preference attainment.

## 附：关键文献锚点
**LLM/agent 方法学（v3 新引，支撑"用 LLM/agent 有据可依"）**
- Gilardi, Alizadeh & Kubli. 2023. "ChatGPT Outperforms Crowd Workers for Text-Annotation Tasks." *PNAS* 120(30). https://doi.org/10.1073/pnas.2305016120
- Törnberg. 2023. "ChatGPT-4 Outperforms Experts and Crowd Workers in Annotating Political Twitter Messages." arXiv:2304.06588.
- Chae & Davidson. 2023/2024. "Large Language Models for Text Classification: From Zero-Shot to Instruction-Tuning." *Sociological Methods & Research* (SocArXiv sthwk).
- Ziems et al. 2024. "Can Large Language Models Transform Computational Social Science?" *Computational Linguistics* 50(1). https://doi.org/10.1162/coli_a_00502
- Du et al. 2023. "Improving Factuality and Reasoning in Language Models through Multiagent Debate." arXiv:2305.14325.
- Zheng et al. 2023. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena." *NeurIPS* 2023. arXiv:2306.05685.
- Wang et al. 2023. "Self-Consistency Improves Chain of Thought Reasoning in Language Models." *ICLR* 2023. arXiv:2203.11171.
- Thurstone. 1927. "A Law of Comparative Judgment." *Psychological Review* 34(4). / Bradley & Terry. 1952. *Biometrika* 39.
- Chiang & Lee. 2023. "Can Large Language Models Be an Alternative to Human Evaluations?" *ACL* 2023.

**领域锚点（承 v1/v2）**
- Lie. 2023. "Treaty Influencers: A Computational Analysis of the Development of International Investment Law." （最近对手）
- Berge & Stiansen. 2023. "Bureaucratic Capacity and Preference Attainment in International Economic Negotiations." *RIO*. https://doi.org/10.1007/s11558-022-09475-z
- Klüver. 2009. "Measuring Interest Group Influence Using Quantitative Text Analysis." *EU Politics*. https://doi.org/10.1177/1465116509346782
- Wilkerson, Smith & Stramp. 2015. "Tracing the Flow of Policy Ideas … Text Reuse." *AJPS*. https://doi.org/10.1111/ajps.12175
- Allee & Elsig. 2019. "Are the Contents of International Treaties Copied and Pasted?" *ISQ*. https://doi.org/10.1093/isq/sqz029
- Beach. 2004. "The Unseen Hand in Treaty Reform Negotiations … Council Secretariat." *JEPP*. https://doi.org/10.1080/rjpp13501760410001100279
- Schneider. 2005. "Capacity and Concessions." *Millennium*. / McKibben. 2020. *EJIR*. https://doi.org/10.1177/1354066120906875
- Kubinec. 2025. "Generalized Ideal Point Models for Noisy Dynamic Measures." OSF. https://doi.org/10.31219/osf.io/8j2bt_v3

---
*说明：LLM/agent 方法学条目中，`arXiv:` 编号与 SocArXiv/DOI 为模型知识补录，请在投稿前对最终引用做一次 DOI/编号核验（paper-search 的 arXiv/Semantic Scholar 连接器本次因 SSL/403 降级，仅 Crossref 可用）。领域锚点已在 v1/v2 检索中核验。*
