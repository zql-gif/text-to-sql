# BIRD-SQL: A BIg Bench for Large-Scale Relational Database Grounded Text-to-SQLs

## Overview

BIRD-SQL is the first cross-domain large-scale benchmark specifically designed to bridge the gap between academic research and real-world applications in the field of text-to-SQL parsing. While models such as Codex and ChatGPT have demonstrated remarkable performance, existing benchmarks such as Spider and WikiSQL concentrate primarily on database schema, leaving database contents largely unexplored. Realizing this limitation, we set out to create a comprehensive benchmark that delves deeper into database values, ultimately unveiling new challenges and opportunities for developments in the text-to-SQL domain.
BIRD-SQL是首个专门针对文本到SQL解析领域学术研究与实际应用差距而设计的跨领域大规模基准测试。尽管Codex、ChatGPT等模型已展现出卓越性能，但现有Spider、WikiSQL等基准主要聚焦于数据库模式层面，对数据库内容鲜有深入探索。基于这一局限性认知，我们着手创建了一个深度挖掘数据库具体数值的综合基准，最终揭示了文本到SQL领域发展的新挑战与机遇。

BIRD-SQL is distinguished by its large dataset, which includes **12,751** text-to-SQL pairs, **95** databases encompassing **37** professional domains, and a total size of **33.4 GB**. By highlighting database values, BIRD-SQL draws attention to new challenges, such as external knowledge, dirty data, and SQL efficiency in vast databases. In order to generate accurate SQL queries, models must not only conduct semantic parsing but also comprehend database values.
BIRD-SQL的独特价值体现在其海量数据集：包含**12,751**组文本-SQL对应样本、覆盖**37**个专业领域的**95**个数据库，总规模达**33.4 GB**。通过突出数据库具体数值的核心地位，该基准凸显了三大创新性挑战维度——外部知识整合能力、脏数据处理能力以及海量数据库环境下的SQL执行效率优化。研究结果表明，要生成精准的SQL查询语句，模型不仅需要完成语义解析，更需深入理解数据库内存储的具体数值信息。

## Dataset Introduction

The dataset contains the main following resources:
- `database`: The database should be stored under the [`./data/dev_databases/`](./data/dev_databases/). In each database folder, it has two components:
  - `database_description`: the csv files are manufactured to describe database schema and its values for models to explore or references.
  - `sqlite`: The database contents in BIRD.
- `data`: Each text-to-SQL pairs with the oracle knowledge evidence is stored as a json file, i.e., `dev.json` is stored on [`./data/dev.json`](./data/dev.json). In each json file, it has three main parts:
  - `db_id`: the names of databases
  - `question`: the questions curated by human crowdsourcing according to database descriptions, database contents.
  - `evidence`: the external knowledge evidence annotated by experts for assistance of models or SQL annotators.
  - `SQL`: SQLs annotated by crowdsource referring to database descriptions, database contents, to answer the questions accurately.
- `ground-truth SQL file`: The SQL file should be stored at [`./llm/data/dev_gold.sql`](./llm/data/dev_gold.sql).
- `llm`: It contains source codes to convert texts to SQLs by calling APIs from LLMs, such as `code-davinci-002`, `gpt-3.5-turbo`.
- `finetuning`: It contains the codes for supervised fine-tuning T5, a prevalent sequence-to-sequence pre-trained language model, to perform text-to-SQL task in BIRD.

该数据集主要包含以下资源：
- **`database`（数据库文件）**：存储于[`./data/dev_databases/`](./data/dev_databases/)路径下。每个数据库文件夹包含两部分：
	- **`database_description`（数据库描述）**： 专门制作的CSV文件，用于描述数据库模式及其具体数值，供模型探索或参考。
	- **`sqlite`（数据库内容）**： BIRD中实际存储的数据库内容。
- **`data`（文本-SQL配对数据）**： 每个文本到SQL的配对数据及关联的知识证据均以JSON文件形式存储，例如开发集文件`dev.json`位于[`./data/dev.json`](./data/dev.json)。每个JSON文件包含四个核心字段：
	- **`db_id`**：数据库名称
	- **`question`**：基于数据库描述与内容，通过人工众包标注的问题
	- **`evidence`**：专家标注的外部知识证据，用于辅助模型或SQL标注者
	- **`SQL`**：参考数据库描述与内容，通过众包标注生成的精准SQL语句
- **`ground-truth SQL file`（基准真值SQL文件）**： 存储于[`./llm/data/dev_gold.sql`](./llm/data/dev_gold.sql)，包含标准SQL查询语句。
- **`llm`（大语言模型接口）**：  包含通过调用大语言模型API（如`code-davinci-002`、`gpt-3.5-turbo`）实现文本到SQL转换的源代码。
- **`finetuning`（微调模块）**： 包含对T5模型（一种主流序列到序列预训练语言模型）进行监督式微调的代码，用于在BIRD数据集上执行文本到SQL任务。

## ~~Fine-tuning (FT)~~
### Environment Setup:

To train T5 via an end-to-end FT method, please first create enviroments following [`UnifiedSKG`](https://github.com/HKUNLP/UnifiedSKG).
You may also need to download the third party packages for evaluations:

若要通过端到端微调（FT）方法训练T5模型，请首先按照[`UnifiedSKG`](https://github.com/HKUNLP/UnifiedSKG)的指引搭建运行环境。  
同时，您还需下载以下第三方依赖包以进行模型评估：

```bash
git submodule update --init --recursive
```

```bash
cd ./finetuning/
conda env create -f finetuning
conda activate finetuning
# install the hugginface package: datasets according to your version.
pip install datasets
# The following line to be replaced depending on your cuda version.
pip install torch==1.11.0+cu113 torchvision==0.12.0+cu113 torchaudio==0.11.0 --extra-index-url https://download.pytorch.org/whl/cu113
```

### Training:

All parameters and attempts are stored in the [`./finetuning/run/`](./finetuning/run/). Please start training by the following commands:

所有配置参数与训练记录均存储于[`./finetuning/run/`](./finetuning/run/)目录下。请通过以下命令启动训练流程：

```bash
sh ./run/run_bird_large.sh
```

## In-Context Learning (ICL):

### Environment Setup:

First, you need install openai in your python environment by:

```bash
pip install openai
```

### Collect results

Then you could directly execute the command line by following instructions (you may need to adjust paramters and paths with your preference):

随后，您可直接按照以下指引执行命令行（您可能需要根据实际需求调整参数与路径）：

```bash
cd ./llm/
sh ./run/run_gpt.sh
```

## Evaluation:

### Execution (EX) Evaluation:

Please post-process your collected results as the format: SQL and its `db_id`, which is splitted by `'\t----- bird -----\t'`. The examples are shown in the [`./llm/exp_result/turbo_output/predict_dev.json`](./llm/exp_result/turbo_output/predict_dev.json). Put the ground-truth sql file in the [`./data/`](./data/). And you may need to design a ChatGPT tag by your own.
The main file for ex evaluation is located at [`./llm/src/evaluation.py`](./llm/src/evaluation.py). \
Then you could evaluate the results by the following command line :


请对收集的结果进行后处理，格式要求为：**SQL语句**与其对应的`db_id`，两者通过`'\t----- bird -----\t'`分隔。具体示例可参考[`./llm/exp_result/turbo_output/predict_dev.json`](./llm/exp_result/turbo_output/predict_dev.json)文件。将基准真值SQL文件置于[`./data/`](./data/)目录下。您可能需要根据需求自行设计ChatGPT标签。  
核心评估脚本位于[`./llm/src/evaluation.py`](./llm/src/evaluation.py)，随后可通过以下命令行执行评估：

```bash
cd ./llm/
sh ./run/run_evaluation.sh
```

### Valid Efficiency Score (VES) Evaluation (time-mainly):

In the newest version, ves and ex can be evaluated in the same shell. Then main file is [`./llm/src/evaluation_ves.py`](./llm/src/evaluation_ves.py), so you can eval your efficiency via:

在最新版本中，**ves**（数值效率验证）与**ex**（执行评估）可在同一Shell环境中完成评估。核心评估脚本已更新至[`./llm/src/evaluation_ves.py`](./llm/src/evaluation_ves.py)，您可通过以下命令执行效率评估：

```bash
cd ./llm/
sh ./run/run_evaluation.sh
```
(For stable VES, you may need to enlarge `timeout` or repeat and average results. In our test evaluation, we will enlarge `timeout` to 3 s/ex; then we repeat 5 times for VES computation, only the highest results will be reported.)
为了确保 VES 的稳定性，您可能需要增大超时时间或重复计算并取平均值。在我们的测试评估中，我们将超时时间扩大到 3 秒/次；然后，我们对 VES 计算重复 5 次，仅报告最高结果。

---
## Test Evaluation

If your code scripts don't need complex environment setting up and can fetch results via openai-api mainly. Please connect bird.bench23@gmail.com for fast test. 

## Acknowledgement

We thank Xin Yan for active involvement in ChatGPT prompt design, discussion, and figure creation in the HKU STAR Lab. We thank Jan Motl, Oliver Schulte for valuable suggestions and assistance in maintaining the databases from [`https://relational.fit.cvut.cz/`](https://relational.fit.cvut.cz/).

## Call for Calibration

In this work, we are committed to delivering high-quality datasets to boost the development of text-to-SQL research. Despite our hard efforts in evaluating and refining this benchmark with ~700 hours, we acknowledge that errors and ambiguities may still exist. To ensure long-term contributions to the text-to-SQLs, we are actively soliciting community feedback on possible enhancements to our datasets. Please consider reporting errors or your suggestions on the [`ISSUES`](https://github.com/AlibabaResearch/DAMO-ConvAI/issues/39), or via emailing us by bird.bench23@gmail.com.

We will also polish this benchmark periodically. Therefore, We would be grateful if you could provide any feedback regarding errors or future directions to BIRD. Let's contribute to the future of text-to-SQL research. Thank you for your support!

## Insteresting Stories about Values:
we are welcome to any findings during experiments about interaction with database values. For example, we find that GPT4-32k even fails to consider the tied results in a joined tables correctly. \
In the `dev_1388`, the predicted SQL of GPT4-32k is:
```
Question: Which students manage to generate the highest income. State his/her name along with the income source.
```
```sql
SELECT T1.first_name, T1.last_name, T2.source  
FROM member AS T1  
INNER JOIN income AS T2 ON T1.member_id = T2.link_to_member  
WHERE T2.amount = (  
    SELECT MAX(amount)  
    FROM income  
)  
ORDER BY T2.amount DESC
```
it leads to a NULL result set since `MAX(amount)` is `3000` in the orignal table income. However, the ground-truth SQL should consider the `MAX(amount)` in the joined table pertaining to tables `member` and `income`. Therefore, the largest amount is only `50`, and the ground-truth SQL should be:
```sql
SELECT T1.first_name, T1.last_name, T2.source
FROM member AS T1
INNER JOIN income AS T2
ON T1.member_id = T2.link_to_member
WHERE T2.amount = (
    SELECT MAX(T4.amount)
    FROM member AS T3
    INNER JOIN income AS T4
    ON T3.member_id = T4.link_to_member
    )
```
We hypothesize that GPT-4 is pre-trained based on semantic parsing framework, losing the enough attention on values. This may also be marked as the initial challenge in achieving Artificial General Intelligence (AGI) for real-world text-to-SQL applications.

## Citation

Please cite the repo if you think our work is helpful to you.

```
@misc{li2023llm,
  title={Can LLM Already Serve as A Database Interface? A BIg Bench for Large-Scale Database Grounded Text-to-SQLs},
  author={Jinyang Li and Binyuan Hui and Ge Qu and Binhua Li and Jiaxi Yang and Bowen Li and Bailin Wang and Bowen Qin and Ruiying Geng and Nan Huo and Xuanhe Zhou and Chenhao Ma and Guoliang Li and Kevin C. C. Chang and Fei Huang and Reynold Cheng and Yongbin Li},
  year={2023},
  eprint={2305.03111},
  archivePrefix={arXiv},
  primaryClass={cs.CL}
}
```
