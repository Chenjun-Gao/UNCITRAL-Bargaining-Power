# D-MATI v3：Agent 标注 + 极简数学的多边谈判文本影响力测量

> 版本关系：v1（`memo/多边谈判文本影响力测量研究设计_2026-07-12.md`）建立了三个量的框架；
> v2（`D-MATI_v2_简化标注_加强数学`）把复杂度从标注搬到数学（OT + Sinkhorn + 多状态生存 + 蒙特卡洛 Shapley）。
> **用户对 v2 的核心不满**：(a) 数学太重、太难；(b) 标注仍含主观项——`legal_importance∈{1,2,3}`
> 标注员之间无法一致，`survives` 要人工对照终稿太麻烦；(c) LLM 用得不够、agent 完全没用。
>
> **v3 的一句话**：**把"判断"从人搬到 agent（可复现、带证据、带不确定性），把"数学"降到两件本科工具，
> 人只做二元核查。** 主观评分改为 agent 的成对比较（更可靠）；存活状态由确定性谱系脚本自动算；
> 归因由"多智能体辩证 + 引用回溯"产出带校准概率的分配，天然保留 secretariat / no-country sink。
>
> **本次修订（v3.1）的两处收敛**：(1) **归因改为"编辑锚定"的顺序**——先锁定每两版草案之间的每处实质
> 改动，把该阶段发言按"是否针对这处改动"归类，在相关发言里识别提案、再区分"最早提出者"与"支持者"，
> 最后才交 Provenance Court 判每处改动里各提案分到多少贡献（概率分布 + 标准差）。这样候选提案天然
> 挂在真实发生的编辑上，不必先做全局提案普查。(2) **只测"成功编辑里的实际贡献"**：删去原 §3.2 的
> 提案转化率模型（logistic + 国家随机效应），不再看失败提案、不再算转化比例；唯一主量升级为 §3.3 的
> **realized / marginal textual influence（以成功为条件的实际文本贡献量）**。下文每处删改都标注"为什么"。

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

> 边界纪律（贯穿全文）：**agent 只产出模型输入（发言归类、提案与提出者/支持者判定、provenance 概率、
> 联盟、重要度成对、存活），两个"量"仍由透明统计模型估计**。绝不让 agent 直接吐出"哪国影响力最大"的
> 名次——那会把结论变成模型意见而非可识别参数。国名在归因阶段一律遮蔽，防止强国先验写进匹配。

---

## 一、标注怎么再简化：人只做"二元核查"，agent 做"判断 + 结构"

### 三层责任划分（这是 v3 的骨架）

| 谁 | 做什么 | 产物 | 为什么它做最合适 |
|---|---|---|---|
| **确定性脚本**（无模型，可复现） | 切版本、对齐条款、算跨版存活、过滤体例改动、抽取秘书处脚注里的 `A/CN.9/xxx` 引用 | `edits_master`、`clause_genealogy`、`footnote_citations` | 纯规则，零主观，100% 复现 |
| **Agent 群**（固定 model+temp=0+seed，留全轨迹） | 发言按"是否针对某处改动"归类、提案抽取、目标条款接地、**最早提出者 vs 支持者判定**、联盟立场、provenance 辩证裁决、重要度成对比较 | 带 `evidence_span+confidence` 的结构化标签 | 判断类、需检索/多跳、易主观——正是模型强于人处 |
| **人工**（唯一任务：核查，二元） | 对 agent 输出的**分层抽样**做 `agree∈{0,1}`；对 agent `confidence=low / 裁判分歧` 的样本做仲裁 | 金标准验证集、agreement/precision 报告 | 二元核查一致性远高于让人产出结构；成本降一个数量级 |

**关键转变**：v2 让人**产出**（打 1/2/3、判存活、连链接、编联盟）；v3 让人**核查**（agent 说这条是提案、
朝放宽、来自 P，人只点"同意/不同意"）。核查是二元、单样本自足的判断，Kappa 天然高、可外包、可复用。

### 编辑锚定的主干流程（v3.1 的顺序，全文据此展开）

v3.1 不再"先普查所有提案、再看哪些落地"，而是**从编辑出发**，对每两版草案之间的每处实质改动 $e$ 依次做四步：

1. **归类发言**：把该阶段各国发言按"是否针对 $e$ 这处改动"分成相关 / 无关两类。判定只看"编辑前正在
   辩论的问题"（哪一条款、当前措辞是什么、想往哪个方向改），**对终稿最终措辞遮蔽**——这叫**措辞盲匹配**
   （wording-blind matching：不拿终稿的字面去反向套发言，避免"结果决定归因"的循环）。
2. **识别提案**：在"针对 $e$ 的相关发言"里抽出原子提案（要动哪条、什么操作、朝哪个方向）。候选集因此
   天然只含与这处改动有关的提案，不需要全局提案普查来凑分母。
3. **区分提出者 vs 支持者**：对每个提案判"谁最早提出、谁只是事后支持"。三条铁律见 §2.2。
4. **交 Provenance Court 判贡献**：把"$e$ + 其候选提案（含提出者/支持者标注）"交给辩证裁判，输出
   $e$ 的贡献如何分到各提案（概率分布 + 标准差），并保留"找不到国家来源→秘书处综合"的 sink。

> 为什么这样排序：先锚定"改了什么"，再问"谁促成的"，能把每一分文本信用都挂到一处**真实发生且留存**
> 的改动上；提案的意义完全由它对应的编辑来定义，避免测量漂到"提了但没落地"的失败提案上（那正是本次
> 删去 §3.2 的原因，见 §三）。

### 人工到底还标什么（全部是"看一眼点是/否"）

- **验证集 V1（发言归类 + 提案检测 + 提出者判定）**：分层抽 ~300–500 个 turn，人核查三件事——(a) 该发言
  是否被正确归为"针对某处改动 $e$"；(b) agent 的 `has_proposal / operation / target_article` 是否正确；
  (c) "最早提出者 vs 支持者"的判定是否与人工阅读一致。→ 报发言归类、proposal detector 与提出者判定的 P/R/F1。
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

### 2.2 编辑锚定：发言归类 → 提案抽取 → 提出者/支持者判定（替代全局提案普查）

这一步是 v3.1 的顺序落点。对 §2.1 产出的每处实质编辑 $e$，脚本先取"该阶段（$e$ 所在的相邻两版之间）
所有 `relevance=1` 的发言"作为待归类池，然后三个子步骤依次跑：

**(a) 归类发言（措辞盲匹配）**。Agent 判每条发言是否"针对 $e$"。检索键**只用编辑前正在辩论的问题**：
$e$ 命中的条款号、$e$ 之前的现状措辞、以及争论方向（放宽/收紧/新增/删除）；**对终稿最终措辞遮蔽**，
不允许拿 $e$ 改成的字面去反向套发言。这样匹配的是"当时在争什么"，而非"最后写成什么"，从根上切断
"用结果解释来源"的泄漏（配套的时间负对照见 §7）。→ 产出 `stmt_to_edit.csv`（发言 × 编辑 相关性 + span）。

**(b) 抽提案**。在"针对 $e$"的发言里抽原子提案，填 `operation / target_article / stance_sign(-1,0,+1，只标符号)`，
给 `proposal_span`。**集成**：跑 K=5 次（变 seed / 轻微改写 prompt）→ 多数票定标签、票型离散度 = `confidence`
（self-consistency，即"同一输入多次采样，答案越一致越可信"）。候选提案集 $C_e$ 因此天然只含与 $e$ 有关者。

**(c) 判提出者 vs 支持者**。对 $C_e$ 里每个提案，判"谁最早提出、谁只是事后支持"，三条铁律：

- **Citation-Follower 先跑**：源头可能是秘书处而非"最早开口的国家"。先解析官方引用（见 §2.3），
  它可以**推翻**"最早发言的国家即提出者"这一朴素假设。
- **同会期近同时按集合**：同一会期内、时间差落在容差窗口内的近同时提案，视为**共同提出者**（co-proposers），
  不强行分先后。
- **判定对结果盲**：判提出者时**遮蔽 $e$ 最终写成什么、遮蔽国名**，只看发言本身的时间与内容，避免
  "谁像赢家谁就是提出者"的循环。
→ 产出 `proposals.csv`（含 `proposer` / `supporters` / `co_proposer_set` + 证据）。人工只在 V1 上核查。

### 2.3 Provenance Court（核心创新：辩证 + 引用回溯，替代 OT）

对每条**存活的实质编辑** $e$，候选提案集 $C_e$ 已由 §2.2 (b) 的编辑锚定给出（同主题、同条款、时间早于 $e$、
且经措辞盲匹配判为"针对 $e$"，天然防未来信息泄漏），并已带 §2.2 (c) 的提出者/支持者标注。本步只做**贡献
裁决**：把 $e$ 与 $C_e$ 交给三类 agent（**国名全遮蔽**），判 $e$ 的文本信用如何分到各提案：
- **Citation-Follower（客观通道，优先）**：解析 e 所在秘书处注释里的 `A/CN.9/xxx, para.` 引用，
  **去读那届会议纪要**，抽取秘书处自己写明的来源方/来源提案。命中即为强证据（官方自述，非模型猜测）。
- **Advocate**：论证"e 实现了候选 p"，必须引 e 与 p 的对应 span。
- **Null-Advocate**：论证"e 是秘书处综合 / 整体共识 / 无可观测来源"（= sink），必须给理由。
- **Judge（K 个独立实例）**：读三方证据，输出 e 在 `{p1..pk, sink}` 上的**概率分布** `q_{pe}`（Σ=1）。
  K 个裁判的均值 = 归因概率，方差 = 不确定性；分歧大 → 进 V2 人工仲裁。
→ 产出 `provenance.csv`（每条 e 的软分配 + sink 概率 + 证据链）。**质量守恒与 sink 由"概率分布 + 显式 sink 类"
直接保证，无需 Sinkhorn。**

### 2.4 Importance via Pairwise（替代主观 1/2/3）

不打绝对分。**Judge agent 做成对比较**：随机配对存活编辑，问"哪条更改变权利义务/准入门槛？"，
每对多裁判投票。把成对胜负喂 **Bradley–Terry（复用 Round2 代码）** → 每条编辑一个连续 `importance_e`（带 SE）。
人工只在 V3 上核查成对方向。**这是把主观评分变客观标度的关键一招，且不引入新数学。**

---

## 三、数学层：只用两件本科工具（对比 v2 砍掉 OT / Sinkhorn / 多状态生存 / 采样 Shapley；v3.1 再砍掉转化率 logit）

> **本节结构变动说明**：v3 原有 §3.1 归因概率、§3.2 提案转化率 logit、§3.3 边际文本影响、§3.4 动态立场。
> v3.1 **删去 §3.2**——因为它是为"提案转化率/每次尝试效力"服务的，需要把"失败提案"当分母，要做全局提案
> 普查、议题级转化锚定、提案侧 sink。而用户只要看**每个 entity 在成功编辑里的实际文本贡献量**，不看失败
> 提案、不算比例。删掉后，§3.3 的 realized / marginal textual influence 升为**唯一主量**，两件本科工具即
> **(1) 归因概率（agent 给分布）+ (2) 等分/闭式 Shapley 信用规则**；动态立场（原 §3.4）保留但只服务
> preference attainment 与措辞盲匹配的方向信号。下面逐条说明"为什么删、删完谁承重"。

记号：编辑 $e$，候选提案 $p$，国家 $i$，会议届次 $t$。所有解释变量只用 $\le t$ 的信息。

### 3.1 归因概率（agent 直接给，不解 OT）

候选集 $C_e$ 来自 §2.2 的编辑锚定（措辞盲匹配 + 时间早于 $e$ + 提出者/支持者标注），不再是全局提案池。
Provenance Court 的 K 个裁判对编辑 $e$ 给出分布，取均值：
$$q_{pe}=\frac{1}{K}\sum_{k=1}^{K} \text{Judge}_k(p\mid e),\qquad \sum_{p\in C_e} q_{pe} + q_{\text{sink},e}=1 .$$
$q_{\text{sink},e}$ 就是"秘书处综合/无国家来源"的质量，**天然保留 no-country 类别**。
不确定性 $\;\widehat{\text{sd}}(q_{pe})=\text{sd}_k \text{Judge}_k(p\mid e)$。若 Citation-Follower 命中官方来源，
则对该 $p$ 施加强先验（在 prompt 中作为证据给裁判，并在敏感性分析里报告"去掉官方引用后"的名次变化）。

### 3.2〔已删除〕提案转化率模型——为什么去掉

> v3 此处曾放一个"提案是否进入后续草案并留到 WP.238"的分层 logistic（国家随机效应 $b_i$ 作转化权）。
> **v3.1 整节删除**，理由：该模型的估计对象是"**每次尝试的转化率/效力**"，必须把**失败提案**当分母，
> 才谈得上"提了 N 次、成功 M 次"的比例。这要求全局提案普查、议题级转化锚定、提案侧 sink——一整套只为
> "凑失败分母"服务的机制。而本研究只问**每个 entity 在已成功编辑里实际贡献了多少文本**，不看失败提案、
> 不算转化比例，上述机制全部冗余，一并去掉。
>
> **估计量的诚实降级**：删掉转化率后，主量不能再叫"控制机会后的 bargaining power"（那需要净掉发言/提案
> 机会与资源，靠的正是被删的第二阶段与随机效应）。它退成**以成功为条件的实际文本贡献量**，正式命名为
> **realized / marginal textual influence**（见 §3.3），即"在最终成功的编辑上，各 entity 分到的文本信用总量"。
> 这是一个描述性、可复现的量，不再声称净掉了机会/资源混淆。
>
> **随之删除的还有**：原打算"从结果反推 proposer 溢价"的角色分离随机效应路径（它依赖 $b_i$ 的第二阶段）。
> 提出者相对支持者的溢价 $\rho$ 因此**不能再从数据估**，只能**预注册**或用**编辑重要度上的精确 Shapley /
> pivotality** 设定，并做敏感性分析（见 §3.3）。GDP 等权力变量原用于"解释 $b_i$ 差异"，现因第二阶段删除
> 而**不再进入任何影响力得分**；若要看它与 MI 的关系，只能作事后描述性相关，不进测量管线。

### 3.3 边际文本影响（唯一主量）：等分 / 闭式 Shapley 信用规则（替代蒙特卡洛 Shapley）

这是删去 §3.2 后的**唯一主量**。对每条存活编辑 $e$，其可分配总量 $=\text{importance}_e$（§2.5 的 BT 标度）。
按归因概率 $q_{pe}$ 落到提案 $p$，再在"提案者 + 支持联盟"间分配。用**预注册的等分规则**：提案者得固定
份额 $\rho$，其余 $(1-\rho)$ 在支持联盟 $S_p$ 成员间**等分**：
$$\phi_{ie}=\sum_{p}\ q_{pe}\Big[\rho\,\mathbb 1\{i=\text{proposer}(p)\}+\frac{1-\rho}{|S_p|}\,\mathbb 1\{i\in S_p\}\Big],\qquad
\text{MI}_i=\sum_e \phi_{ie}.$$
> **为什么这等价于 Shapley 又更简单**：Shapley 值是合作博弈里"按边际贡献公平分配总收益"的标准解；
> 当联盟成员对结局的贡献可交换（对称）且价值可加时，它就退化为**等分**。所以等分规则是 Shapley 在
> "对称+可加"假设下的闭式解，无需指数级枚举或蒙特卡洛采样——把假设写明即可。
>
> **$\rho$（提出者溢价）怎么定——注意不能再从结果估**：删掉 §3.2 后，"从数据反推提出者比支持者强多少"
> 的路径没有了。$\rho$ 只能**外生设定**，两条合规来源：(i) **预注册**一个固定值（如 $\rho=0.7$）；
> (ii) 在**编辑重要度**这个"联盟价值"上做**小联盟精确 Shapley / pivotality**——因为每处编辑的相关联盟只有
> 少数几方，可**穷举所有加入次序**精确算，不必采样。**pivotality（关键性）**指某成员在所有次序里"由它加入
> 才使联盟从'促不成该编辑'翻转为'促成'"的频率；提出者若确为关键，Shapley 会自动给它更高份额。
> **敏感性分析（预注册）**：报 $\rho=0.5$、$\rho=0.7$、精确 Shapley 三个版本，检查**提案/国家排名的稳定性**；
> 排名随 $\rho$ 大幅翻转即说明结论对该假设敏感，需在正文标注。
> 不确定性：对 $(q_{pe},\,\text{importance}_e,\,S_p)$ 做 bootstrap（重采样估区间），报 $\text{MI}_i$ 的区间。

### 3.4 动态立场（防泄漏，替代静态 BT 直接入模）
仍复用 Round2 的成对数据，但让立场随会议游走：
$$\Pr(y_{ijkt}=1)=\text{logistic}(\theta_{ikt}-\theta_{jkt}+\text{order}+\text{annotator}),\quad
\theta_{ikt}\sim\mathcal N(\theta_{i,k,t-1},\sigma_k^2).$$
解释 $t\!\to\!t\!+\!1$ 版只用 $\theta_{i,k,\le t}$。**删掉 §3.2 后它的用途收窄但仍必要**：动态 $\theta$
一是为 §3.5 的 **preference attainment**（终稿离谁的立场近）提供坐标，二是给 §2.2 措辞盲匹配提供"当时
争论方向"的信号（判发言是否针对某处改动时用到"放宽/收紧"方向）。它**不再**作为任何转化率模型的特征。

### 3.5 两个量、两种报告（删 §3.2 后由三量收敛为两量）

| 量 | 定义 | 来源 |
|---|---|---|
| 偏好实现 preference attainment | 终稿离谁 $\theta$ 近 | §3.4 + 终稿位置；**结果接近度，非因果** |
| 总边际文本影响 $\text{MI}_i$（主量） | $\sum_e\phi_{ie}$ | §3.3，成功编辑里实际产生的文本贡献总量 |

原"提案转化权 $b_i$"一行随 §3.2 删除（不再测转化率）。两量一致时"该国影响大"最可信；主报 $\text{MI}_i$
及其 bootstrap 区间，辅以反事实删除
$\Delta_i=D(\mathbb E[\text{Final}\mid\text{all}],\ \mathbb E[\text{Final}\mid\text{remove }i])$
作为稳健性佐证。

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
6. **防泄漏审计（删 §3.2 后更承重）**：措辞盲匹配（检索键落在"编辑前正在辩论的问题"，对终稿最终措辞
   遮蔽）；归因候选只用 $\le t$、且经 §2.2 判为"针对该编辑"者；提出者判定对结果盲、Citation-Follower 先跑、
   同会期近同时按集合视为共同提出者；国名遮蔽。附"关掉官方引用通道"的稳健性对照。§7 时间负对照当泄漏探测器。

## 五、基线（先跑，D-MATI 必须超过才算成功）
- **B1 Jaccard**：提案与编辑 n-gram 重叠 → argmax 归因。
- **B2 嵌入相似**：SBERT 余弦 → argmax。
- **B3 单次 LLM 蕴含**：遮蔽国名的一次性 NLI 打分 → argmax（**无辩证、无 sink、无集成**——用来证明"辩证+回溯"的增量）。
- **B4 发言次数**：国家在该主题 `relevance=1` turn 数排名。
- **B5 原始采纳计数**：国家提案被后续编辑匹配上的简单计数（不区分提出者/支持者、不加重要度权、无 sink）。
  作为 $\text{MI}_i$ 的朴素对照——若 MI 排名与它无异，说明编辑锚定 + 信用分配没带来增量。
- **B6 终稿—立场距离**：preference attainment 排名（Berge & Stiansen 型）。

D-MATI 若不能在 provenance precision / $\text{MI}_i$ 排名稳定性上稳定超过 B1–B3、B5，或时间负对照仍给同样
名次 → 核心方法失败。（原"终稿存活预测"指标随 §3.2 转化率模型一并移除，不再作为成败判据。）
**特别地**：Provenance Court 必须显著超过 B3（单次 LLM），否则"多智能体"这层没有增量，应回退到单次 LLM + 校准。

## 六、识别边界（删 §3.2 后更诚实，不放松）
主量是 **realized / marginal textual influence**——一个**以成功编辑为条件的实际文本贡献量**，**不是**因果
bargaining power。删掉 §3.2 的转化率第二阶段后，我们**不再声称净掉了机会/资源混淆**：$\text{MI}_i$ 高既可能
因为该国真有影响力，也可能因为它发言/提案机会多、议题恰好活跃。因此 $\text{MI}_i$ 应读作"在最终留存的文本里，
按可审计的 provenance + 预注册信用规则，各 entity 分到多少字面贡献"，而非"控制一切后的议价能力"。
为把这个描述量做扎实，仍尽量控制并报告：发言/提案机会、提案质量、是否给可直接起草语言、初始 status-quo
距离、主题与条款 FE、会议阶段、联盟规模、secretariat/Chair uptake、是否议题首倡者。**GDP 等权力变量不进
影响力得分**；因第二阶段已删，它们只能作 $\text{MI}_i$ 的**事后描述性相关**，不进测量管线、不用于"解释得分"。
**agent 引入的新识别威胁**：LLM 的训练先验可能偏向著名国家 → 用 §四的国名遮蔽 + 身份置换检验 + "关官方引用"稳健性对照防御。

## 七、验证与证伪
- **时间负对照**：用草案发布之后的发言匹配此前编辑，归因准确率应降到接近随机。
- **身份置换**：同主题同会议内随机置换国名，$\text{MI}_i$ 排名应消失（也检验 agent 是否偷用国名先验）。
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
2. **编辑锚定预处理**（§2.2）：对每处存活编辑，归类该阶段发言 → 抽提案 → 判提出者/支持者（集成）；
   人工在 V1（~300–500）核查发言归类、提案检测、提出者判定 → 报 P/R/F1。
3. 跑基线 B1–B6，出对照表。
4. Provenance Court（Citation-Follower + Advocate + Null + K 裁判）在每处编辑的候选集 $C_e$ 上裁决；
   人工在 V2（~150 对）校准 → 报 top-k/Brier。
5. Importance 成对 + Round2-BT 标度；人工在 V3（~100 对）核查。
6. 算 §3.3 信用分配 $\text{MI}_i$：$\rho=0.5/0.7$/精确 Shapley 三版 + bootstrap 区间 + 反事实删除。
7. 全套证伪与敏感性；不足则收敛到 2–3 主题，全局排名入附录。

## 十、创新性结论（scoop-check 复核）
**实质创新未变，仍 Level 3 — Medium Overlap**（最近对手 Lie 2023 *Treaty Influencers* 只做条约级文本演化、
无联盟归因、无 sink、无不确定性；Berge & Stiansen 2023 只做双边 preference attainment）。三个承重件不变：
**(1) 提案级动态 provenance；(2) secretariat/no-country sink；(3) 联盟反事实/信用分配**。缺一即退化为既有文本重用或 preference-attainment 研究。

**v3 新增的、可辩护的方法学增量**（相对 v1/v2 是"how"的升级，不改"what"的 delta）：
> 首次在**同一多边起草记录内**用**多智能体辩证 + 官方引用回溯**产出带校准与不确定性的**条款级 provenance**，
> 并把主观强度/存活判断替换为**成对比较标度 + 确定性谱系**，使影响力测量在**可复现、可追溯、低人工**的前提下成立。

一句话 delta（承 v2，v3.1 收敛）：Unlike Lie 2023, which traces treaty-level text development without assigning
actor credit, D-MATI anchors on each surviving clause edit, classifies the debate turns that target it, distinguishes
each proposal's originator from mere supporters, and attributes the edit to countries, coalitions, and a
secretariat/no-country sink within one multilateral drafting process — via **reproducible multi-agent adjudication
grounded in the UN's own documentary citations** and a symmetric equal-split (closed-form Shapley) credit rule —
yielding actor-level **realized/marginal textual influence conditional on adopted edits**, with posterior uncertainty,
distinct from preference attainment. (Proposal-conversion power / success-rate modeling is deliberately dropped: only
credit within successful edits is measured.)

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
