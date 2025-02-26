Idea 1: Metamorphic prompt for LLM
* Motivation：判断LLM在代码生成中的推理能力如何
* Plan1:构造一个Benchmark dataset,包含metamorphic prompt,test input 以及 test output,没有技术含金量。 仅仅只是测试。
* Plan 2: 自动化生成一些prompt，可以边生成边测试，使得这套流程更具有Usefulness。
* Plan 3: 在 Plan 2的基础上，测试完成后如何呢，可以想办法进行Repair。这就是一个Multi-agents的过程
* 具体的Metamorphic relationship，针对普通代码，例如Python,可以使用变量名替换，来判断结构是否发生改变; 可以修改动词，使得输出的结果满足另一层蜕变关系， add ->delete。
* 针对SQL生成，我们就可以解决test input的问题，数据库系统就是一个非常好的validator(假设没有bug)，可以检测并修复LLM在SQL生成方面的能力。
* 一些衍生的点: 可以通过自相矛盾的输出来诱发LLM生成恶意代码。 给定函数名 def add(); docstring里的描述说到add函数需要实现divide功能，验证LLM会不会产生自相矛盾的结果。



Target: 针对 LLM的SQL生成 进行检验，先比之前手工Bench的工具，**+自动化、+推理能力**
思路一:
针对Spider数据集，对Question进行文本解析，从解析树上构造一些metamorphic prompt，让LLM生成一些SQL语句，由数据库执行结果进行验证。
* https://hanlp.hankcs.com/demos/sdp.htm -->语义依存分析，语义图调研(结点操作，结点属性...)
* 在Spider数据集上设计简单的蜕变关系。Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SOL Task
* 模仿Pinolo的流程....

思路二
反向生成（sql-to-question），已有metamorphic-oracle based fuzzer来生成metamorphic sql pairs。能否从metamorphic sql pairs反向推出一对metamorphic prompts，具体而言，根据AST和特定的templalte来还原prompt。
思路二疑问：
1. 想通过满足蜕变关系的prompt测试大模型的能力？针对一对prompt，大模型的一对sql输出可能较为随机，其输出结果如何判断是否为满足蜕变关系（即此处的text-to-sql效果如何评价：通过执行结果）
2. sql-to-question的效果怎么评估的
* 怎么确定分别进行sql-to-question，得到的一对question满足蜕变关系
3. 为什么用蜕变关系这种question对儿来测试大模型text-to-sql的能力（依据）？蜕变关系的qusetions对儿相比于已有text-to-sql dataset的不同点在于什么？