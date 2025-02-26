1. 在Spider的基础上实现: MR-SQLGen， 通过询问大模型，实现等价关系、混排关系、包含关系、交集关系和并集关系的 metamorphic prompt，测试LLM。
* 蜕变关系：[https://linli1724647576.github.io/2023/07/18/%E4%BB%80%E4%B9%88%E6%98%AF%E8%9C%95%E5%8F%98%E6%B5%8B%E8%AF%95/](https://linli1724647576.github.io/2023/07/18/%E4%BB%80%E4%B9%88%E6%98%AF%E8%9C%95%E5%8F%98%E6%B5%8B%E8%AF%95/)
* 可以通过CoT+few-shot的形式，找到合适的蜕变点，再让LLM找到合适的蜕变空间， For example：查询员工表中年龄小于10岁的人的数量 --> 蜕变点 “年龄小于10岁” -> 利用LLM的语义理解能力，我们可以扩展出包含关系，查询员工表中年龄小于20/30/40岁的人的数量 -->并集关系，查询员工表中年龄大于等于10岁的人的数量



* MR-SQLGen的基础实现：Basic Instruction(llm对蜕变关系的理解不太准确)+CoT(蜕变点，蜕变空间的分步进行)+Few-Shot
* 障碍
	* 蜕变点：有的样例不一定包含明显且合适的蜕变点，比方说：What is the total number of singers ?
	* 蜕变空间：比方说包含关系（收紧或者扩大范围，需要得到database schema，开销大；方法是否可以设计为尽量不依赖于database schema的），交集关系，并集关系同理。等价关系和混排关系的区别是什么？？有交集吗？？