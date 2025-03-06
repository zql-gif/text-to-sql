# Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task

Spider is a large human-labeled dataset for complex and cross-domain semantic parsing and text-to-SQL task (natural language interfaces for relational databases). It is released along with our EMNLP 2018 paper: [Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task](https://arxiv.org/abs/1809.08887). This repo contains all code for evaluation, preprocessing, and all baselines used in our paper. Please refer to [the task site](https://yale-lily.github.io/spider) for more general introduction and the leaderboard.

:+1: `03/20/2022`: **We open-sourced a simple but SOTA model (just T5) for the task! Please check out our code in the [UnifiedSKG repo](https://github.com/hkunlp/unifiedskg)!!**

### Installation

`evaluation.py` and `process_sql.py` are written in Python 3. Enviroment setup for each baseline is in README under each baseline directory.

### Data Content and Format

#### Question, SQL, and Parsed SQL

Each file in`train.json` and `dev.json` contains the following fields:
- `question`: the natural language question
- `question_toks`: the natural language question tokens
- `db_id`: the database id to which this question is addressed.
- `query`: the SQL query corresponding to the question. 
- `query_toks`: the SQL query tokens corresponding to the question. 
- `sql`: parsed results of this SQL query using `process_sql.py`. Please refer to `parsed_sql_examples.sql` in the`preprocess` directory for the detailed documentation.

`train.json` 和 `dev.json` 中的每个文件包含以下字段：
- `question`：自然语言问题
- `question_toks`：自然语言问题的tokens
- `db_id`：该问题所针对的数据库的 ID
- `query`：与问题对应的 SQL 查询
- `query_toks`：与问题对应的 SQL 查询的tokens
- `sql`：使用 `process_sql.py` 解析后的 SQL 查询结果。详细文档请参考 `preprocess` 目录中的 `parsed_sql_examples.sql` 文件。

```
 {
        "db_id": "world_1",
        "query": "SELECT avg(LifeExpectancy) FROM country WHERE Name NOT IN (SELECT T1.Name FROM country AS T1 JOIN countrylanguage AS T2 ON T1.Code  =  T2.CountryCode WHERE T2.Language  =  \"English\" AND T2.IsOfficial  =  \"T\")",
        "query_toks": ["SELECT", "avg", "(", "LifeExpectancy", ")", "FROM", ...],
        "question": "What is average life expectancy in the countries where English is not the official language?",
        "question_toks": ["What", "is", "average", "life", ...],
        "sql": {
            "except": null,
            "from": {
                "conds": [],
                "table_units": [
                    ...
            },
            "groupBy": [],
            "having": [],
            "intersect": null,
            "limit": null,
            "orderBy": [],
            "select": [
                ...
            ],
            "union": null,
            "where": [
                [
                    true,
                    ...
                    {
                        "except": null,
                        "from": {
                            "conds": [
                                [
                                    false,
                                    2,
                                    [
                                    ...
                        },
                        "groupBy": [],
                        "having": [],
                        "intersect": null,
                        "limit": null,
                        "orderBy": [],
                        "select": [
                            false,
                            ...
                        "union": null,
                        "where": [
                            [
                                false,
                                2,
                                [
                                    0,
                                   ...
        }
    },

```

#### Tables

`tables.json` contains the following information for each database:
- `db_id`: database id
- `table_names_original`: original table names stored in the database.
- `table_names`: cleaned and normalized table names. We make sure the table names are meaningful. [to be changed]
- `column_names_original`: original column names stored in the database. Each column looks like: `[0, "id"]`. `0` is the index of table names in `table_names`, which is `city` in this case. `"id"` is the column name. 
- `column_names`: cleaned and normalized column names. We make sure the column names are meaningful. [to be changed]
- `column_types`: data type of each column
- `foreign_keys`: foreign keys in the database. `[3, 8]` means column indices in the `column_names`. These two columns are foreign keys of two different tables.
- `primary_keys`: primary keys in the database. Each number is the index of `column_names`.


`tables.json` 为每个数据库包含以下信息：
- `db_id`：数据库 ID，是字符串形式表示的db名字。
- `table_names_original`：存储在数据库中的原始表名。
- `table_names`：清理和标准化后的表名。我们确保表名有意义。
- `column_names_original`：存储在数据库中的原始列名。每个列名的格式为：`[0, "id"]`。`0` 是 `table_names` 中表名的索引，在本例中为 `city`。`"id"` 是列名。
- `column_names`：**清理和标准化后的列名。我们确保列名有意义。** 内部有所有表的所有列名，每个列以[table_id, column_name]的形式存储（table_id为非负数，-1表示无意义的列）。
- `column_types`：每个列的数据类型。**与上面从column_names一一对应。**
- `foreign_keys`：数据库中的外键。`[3, 8]` 表示在 `column_names` 中的列索引。这两列是两个不同表的外键。
- `primary_keys`：**数据库中的主键。每个数字是 `column_names` 中的索引。（从0开始的）***

```
{
    "column_names": [
      [
        0,
        "id"
      ],
      [
        0,
        "name"
      ],
      [
        0,
        "country code"
      ],
      [
        0,
        "district"
      ],
      .
      .
      .
    ],
    "column_names_original": [
      [
        0,
        "ID"
      ],
      [
        0,
        "Name"
      ],
      [
        0,
        "CountryCode"
      ],
      [
        0,
        "District"
      ],
      .
      .
      .
    ],
    "column_types": [
      "number",
      "text",
      "text",
      "text",
         .
         .
         .
    ],
    "db_id": "world_1",
    "foreign_keys": [
      [
        3,
        8
      ],
      [
        23,
        8
      ]
    ],
    "primary_keys": [
      1,
      8,
      23
    ],
    "table_names": [
      "city",
      "sqlite sequence",
      "country",
      "country language"
    ],
    "table_names_original": [
      "city",
      "sqlite_sequence",
      "country",
      "countrylanguage"
    ]
  }
```


#### Databases

All table contents are contained in corresponding SQLite3 database files.

### Evaluation

Update 11/15/20: We will use [Test Suite Accuracy](https://arxiv.org/abs/2010.02840) as our official evaluation metric for Spider, SParC, and CoSQL. Please find the evaluation code from [here](https://github.com/taoyds/test-suite-sql-eval).
更新日期：2020年11月15日：我们将使用 [Test Suite Accuracy](https://arxiv.org/abs/2010.02840) 作为 Spider、SParC 和 CoSQL 的官方评估指标。评估代码请从 [这里](https://github.com/taoyds/test-suite-sql-eval) 获取。

Our evaluation metrics include Component Matching, Exact Matching, and Execution Accuracy. For component and exact matching evaluation, instead of simply conducting string comparison between the predicted and gold SQL queries, we decompose each SQL into several clauses, and conduct set comparison in each SQL clause. 
我们的评估指标包括组件匹配、精确匹配和执行准确性。对于组件匹配和精确匹配评估，我们不仅仅进行预测和真实 SQL 查询之间的字符串比较，而是将每个 SQL 语句分解为多个子句，并对每个 SQL 子句进行集合比较。

For Execution Accuracy, our current models do not predict any value in SQL conditions so that we do not provide execution accuracies. However, we encourage you to provide it in the future submissions. For value prediction, you can assume that a list of gold values for each question is given. Your model has to fill them into the right slots in the SQL.
对于执行准确性，我们当前的模型不预测 SQL 条件中的任何值，因此我们不提供执行准确性。不过，我们鼓励您在未来的提交中提供此项指标。对于值预测，您可以假设每个问题都有一个给定的金标准值列表。您的模型需要将这些值填充到 SQL 中的正确位置。

Please refer to [our paper]() and [this page](https://github.com/taoyds/spider/tree/master/evaluation) for more details and examples.
更多细节和示例，请参考 [我们的论文](https://chatgpt.com/c/67be8671-6520-8002-ab8f-7a78f7650db6) 和 [这个页面](https://github.com/taoyds/spider/tree/master/evaluation)。


```
python evaluation.py --gold [gold file] --pred [predicted file] --etype [evaluation type] --db [database dir] --table [table file]

arguments:
  [gold file]        gold.sql file where each line is `a gold SQL \t db_id`
  [predicted file]   predicted sql file where each line is a predicted SQL
  [evaluation type]  "match" for exact set matching score, "exec" for execution score, and "all" for both
  [database dir]     directory which contains sub-directories where each SQLite3 database is stored
  [table file]       table.json file which includes foreign key info of each database

参数： 
[gold file]： gold.sql 文件，每行包含 `a gold SQL \t db_id` 
[predicted file]: 预测 SQL 文件，每行包含一个预测的 SQL 
[evaluation type]: "match" 表示精确集合匹配分数，"exec" 表示执行分数，"all" 表示两者兼有 
[database dir]: 包含子目录的目录，每个 SQLite3 数据库存储在一个子目录中
[table file]: table.json 文件，包含每个数据库的外键信息
```

### Spider_data的目录说明

该文件夹包含了EMNLP 2018论文《Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task》的Spider训练和开发数据集。

它包含以下文件：

---
- dev.json  : 开发数据集，列表形式存储text-to-sql对儿
    * Training Examples: 1034  
    * Databases:       20  
- dev_gold.sql （1034） ：开发数据集，行形式存储所有的text-to-sql对儿中的标准答案sql语句
- tables.json  ：存储所有table的基本信息，格式见本文件前面的tables部分
    * Databases:        166  
- database/  （166个子目录）: 每个数据库都有一个子文件夹，包含一个[db_name].sqlite文件。 对于大多数数据库，我们还提供了一个schema.sql文件，其中包含用于创建数据库的SQL语句。
---
- test.json  : 测试数据集，列表形式存储text-to-sql对儿
    * Testing Examples: 2147
    * Databases:       40
- test_gold.sql (2147) : 测试数据集，行形式存储所有的text-to-sql对儿中的标准答案sql语句
- test_tables.json  ：存储所有table的基本信息，格式见本文件前面的tables部分
    * Databases:        206
- test_database/ （206个子目录）: 每个数据库都有一个子文件夹，包含一个[db_name].sqlite文件。 对于大多数数据库，我们还提供了一个schema.sql文件，其中包含用于创建数据库的SQL语句。
---
- train_spider.json  : 训练数据集，列表形式存储text-to-sql对儿
	* Training Examples: 7000  
	* Databases:       140  （均被database/和test_database/ 分别包含，评估时用哪个都成）
- train_others.json  ：训练数据集，列表形式存储text-to-sql对儿
    * Training Examples: 1659  
    * Databases:       6  （均被database/和test_database/ 分别包含，评估时用哪个都成）
- train_gold.sql  （7000+1659=8659）：合并了train_spider.json和train_others.json，训练数据集，行形式存储所有的text-to-sql对儿中的标准答案sql语句
---
- README.txt  ：解释说明

其他说明如下：
* 官方的最终Spider训练数据集结合了train_spider.json和train_others.json。
* train_others.json中使用的数据库来自Restaurants、GeoQuery、Scholar、Academic、IMDB和Yelp，这些数据库由Finegan-Dollak等人于2018年准备。 
* train_spider.json中的数据库和SQL示例是由我们最初准备的。 
* 有关每个json文件的格式，请参考我们的GitHub页面：[https://github.com/taoyds/spider。](https://github.com/taoyds/spider%E3%80%82)


### Changelog
-`11/15/2020` We will use [Test Suite Accuracy](https://arxiv.org/abs/2010.02840) as our official evaluation metric for Spider, SParC, and CoSQL. Please find the evaluation code from [here](https://github.com/taoyds/test-suite-sql-eval).
- `08/03/2020` Corrected `column_name` and `column_name_original` mismatches in 2 dbs (`scholar` and `formula_1`) in `tables.json`, and reparsed SQL queries (this only affects some models (e.g. RATSQL) which use our parsed SQL as the SQL input). Please download the Spider dataset from [the page](https://yale-lily.github.io/spider) again.
- `06/07/2020` We corrected some annotation errors and label mismatches (not errors) in Spider dev and test sets (~4% of dev examples updated, click [here](https://github.com/taoyds/spider/commit/25fcd85d9b6e94acaeb5e9172deadeefeed83f5e#diff-18b0a730a7b0d29b0a78a5070d971d49) for more details). Please download the Spider dataset from [the page](https://yale-lily.github.io/spider) again.
- `01/16/2020` For value prediction (in order to compute the execution accuracy), your model should be able to 1) copy from the question inputs, 2) retrieve from the database content (database content is available), or 3) generate numbers (e.g. 3 in "LIMIT 3").
- `1/14/2019` The submission toturial is ready! Please follow it to get your results on the unreleased test data.
- `12/17/2018` We updated 7 sqlite database files. Please download the Spider data from the official website again. Please refer to [the issue 14](https://github.com/taoyds/spider/issues/14) for more details.
- `10/25/2018`: evaluation script is updated so that the table in `count(*)`cases will be evaluated as well. Please check out [the issue 5](https://github.com/taoyds/spider/issues/5) for more info. Results of all baselines and [syntaxSQL](https://github.com/taoyds/syntaxSQL) on the papers are updated as well.
- `10/25/2018`: to get the latest SQL parsing results (a few small bugs fixed), please use `preprocess/parse_raw_json.py` to update. Please refer to [the issue 3](https://github.com/taoyds/spider/issues/3) for more details.

### Citation

The dataset is annotated by 11 college students. When you use the Spider dataset, we would appreciate it if you cite the following:

```
@inproceedings{Yu&al.18c,
  title     = {Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task},
  author    = {Tao Yu and Rui Zhang and Kai Yang and Michihiro Yasunaga and Dongxu Wang and Zifan Li and James Ma and Irene Li and Qingning Yao and Shanelle Roman and Zilin Zhang and Dragomir Radev}
  booktitle = "Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing",
  address   = "Brussels, Belgium",
  publisher = "Association for Computational Linguistics",
  year      = 2018
}
```


### FAQ