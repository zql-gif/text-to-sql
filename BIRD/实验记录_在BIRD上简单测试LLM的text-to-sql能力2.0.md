## links

## setup

### 环境
见github项目中的requirements.txt
### bird数据集下载
[BIRD_数据集说明](../BIRD/BIRD_数据集说明.md)

## BIRD-SQL官方代码说明（修改）
**关于官方代码，有以下几点事实：**
* 官方提供了dev.json数据的text-to-sql结果，直接提供在其官网（配置见下面的dev_kg_1.0）
* **官方代码中并没有做所谓的cot的处理，没有这一处理分支，故不存在思维链的情况。只有如下的内容，所有的都添加了这句。**
```c
def cot_wizard():  
    cot = "\nGenerate the SQL after thinking step by step: "  
    return cot
```
* **官方代码对于每条数据的测试过程，没有进行错误处理和单条测试内容的记录，所以这里完善修改了这部分的代码**
* 官方代码的evaluation部分，对于数据集的处理有误，数据集train.json文件中并未标注其困难等级，代码却默认数据中包含了表示困难等级的键“difficulty”
* 官方代码的evaluation部分，对于测试数据生成的sql语句，评估前没有对sql进行执行结果的获取（根本没有进行执行），所以评估代码直接报错

具体修改如下：
* 记录每条数据的cost
* 每完成一条数据的测试，当即将每条测试数据的中间结果(sql,cost,格式同spider的实验)按行记录到predict.jsonl格式文件中，最后再读取中间记录的结果并合并为predict.json文件。以防止出现程序崩溃而中间数据丢失。
* 默认设置所有的测试数据的'difficulty'键为简单。
* 为evaluation部分添加sql语句的语句执行代码和结果存储。
## In-Context Learning (ICL):
### Collect results

随后，您可直接按照以下指引执行命令行（您可能需要根据实际需求调整参数与路径）：

```bash
cd ./llm/
sh ./run/run_gpt.sh
```

#### dev_kg_1.0
1. 测试数据取自`bird`中的测试文件dev.json 的1534条,  对应的database schema文件dev_tables.json以及标准答案dev.sql
2. input : bird数据集的dev文件 ,  以及对应的database schema文件tables.json
3. output : 生成的sql文件 "predict.txt" ，llm的生成结果和代价的记录文件predict.jsonl
4. llm setup 
	* model = "gpt-3.5-turbo"
	* temperature = 0.0（代码中写固定的值）
	* 测试数目：1534条

run_gpt.sh的内容如下：
```shell
eval_path='../data/dev.json'  
dev_path='../output/'  
db_root_path='../data/dev_databases/'  
use_knowledge='True'  
not_use_knowledge='False'  
mode='dev' # choose dev or dev  
cot='False'  
no_cot='True'
cnt=10
  
YOUR_API_KEY=''  
  
engine1='code-davinci-002'  
engine2='text-davinci-003'  
engine3='gpt-3.5-turbo'  
  
# data_output_path='./exp_result/gpt_output/'  
# data_kg_output_path='./exp_result/gpt_output_kg/'  
  
data_output_path='../exp_result/turbo_output/'  
data_kg_output_path='../exp_result/turbo_output_kg/'  
  
echo 'generate GPT3.5 batch with knowledge'  
python -u ../src/gpt_request.py --db_root_path ${db_root_path} --api_key ${YOUR_API_KEY} --mode ${mode} \  
--engine ${engine3} --eval_path ${eval_path} --data_output_path ${data_kg_output_path} --use_knowledge ${use_knowledge} \  
--chain_of_thought ${cot}  
  
# echo 'generate GPT3.5 batch without knowledge'  
# python3 -u ./src/gpt_request.py --db_root_path ${db_root_path} --api_key ${YOUR_API_KEY} --mode ${mode} \  
#--engine ${engine3} --eval_path ${eval_path} --data_output_path ${data_output_path} --use_knowledge ${not_use_knowledge} \  
#--chain_of_thought ${cot}
```
#### train_kg_1.0
1. 测试数据取自`bird`中的测试文件train.json 的xxx条,  对应的database schema文件train_tables.json以及标准答案train_gold.sql
2. input : bird数据集的train文件 ,  以及对应的database schema文件tables.json
3. output : 生成的sql文件 "predict.txt" ，llm的生成结果和代价的记录文件predict.jsonl
4. llm setup 
	* model = "gpt-3.5-turbo"
	* temperature = 0.0（代码中写固定的值）
	* 测试数目：xxx条

run_gpt.sh的内容如下：
```shell
eval_path='../data/train.json'  
dev_path='../output/'  
db_root_path='../data/train_databases/'  
use_knowledge='True'  
not_use_knowledge='False'  
mode='train' # choose dev or dev  
cot='False'  
no_cot='True'  
cnt=10
  
YOUR_API_KEY=''  
  
engine1='code-davinci-002'  
engine2='text-davinci-003'  
engine3='gpt-3.5-turbo'  
  
# data_output_path='./exp_result/gpt_output/'  
# data_kg_output_path='./exp_result/gpt_output_kg/'  
  
data_output_path='../exp_result/turbo_output/'  
data_kg_output_path='../exp_result/turbo_output_kg/'  
  
echo 'generate GPT3.5 batch with knowledge'  
python -u ../src/gpt_request.py --db_root_path ${db_root_path} --api_key ${YOUR_API_KEY} --mode ${mode} \  
--engine ${engine3} --eval_path ${eval_path} --data_output_path ${data_kg_output_path} --use_knowledge ${use_knowledge} \  
--chain_of_thought ${cot}  
  
# echo 'generate GPT3.5 batch without knowledge'  
# python3 -u ./src/gpt_request.py --db_root_path ${db_root_path} --api_key ${YOUR_API_KEY} --mode ${mode} \  
#--engine ${engine3} --eval_path ${eval_path} --data_output_path ${data_output_path} --use_knowledge ${not_use_knowledge} \  
#--chain_of_thought ${cot}
```

### Results Format
* predict.jsonl: 记录每条数据的生成sql结果和代价: id, sql, cost
* predict.json: BIRD-SQL官方代码的最终sql结果合并文档，后续评估需要使用。包含id:sql。

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
