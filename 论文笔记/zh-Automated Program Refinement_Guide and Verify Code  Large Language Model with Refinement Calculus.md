自动化程序细化：使用细化演算引导和验证代码大语言模型

# 0 Abstract

近年来，代码驱动的大语言模型（LLMs）迅速崛起，借助如 Copilot 这类门槛低的工具，正在重塑软件工程领域，它们可以轻松生成代码。然而，LLM 所生成的代码并不具备正确性保障，容易受到“幻觉”问题的影响，其输出充满风险。此外，从规范到代码的端到端过程是一个不透明、不可控的“黑盒”，这使得用户难以理解并信任生成的代码。解决这些挑战既必要又关键。

相比之下，程序细化（Program Refinement）能够将高级规范语句转换为可执行代码，同时保持其正确性。传统的程序细化工具主要面向形式化方法专家，缺乏自动化能力和可扩展性。

为此，我们将程序细化引入 LLM 的代码生成过程，旨在指导 LLM 并验证其生成的代码，同时将程序细化转化为一个更易用、灵活的框架。为了实现这一愿景，我们提出了 Refine4LLM，一种旨在实现以下目标的方法：

1. 形式化地细化规范；
    
2. 自动化地使用细化演算对 LLM 进行提示和引导；
    
3. 与 LLM 交互生成代码；
    
4. 验证生成的代码是否满足约束条件，从而确保其正确性；
    
5. 学习并构建更高级的细化定律，以扩展细化演算体系。
    

我们将 Refine4LLM 与当前在程序细化和 LLM 领域的最新基线方法进行了对比评估。实验结果表明，Refine4LLM 能够更高效地生成更健壮的代码，并减少细化与验证所需的时间。


# 1 Introduction

## 1.1 Challenges in LLM-based Code Generation
近年来，大语言模型（LLM）在数学、推理和编程方面取得了快速进展 [49, 66]。像 GPT-4 [44] 和 Copilot [31] 这样的工业产品极大地帮助了程序员完成与编码相关的任务，并在编程竞赛中表现超过了第 50 百分位 [45]。**然而，它们面临的一个关键挑战是“幻觉”问题，即模型生成看似合理但实际上不正确的信息。** 此外，一些用户研究 [25, 57] 表明，**程序员发现很难信任和调试由 LLM 生成的代码，因为生成过程是不透明且不可控的。** 更糟糕的是，研究人员发现 ChatGPT 对编程相关问题的回答中有一半以上包含错误信息 [36]，而最近甚至有数学证明表明 LLM 的幻觉是不可避免的 [60]。我们以经典的细化示例——平方根算法（见图 1）为例，展示了最新 LLM 模型（OpenAI 的 o1-preview [43]、GPT-4 [44] 和 GitHub Copilot [31]）在输入同一个提示语时所生成的代码片段：
```
Find the square root of N within the error bound e.
在误差范围 e 内求出 N 的平方根。
```

LLM 可以生成几乎正确的代码，但这些程序仍然存在一些错误。基于 LLM 的 GitHub Copilot 和 OpenAI GPT-4 所生成的代码会陷入无限循环，而 OpenAI 的 o1-preview 模型在某些情况下则给出了错误的结果。
具体来说，Copilot 生成的程序在输入 n < 1 时是错误的。从数学上讲，变量 `high` 作为N 的平方根的上界，其选择应大于 N+1/4，因为对于所有 N > 0，都有 (N+1/4)^2≥N。
GPT-4 生成的代码在某些情况（如求 sqrt(5)）下会失败，因为变量 `x` 会趋近于某个固定点但由于浮点精度误差无法终止循环。
OpenAI o1-preview 模型生成的代码在 (e/4)^2 时有时会给出错误结果，因为变量 `x` 被初始化为负数。

![[Pasted image 20250422145025.png]]


用于程序生成的可靠大语言模型（LLMs）仍然是一个尚未解决的挑战。目前的策略主要集中在输入引导和输出验证两个方面。引导 LLM 的做法是通过向其提供与任务相关的信息来引导其朝着更简洁、在其能力范围内的解法前进。近期的研究通常采用非正式的启发式方法，例如“思维链”（Chain-of-Thought）策略，引导 LLM 的推理过程 [59]。另一方面，从白盒角度彻底验证深度学习模型仍然仅限于小规模的量化神经网络，距离验证 LLM 尚有很大差距 [34]。目前最新的 LLM 验证方法依赖多个 LLM，通过多数投票或自然语言中的共识来评估输出 [1, 40, 64]。然而，已有研究 [60] 在数学上证明，单靠改变提示词无法消除幻觉现象。他们还指出，多个 LLM 的集成本质上仍然是一个 LLM，无法根本解决幻觉问题。

相比之下，我们提出的 Refine4LLM 方法融合了 LLM 与符号人工智能系统 CoqHammer [20]，通过程序细化与验证来在约束条件下引导 LLM，并验证其生成代码的正确性。我们的方法模拟人类使用计算器或代码解释器等工具来解决超出自身即时能力范围的问题的方式。直观上，我们将 LLM 视为“约束求解器”，其强大的扩展性和丰富的背景知识为程序细化的自动化带来了新契机。我们的程序细化方法能够“断言”有助于调试的约束条件，并“验证”这些约束，以确立生成代码的正确性。这种方法在将 LLM 应用于程序生成方面迈出了重要一步，不仅降低了错误率，更实现了可靠且可验证的代码生成。

## 1.2 Challenges in Traditional Program Refinement

程序精化演算（refinement calculus）[5, 14, 41, 52, 56] 是对**逐步精化式程序构造方法**的一种形式化表达。程序精化的过程是：首先通过一个**不可执行的规范**来描述程序应具备的行为，然后通过一系列**保持正确性的变换步骤**，将其转化为一个可执行程序。

然而，这种基于程序精化演算的转化过程**主要依赖人工进行**，既耗时又容易出错。手动编写代码的需求使得程序精化过程**劳动强度大，难以自动化**。

因此，将**大语言模型（LLMs）在代码生成方面的能力**引入精化流程，是一种**合理的发展方向**。

总体而言，我们将这一方法实现为一个自动化工具，名为 **Refine4LLM**，它将**形式化的程序精化演算**与**非形式的大语言模型**相结合，支持从规范出发，逐步生成经过验证的代码。我们的做法利用了**LLMs 的创造力**和**形式化方法的严格性证明**，从而提供了一个可靠的代码生成器。

该方法与 LLM、本体自动定理证明器（ATP）以及传统验证工具**互为补充**。据我们所知，**Refine4LLM 是第一个将 LLM 与程序精化技术结合的框架**。

本文的贡献总结如下：

(1) 提出一个名为 **Refine4LLM** 的框架，用于实现由大语言模型辅助的自动化程序精化。该框架包括一个**形式化规范语言** Lspec、一个与我们精化演算相关联的**编程语言** Lpl，以及一套用于**验证生成代码**的验证策略。
(2) 提出一种**精化定律学习策略**，用于推导出更高级的精化规则，以此**减少程序精化过程的深度**。
(3) 设计一种**自顶向下拆分与自底向上精化算法**，用于构建程序精化库，从而**降低规范的复杂度**。

## 1.3 Outline
第 2 节介绍了程序精化的背景知识。第 3 节通过一个平方根算法的示例说明了研究动机。第 4 节展示了工具 Refine4LLM 的整体概览，随后在第 5 节定义了形式化的规范语言和程序语言。Refine4LLM 采用了一种学习策略，用于构建新的精化定律，以减少精化过程的深度，这部分内容详见第 6 节。

第 7 节介绍了一个形式化系统，它可以基于精化定律自动地、形式化地对规范进行精化，并生成相应的证明义务。Refine4LLM 使用自动定理证明器（ATP）来验证精化定律的正确性。

第 8 节介绍了非形式系统，包括一个**自顶向下**的算法用于拆分高层规范，以及一个**自底向上**的算法结合大语言模型（LLMs）对子规范进行精化。

第 9 节展示了我们在经典程序精化基准任务 [37] 以及大语言模型基准集 Humaneval 和 EvalPlus 数据集 [38] 上对 Refine4LLM 的评估结果。


# 2 Preliminaries
## 2.1 Notation[41] 

我们遵循 Morgan 关于程序精化的著作 [41] 中的符号体系。

**规范（Specification）**：  
本研究考虑使用一阶逻辑（FOL）定义的形式化规范。一个规范由变量、前置条件和后置条件组成，形式如下：  
  **variables : [precondition, postcondition]**  
其中，variables 是程序中使用的变量列表；precondition（前置条件）描述程序的初始状态，postcondition（后置条件）描述程序的最终状态。

**程序精化（Program Refinement）**：  
形式上，精化关系通过相关程序的**最弱前置条件（weakest precondition）** 来定义 [13]。  
对于程序 S 和后置条件 P，表达式 wp(S,P)表示使得程序 S 能够终止并满足 P 的最弱前置条件。

若程序 S_0 被程序 S_1 精化，记作 **S0⊑S1**，当且仅当对所有 P 都成立：  
  **∀P,wp(S_0,P)→wp(S_1,P)**  (1)

这意味着程序 S1 保留了程序 S0 的**完全正确性**（即程序能够终止并满足规范）。

程序精化可以通过一系列线性步骤进行如下表示：  
  **S0⊑S1⊑S2⊑S3⋯⊑Sn**  (2)

这表明利用精化关系的传递性，可以从 S_0 推导出最终精化程序 S_n，即 S0⊑Sn。

## 2.2 The Basic Refinement Calculus

精化演算是基于文献 [24] 中提出的程序的最弱前置条件语义。

精化过程：是一个连续应用**精化法则（refinement laws）** 的过程，它将形式规范逐步转化为混合了规范与程序代码的形式（即**混合程序**），最终完全转化为程序代码，如下所示：
𝑠𝑝𝑒𝑐𝑖𝑓𝑖𝑐𝑎𝑡𝑖𝑜𝑛 ⊑ 𝑚𝑖𝑥𝑒𝑑 𝑝𝑟𝑜𝑔𝑟𝑎𝑚 ⊑ · · · ⊑ 𝑚𝑖𝑥𝑒𝑑 𝑝𝑟𝑜𝑔𝑟𝑎𝑚′ ⊑ 𝑝𝑟𝑜𝑔𝑟𝑎𝑚.

Morgan 所著的书 [41] 中总结了**核心的精化演算内容**，如下所示：

**引理 2.1（加强后置条件法则 Strengthen Postcondition Law）**
**英文原文：**  
Let 𝑝𝑟𝑒 (precondition) and 𝑝𝑜𝑠𝑡, 𝑝𝑜𝑠𝑡′ (postconditions) be any FOL formula,  
if 𝑝𝑜𝑠𝑡′ ⇛ 𝑝𝑜𝑠𝑡, then 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡′].
**中文翻译：**  
设 𝑝𝑟𝑒 是前置条件，𝑝𝑜𝑠𝑡 和 𝑝𝑜𝑠𝑡′ 是后置条件，三者均为一阶逻辑公式（FOL）。  
若 𝑝𝑜𝑠𝑡′ 推出 𝑝𝑜𝑠𝑡（记作 𝑝𝑜𝑠𝑡′ ⇛ 𝑝𝑜𝑠𝑡），  
则有：𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡′]。
**解释：**  
这条法则表示，如果你把程序的后置条件变得**更强**（更具体、更严格），那么程序就被精化了。因为更强的后置条件意味着程序必须满足更多约束，从而语义更精确、更安全。


**引理 2.2（削弱前置条件法则 Weaken Precondition Law）**
**英文原文：**  
Let 𝑝𝑟𝑒, 𝑝𝑟𝑒′ (preconditions) and 𝑝𝑜𝑠𝑡 (postcondition) be any FOL formula,  
if 𝑝𝑟𝑒 ⇛ 𝑝𝑟𝑒′, then 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒′, 𝑝𝑜𝑠𝑡].
**中文翻译：**  
设 𝑝𝑟𝑒 和 𝑝𝑟𝑒′ 是前置条件，𝑝𝑜𝑠𝑡 是后置条件，三者均为一阶逻辑公式。  
若 𝑝𝑟𝑒 推出 𝑝𝑟𝑒′（记作 𝑝𝑟𝑒 ⇛ 𝑝𝑟𝑒′），  
则有：𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒′, 𝑝𝑜𝑠𝑡]。
**解释：**  
如果你把程序的前置条件变得更弱（更宽松），那么这个程序就能接受更多的输入，因此是对原程序的一个精化。更弱的前置条件使得程序更“通用”。

**引理 2.3（跳过法则 Skip Law）**
**英文原文：**  
If 𝑝𝑟𝑒 ⇛ 𝑝𝑜𝑠𝑡, then 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ skip.
**中文翻译：**  
若 𝑝𝑟𝑒 推出 𝑝𝑜𝑠𝑡（即若前置条件成立，则后置条件自然成立），  
则有：𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ skip。
**解释：**  
这个法则说明：如果某个规范的后置条件已经被前置条件“自动保证”，那么你可以将其精化为一个什么也不做的操作——`skip`。因为无需实际执行代码，就能满足该规范。

**引理 2.4（顺序组合法则，Seq）**
**英文原文：**  
Let 𝑚𝑖𝑑 be one formula except for 𝑝𝑟𝑒 or 𝑝𝑜𝑠𝑡 and not contain initial variables.  
𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒, 𝑚𝑖𝑑]; 𝑥 : [𝑚𝑖𝑑, 𝑝𝑜𝑠𝑡].
**中文翻译：**  
设 𝑚𝑖𝑑 是一个逻辑公式，不等于前置条件（𝑝𝑟𝑒）或后置条件（𝑝𝑜𝑠𝑡），且不包含初始变量。  
则有：  
𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑥 : [𝑝𝑟𝑒, 𝑚𝑖𝑑]; 𝑥 : [𝑚𝑖𝑑, 𝑝𝑜𝑠𝑡]。
**解释：**  
这是顺序组合规则，意思是可以把一个整体的规范 `[前置条件, 后置条件]` 拆成两个顺序执行的部分，其中 `mid` 是一个中间状态。如果从 `pre` 到 `mid` 是正确的，从 `mid` 到 `post` 也是正确的，那么整体上从 `pre` 到 `post` 的程序是被精化的。就像在写程序时把一个大任务拆成两个子任务。

 **引理 2.5（赋值法则 Assignment Law）**
**英文原文：**  
Let 𝐸 be any expression, 𝑝𝑜𝑠𝑡⟨𝑥 := 𝐸⟩ assigns every 𝑥 in 𝑝𝑜𝑠𝑡 with 𝐸.  
If 𝑝𝑟𝑒 ⇛ 𝑝𝑜𝑠𝑡⟨𝑥 := 𝐸⟩, then 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ x = E.
**中文翻译：**  
设 𝐸 是任意一个表达式，𝑝𝑜𝑠𝑡⟨𝑥 := 𝐸⟩ 表示将后置条件中所有的 𝑥 替换为 𝐸。  
如果 𝑝𝑟𝑒 推出 𝑝𝑜𝑠𝑡⟨𝑥 := 𝐸⟩，  
则有：𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ x = E。
**解释：**  
这条法则描述了赋值语句的精化条件：如果在执行 `x = E` 之前的状态（即 `pre`）可以推出执行完赋值后目标 `post` 的“回代形式”，那么就可以把这个规范精化为赋值语句 `x = E`。

**引理 2.6（选择法则 Alternation Law）**
**英文原文：**  
Let 𝐺𝐺 be the disjunctive normal form of the guards 𝐺0, 𝐺1, ..., 𝐺𝑛,  
if 𝑝𝑟𝑒 ⇛ 𝐺𝐺, then  
𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ if ⨆ 𝑖 (𝐺𝑖 then 𝑥 : [𝐺𝑖 ∧ 𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡])  
where “if ⨆ 𝑖 𝐺𝑖 then” means if G0 then ... else if Gi then ....
**中文翻译：**  
设 𝐺𝐺 是一组守卫条件（𝐺0, 𝐺1, ..., 𝐺𝑛）的析取范式，  
若 𝑝𝑟𝑒 推出 𝐺𝐺，  
则有：  
𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ if ⨆ 𝑖 (𝐺𝑖 then 𝑥 : [𝐺𝑖 ∧ 𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡])，  
其中 `if ⨆ 𝑖 G𝑖 then` 表示多分支条件语句，即：`if G0 then ... else if G1 then ...`。
**解释：**  
这个法则表明，如果前置条件能推出所有分支守卫条件的联合，那么整个程序可以被精化成一个“按守卫条件分类执行的选择结构”，即 if-else 结构，并且每个分支都带有一个局部前置条件 `G𝑖 ∧ pre`。这就是**分支结构的精化**规则。


**引理 2.7（迭代法则 Iteration Law）**
**英文原文：**  
Let 𝐼𝑛𝑣, the invariant, be any formula; let 𝑉, the variant, be any integer-valued expression.  
Let 𝐺𝐺 be the disjunctive normal form of the guards 𝐺0, 𝐺1, ..., 𝐺𝑛.  
Then  
**𝑥 : [𝐼𝑛𝑣, 𝐼𝑛𝑣 ∧ ¬𝐺𝐺] ⊑ while ⨆𝑖 (𝐺𝑖 do 𝑥 : [𝐼𝑛𝑣 ∧ 𝐺𝑖, 𝐼𝑛𝑣 ∧ (0 ≤ 𝑉 < 𝑉0)])**  
where 𝑉0 is the initial value of 𝑉, and  
`while ⨆𝑖 𝐺𝑖 do` 表示：`while G0 do ... else if G1 do ... else if Gn do`。
**中文翻译：**  
设 𝐼𝑛𝑣（不变式）为任意公式，𝑉（变式）为任意整数值表达式，  
设 𝐺𝐺 是守卫条件 𝐺0, 𝐺1, ..., 𝐺𝑛 的析取范式。  
则有：  
**𝑥 : [𝐼𝑛𝑣, 𝐼𝑛𝑣 ∧ ¬𝐺𝐺] ⊑ while ⨆𝑖 (𝐺𝑖 do 𝑥 : [𝐼𝑛𝑣 ∧ 𝐺𝑖, 𝐼𝑛𝑣 ∧ (0 ≤ 𝑉 < 𝑉0)])**  
其中 𝑉0 是 𝑉 的初始值，  
`while ⨆𝑖 𝐺𝑖 do` 表示“如果 G0 则执行... 否则如果 G1 执行... 直到 Gn”。
**解释：**  
这是**构造 while 循环的精化规则**。如果你能证明在某个不变式下，每个守卫条件下的体执行后仍维持不变式，且变式的值不断减少并保持非负（即满足终止性），那么这个整体行为可以被精化为一个 `while` 循环。


**引理 2.8（扩展法则 Expand Law）**
**英文原文：**  
Let x be the origin variant and y be another variant and 𝑦0 be the initial value of y,  
then  
**𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] = 𝑥, 𝑦 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡 ∧ 𝑦 = 𝑦0]**
**中文翻译：**  
设 `x` 是原始变量，`y` 是新增变量，`y₀` 是其初始值，  
则有：  
**𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] 等价于 𝑥, 𝑦 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡 ∧ 𝑦 = 𝑦₀]**
**解释：**  
这是一个扩展规则，允许你在程序中引入一个新变量 `y`，只要你确保它的值等于初始值 `y₀` 且不影响其他行为，就不会改变程序语义。常用于中间变量、辅助变量的引入。

**引理 2.9（断言法则 Assertion Law）**
**英文原文：**  
Let E be a boolean condition for the variable x, then  
**𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] = assert E; 𝑥 : [𝑝𝑟𝑒 ∧ E, 𝑝𝑜𝑠𝑡]**
**中文翻译：**  
设 E 是关于变量 x 的布尔条件，  
则有：  
**𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] 等价于 assert E; 𝑥 : [𝑝𝑟𝑒 ∧ E, 𝑝𝑜𝑠𝑡]**
**解释：**  
这是对 `assert`（断言）的精化建模。如果你在某个点添加 `assert E`，那么就相当于要求前置条件额外满足 E。也就是说，程序会在执行时“检查” E 是否成立。

**定义 2.10（过程 Procedure）**
**英文原文：**  
A procedure is declared by a name, some parameters, and a program.  
**Definition 2.10 (Procedure):**  
**𝑝𝑟𝑜𝑐𝑒𝑑𝑢𝑟𝑒 ⟨Name⟩ (⟨Variable⟩ : ⟨Type⟩) ≜ ⟨Prog⟩**
**中文翻译：**  
过程由名称、一些参数和一个程序体组成。  
**定义 2.10（过程）：**  
**过程 ⟨名称⟩（⟨变量⟩ : ⟨类型⟩） ≜ ⟨程序体⟩**
**解释：**  
这个定义说明了如何形式化地表示一个过程。过程就是命名的代码块，带有参数和实现体。

**引理 2.11（过程值传递规格 Procedure Value Specification）**
**英文原文：**  
Given a procedure `procedure ⟨Name⟩(value f : Type) ≜ w`,  
且 `f : [pre, post]`，其中 w 与 f 是不同变量。  
设 A 是实际的参数表达式。  
则有：  
**w : [pre ⟨f := A⟩, post ⟨f0 := A0⟩] ⊑ procedure ⟨Name⟩(A)**
**中文翻译：**  
设有过程 `procedure ⟨名称⟩(value f : 类型) ≜ w`，  
并且 `f : [前置条件, 后置条件]`，其中 w 与 f 是不同变量，  
设 A 是该类型的一个实际参数表达式，  
则有：  
**w : [前置条件 ⟨f := A⟩, 后置条件 ⟨f₀ := A₀⟩] ⊑ 过程 ⟨名称⟩(A)**
**解释：**  
这个法则描述了**按值传参的过程精化方式**：如果你有一个过程，它接收参数 `f`，那么你可以将其精化为将 `f` 替换成实际参数 `A` 之后的规范语句。

**引理 2.12（过程返回值规格 Procedure Result Specification）**
**英文原文：**  
Given a procedure `procedure ⟨Name⟩(result f : Type) ≜ w`,  
且 `f : [pre, post ⟨a := f⟩]`，其中 w, a, f 为不同变量，  
则有：  
**w, a : [pre, post] ⊑ procedure ⟨Name⟩(a)**
**中文翻译：**  
设有过程 `procedure ⟨名称⟩(result f : 类型) ≜ w`，  
并且 `f : [前置条件, 后置条件 ⟨a := f⟩]`，其中 w、a、f 是不同变量，  
则有：  
**w, a : [前置条件, 后置条件] ⊑ 过程 ⟨名称⟩(a)**
**解释：**  
这是**返回值传递的精化规则**。意思是说，你可以用过程调用替代赋值表达式 `a := f`，只要你知道该过程的结果将赋值给 `a`，并且满足其前后条件。


文献 [4, 42] 已经确立了本节中各项法则的正确性。特别地，我们定义一种类似 Hoare 三元组的表示法：  **{𝑝𝑟𝑒} 𝑝𝑟𝑜𝑔 {𝑝𝑜𝑠𝑡}**，表示：若程序 𝑝𝑟𝑜𝑔 从满足前置条件 𝑝𝑟𝑒 的状态开始执行，并且能够终止，那么其结束后将满足后置条件 𝑝𝑜𝑠𝑡。

我们在不给出证明的情况下陈述如下结果：

**定理 2.13（核心精化法则的正确性）：**  
若从第 2.2 节中的法则可以推导出 ® 𝑥 : [𝑝𝑟𝑒, 𝑝𝑜𝑠𝑡] ⊑ 𝑝𝑟𝑜𝑔，那么 {𝑝𝑟𝑒} 𝑝𝑟𝑜𝑔 {𝑝𝑜𝑠𝑡} 成立。


# 3 Motivating Example
本节通过平方根（sqrt）算法来说明我们对 Refine4LLM 的直觉理解。与其他程序精化相关工作 [41, 52] 相比，**我们将 sqrt 算法从整数扩展到了实数，并展示了我们是如何引导大语言模型（LLMs）生成代码以及验证其正确性的。**

接着，我们说明了一种学习策略，用于扩展精化法则，从而发展和拓展精化演算，以降低程序精化的复杂度。
## 3.1 Guide the LLM
平方根示例的规范描述如下：对于任意正实数常量 N 和 e，程序 C 将不断修改变量 x，直到满足 𝑥^2 ≤ 𝑁 且 (x + e)^2 > N。
程序精化的目标是将这一规范逐步细化为更小的部分，同时一步步构建出完整的程序。

我们从基本的精化演算出发，即第 2 节中定义的核心法则，并在图 2 中展示了整个精化过程。第一个可能的选择是使用**顺序组合法则（Sequential Composition Law）**，将两个约束条件𝑥^2 ≤ 𝑁和 𝑁 < (𝑥 + 𝑒)^2分离开来。

![[Pasted image 20250422170304.png]]

对于第一部分，大语言模型（LLM）可以利用规范推导出一个可能正确的赋值，比如 x = 0，这个赋值可以通过 Hoare 逻辑和自动定理证明器（ATP）进行验证。

接下来，Refine4LLM 可使用**迭代法则（Iteration Law）** 来处理第二部分，使用迭代的规范结构 [𝐼𝑛𝑣𝑎𝑟𝑖𝑎𝑛𝑡, 𝐼𝑛𝑣𝑎𝑟𝑖𝑎𝑛𝑡∧¬𝐺𝑢𝑎𝑟𝑑𝑠]。迭代法则会将后置条件拆解为两部分：守卫条件（guard condition）和不变式（invariant），并进一步构建出一个保持不变式的迭代结构，同时不断改变变式（variant），直到守卫条件不再成立。

符号 ↓ 表示变式严格递减。Refine4LLM 会将约束条件  
“𝐼𝑛𝑣𝑎𝑟𝑖𝑎𝑛𝑡 ∧ 𝐺𝑢𝑎𝑟𝑑𝑠 → 𝐼𝑛𝑣𝑎𝑟𝑖𝑎𝑛𝑡 ∧ Variant is strictly decreasing”  
作为提示，指导 LLM 生成如 `x = x + e` 的代码。

最后，Refine4LLM 会验证所生成的赋值语句是否能够保持不变式，并使变式递减。需要注意的是，这样的验证属于**部分正确性**，因为它只验证了变式在每一步都在递减。

若要达到**完全正确性**，还需要验证在某一次迭代中，守卫条件最终会被违反（即迭代最终终止）。

## 3.2 Failure Feedback




## 3.3 Optimizing the Algorithm with Binary Search


## 3.4 Learning Strategies for Extending the Refinement Calculus