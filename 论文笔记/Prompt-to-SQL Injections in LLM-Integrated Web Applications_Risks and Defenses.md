
## links
* [[2308.01990] From Prompt Injections to SQL Injection Attacks: How Protected is Your LLM-Integrated Web Application?](https://arxiv.org/abs/2308.01990)
* [Prompt-to-SQL Injections in LLM-Integrated Web Applications: Risks and Defenses](https://www.computer.org/csdl/proceedings-article/icse/2025/056900a076/215aWuWbxeg)
## 0 ABSTRACT

大语言模型（LLMs）已广泛应用于多个领域，包括具有聊天机器人接口的 Web 应用程序。

在诸如 LangChain 之类的 LLM 集成中间件的支持下，用户输入的提示被转换为 SQL 查询，以便 LLM 生成有意义的响应。然而，未经处理的用户输入可能导致 SQL 注入攻击，进而危及数据库的安全性。

在本文中，我们对基于 LangChain 和 LlamaIndex 等框架的 Web 应用程序中的**提示到 SQL（P2SQL）注入**进行了全面分析。我们对 P2SQL 注入的不同变体及其对应用安全性的影响进行了刻画，并通过多个具体示例进行探讨。此外，我们评估了七种最先进的 LLM，展示了 P2SQL 攻击在不同语言模型中的风险。

通过手动和自动化方法，我们在五个真实世界的应用中发现了 P2SQL 漏洞。我们的研究表明，LLM 集成应用极易受到 P2SQL 注入攻击，因此迫切需要采取有效的防御措施。为应对这些攻击，我们提出了四种可作为 LangChain 框架扩展的防御技术。