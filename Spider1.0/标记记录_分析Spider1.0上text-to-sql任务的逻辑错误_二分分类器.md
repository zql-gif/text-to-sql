## 0 标记任务描述
### 0.1 人工标注
* 人工分析error type和root cause，表象可能是用错某个函数，根本原因是LLM对某个操作理解不当。总结出一系列error type和root cause。
	* 人工标注gpt-3.5-turbo_3.0：spider-train-merged-info https://docs.qq.com/sheet/DTG9La1VpbXd6TFhT?tab=000001
	* 人工标注gpt-3.5-turbo_2.0：spider-test-merged-info https://docs.qq.com/sheet/DTENnVHBZbHZDcHJ3?tab=000001
	* **整理确认人工标注gpt-3.5-turbo_3.0的spider-train-merged-info集合，用作二分分类器的效果评估集**：spider-train-merged-info-manual-label https://docs.qq.com/sheet/DRnhER3RQeG1YZEdE 
* 整理并总结error type的定义，重新核对并补充前面人工标记的结果（主要是error type和root cause的分析）
### 0.2 llm自动标注剩余的test cases with text to sql logic error
* PossibleTypeLLM
	* 简介：LLM先根据Error Types的定义，判断该实例可能属于的所有Error Types。这一做法比单独将实例输入到所有Error Types的BinaryClassifierLLM中进行判断，更经济快速。
* BinaryClassifierLLM
	* 微调得到BinaryClassifierLLM的必要性：以 Join Logic Hallucination 类型，提问llm根据定义判断测试用例是否属于该类型，发现未经微调的gpt幻觉现象较为严重（**倾向于回答yes，即便给的例子不是**），具体提问记录可见链接 https://chatgpt.com/share/67ebac8f-0144-8002-b716-cc696695da5d 
## 2 PossibleTypeLLM

### 2.1 Prompt构成
* spider.classify_plus.auto_label_possible_types函数
* 对于每个实例的error type的判断，输出格式为
``` python
[
{"type": "type string 1", "explanation": "explanation string 1"},
...
]
```

``` python
# logic error type 定义文件位于路径"FineTuning_TrainData/BinaryClassifier/logic_error_intro_for_possible_type_label_2.txt"

# 组织问题内容  
question_content = (  
    f"问题: {question}\n"  
    f"Schema: {json.dumps(schema, ensure_ascii=False, indent=4)}\n"  
    f"正确答案: {correct_answer}\n"  
    f"LLM预测答案: {llm_predict_answer}\n"  
)

context_issue = [  
    {  
        "role": "system",  
        "content": "You are an expert in analyzing code issues and you will respond in Chinese."  
    },  
    {  
        "role": "user",  
        "content": (  
            f"Let's think step by step. 根据logic error type定义，依次思考并判断该text to sql任务的logic error type是否属于每一种type，最终给出所有可能的logic error type类型名称（一个text to sql任务可能涉及多个logic error types）？"  
            f"Logic error types类型名称:{error_types}\n"  
            f"Logic error types定义如下：{logic_error_type_intro}\n"  
            "请严格按照下面json样例的格式输出答案，仅输出符合[]列表的json格式，不要输出任何额外内容："  
            "[{\"type\": \"{{type string 1}}\", \"explanation\": \"{{explanation string 1}}\"}]\n"  
            f"{question_content}"  
        )  
    }  
]
```

### 2.2 效果评估
* spider.compare_auto_manual_possible_type_label函数
	* 对于每个实例，对比manual label的error types集合 和 PossibleTypeLLM auto label的error types集合
	* 对比是否有交集（至少标对一个）
	* 对比manual label是否为auto label的子集（是否标的不遗漏）
### 2.3 结果
* total cnt:77
* manual label和auto label有交集的cnt: 73
* manual label是auto label子集的cnt: 47
* manual label和auto label有交集的比例:0.948051948051948
* manual label是auto label子集的比例:0.6103896103896104
## 3 BinaryClassifierLLM

* BinaryClassifierLLM针对10余类错误类型的每一个，微调独立的模型。每个微调模型判断实例是否是否属于该错误类型。
	* prompt构成
	* 微调训练集：结合设计的prompt，为每一种error type构造若干个微调训练集（**同时包含是和否的数据**）
	* 微调模型：微调得到BinaryClassifier模型
	* 效果评估：验证fine-tuned BinaryClassifier 模型的效果
### 3.1 Prompt构成
* spider.classify_plus.fine_tuned_binary_classifier函数
* prompt构成有几个版本（根据未微调自动binary classifier的标注结果，不断迭代调整），具体见：[二分分类器_binary_classifier_prompt](二分分类器_binary_classifier_prompt.md)
* 对于每个实例的error type的判断，输出格式为
``` python
{"judgement": true, "explanation": "explanation string"}
```
### 3.2 微调数据集
* 针对每一个logic error type，构造BinaryClassifierLLM的微调数据集
* 微调主要目标：**需要微调以降低fp，未微调模型的回答很不合理，倾向于将正确答案未false的判断为true**
* 数据集文档如下：
	* Missing LIMIT Clause：【腾讯文档】MissingLIMITClause https://docs.qq.com/sheet/DRnRKS25MU3RnUXh1
		``` json
		"Missing LIMIT Clause": {  
		    "tp": 2,  
		    "fn": 1,  
		    "fp": 0,  
		    "tn": 16,  
		    "total": 19,  
		    "acc": 0.9473684210526315,  
		    "precision": 1.0,  
		    "recall": 0.6666666666666666,  
		    "false_alarm": 0.0,  
		    "miss_rate": 0.3333333333333333  
		}
		```
	* Schema Misinterpretation Error：本身效果已经很好了，考虑暂时不进行微调
		``` json
		"Schema Misinterpretation Error": {  
		    "tp": 15,  
		    "fn": 0,  
		    "fp": 2,  
		    "tn": 0,  
		    "total": 17,  
		    "acc": 0.8823529411764706,  
		    "precision": 0.8823529411764706,  
		    "recall": 1.0,  
		    "false_alarm": 1.0,  
		    "miss_rate": 0.0  
		}
		```
	* Column Selection Error：【腾讯文档】ColumnSelectionError https://docs.qq.com/sheet/DRnJQaGdXTE5LYm5X
		``` json
		"Column Selection Error": {  
		    "tp": 5,  
		    "fn": 8,  
		    "fp": 1,  
		    "tn": 16,  
		    "total": 30,  
		    "acc": 0.7,  
		    "precision": 0.8333333333333334,  
		    "recall": 0.38461538461538464,  
		    "false_alarm": 0.058823529411764705,  
		    "miss_rate": 0.6153846153846154  
		}
		```
	* Condition Logic Hallucination：
	* Aggregation Function Misuse：
	* OrderBy Misuse：
	* Join Logic Hallucination：

### 3.3 微调模型
* 微调使用的key：fine_tune_binary_classifier
### 3.4 效果评估
* spider.compare_auto_manual_binary_classifier_label函数
* 评估以下指标：对于所有的possible types的每一个possible type  
	- P：TP+FN（所有真实为true的）  
	- TP：possible type属于manual label types；并且possible type被binary classifier 判断为 trueFN：possible type属于manual label types；但是possible type被binary classifier 判断为 false  
	- N：FP+TN（所有真实为false的）  
	- FP：possible type不属于manual label types；但是possible type被binary classifier 判断为 trueTN：possible type不属于manual label types；并且possible type被binary classifier 判断为 false  
	- ACC=(TP+TN)/(TP+TN+FP+FN)  
	- precision=TP/(TP+FP)  
	- recall=TPR=TP/(TP+FN)=TP/P  
	- false alarm=FPR=FP/(FP+TN)=FP/N  
	- miss rate=FNR=FN/(TP+FN)=FN/P
### 3.5 结果
* 未微调自动标注模型的准确率统计
* 未微调自动标注模型的准确率conclusion：**召回率还可以，但误报极多**，符合前面提到的“未经微调的gpt幻觉现象较为严重（倾向于回答yes，即便给的例子不是）”的直观感受
	* 微调得到BinaryClassifierLLM的必要性：以 Join Logic Hallucination 类型，提问llm根据定义判断测试用例是否属于该类型，发现未经微调的gpt幻觉现象较为严重（倾向于回答yes，即便给的例子不是），具体提问记录可见链接 https://chatgpt.com/share/67ebac8f-0144-8002-b716-cc696695da5d 
		* 不存在当前分类器的logic error type，但是大模型分析出存在其他logic error，故将false判断为true
```json
"merge": {  
    "tp": 186,  
    "fn": 6,  
    "fp": 238,  
    "tn": 14,  
    "total": 222,  
    "acc": 0.45045045045045046,  
    "precision": 0.4386792452830189,  
    "recall": 0.96875,  
    "false_alarm": 0.9444444444444444,  
    "miss_rate": 0.03125  
}
```
* 微调后自动标注模型的准确率统计


## 1 Logic Error Type Enumeration

```
Operator Misuse
Missing LIMIT Clause
Join Logic Hallucination
Violating Value Specification
Schema Misinterpretation Error
Column Selection Error
Condition Logic Hallucination
Aggregation Function Misuse
Missing Distinct Error
OrderBy Misuse
GroupBy Misuse
Others
```
### 1.1 Operator Misuse（运算符误用）
定义：Operator Misuse发生在 SQL 语句中，涉及 比较运算符（Comparison Operators）、逻辑运算符（Logical Operators）或数学运算符（Arithmetic Operators） 的不当使用，使得查询逻辑偏离自然语言查询的正确含义。Operator Misuse 主要源于模型误解了自然语言查询的逻辑关系。

下面是几个示例：
1. **比较运算符误用**
    - **错误示例**（不等于 `!=` 误用）：  
        **NLQ**: _Find employees with a salary greater than 5000._  
        **错误 SQL**:
        ```sql
        SELECT * FROM employees WHERE salary != 5000;
        ```
        **正确 SQL**:
        ```sql
        SELECT * FROM employees WHERE salary > 5000;
        ```    
        **错误分析**: `!=` 代表“不等于”，但查询需求是“工资大于5000”。
2. **逻辑运算符误用**
    - **错误示例**（`AND` 误用）：  
        **NLQ**: _Find customers who are from "New York" or "Los Angeles"._  
        **错误 SQL**:
        ```sql
        SELECT * FROM customers WHERE city = 'New York' AND city = 'Los Angeles';
        ```
        **正确 SQL**:
        ```sql
        SELECT * FROM customers WHERE city = 'New York' OR city = 'Los Angeles';
        ```
        **错误分析**: `AND` 要求 `city` 同时等于 `'New York'` 和 `'Los Angeles'`，这是不可能的，应该用 `OR`。
3. **数学运算符误用**
    - **错误示例**（加法 `+` 误用）：  
        **NLQ**: _Find the total sales price of all items, including tax (10% increase)._  
        **错误 SQL**:
        ```sql
        SELECT SUM(price) + 0.1 FROM sales;
        ```
        **正确 SQL**:
        ```sql
        SELECT SUM(price * 1.1) FROM sales;
        ```
        **错误分析**: `+ 0.1` 只是对 `SUM(price)` 加 0.1，而不是增加 10%。
        
### 1.2 Missing LIMIT Clause（缺失的限制条件）
该错误通常与 `LIMIT` 语句的使用不当有关，可能由于 `LIMIT` 缺失（查询要求选取最大或最小值等等）、语法错误、不合理的参数、或者在不支持 `LIMIT` 的 SQL 语句中使用 `LIMIT`，导致查询逻辑错误。Mis Limit Condition主要表现为：
1. 遗漏 `LIMIT`（Missing Limit）：SQL 语句缺少 `LIMIT`，导致返回的结果集过大。
2. 误用 `LIMIT`（Incorrect Limit）：SQL 语句错误地使用 `LIMIT`，导致查询返回的结果数与预期不符。
3. 误用 `OFFSET`（Incorrect Offset）：SQL 语句错误地使用 `OFFSET`，导致查询结果的起始位置偏移错误。

下面是几个例子：
1. 缺少 `LIMIT`（结果过大）
	* **问题**：没有使用 `LIMIT`，导致查询返回了所有记录，而用户可能只想要前 N 条。
	- **NLQ**: _Find the top 10 highest-paid employees._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM employees ORDER BY salary DESC;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM employees ORDER BY salary DESC LIMIT 10;
	    ```
	- **错误分析**：
	    - SQL 语句缺少 `LIMIT 10`，会返回所有员工，而不是前 10 名。
2. `LIMIT` 误用（错误的数量限制）
	* **问题**：错误地设置 `LIMIT` 的值，导致查询结果不符合要求。
	- **NLQ**: _Find the top 5 most expensive products._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM products ORDER BY price DESC LIMIT 50;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM products ORDER BY price DESC LIMIT 5;
	    ```
	- **错误分析**：
	    - `LIMIT 50` 而不是 `LIMIT 5`，导致返回的数据比预期的多。

3. `OFFSET` 误用（起始位置错误）
	* **问题**：错误地使用 `OFFSET`，导致查询跳过了不应跳过的数据。
	- **NLQ**: _Find the 11th to 20th highest-paid employees._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM employees ORDER BY salary DESC LIMIT 10;
	    ``` 
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM employees ORDER BY salary DESC LIMIT 10 OFFSET 10;
	    ```
	- **错误分析**：
	    - `LIMIT 10` 只返回前 10 名，没有 `OFFSET 10`，无法获得 11~20 名的员工。
4. `LIMIT` 与 `ORDER BY` 搭配错误
	* **问题**：缺少 `ORDER BY`，导致 `LIMIT` 选择的记录顺序错误。
	- **NLQ**: _Find the 3 most recently hired employees._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM employees LIMIT 3;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM employees ORDER BY hire_date DESC LIMIT 3;
	    ```
	- **错误分析**：
	    - 缺少 `ORDER BY hire_date DESC`，导致 SQL 可能随机返回 3 名员工，而不是最新入职的 3 人。

### 1.3 Join Logic Hallucination（连接逻辑幻觉）
定义：Join Logic Hallucination 指的是模型在生成 SQL 查询时对表连接（JOIN）逻辑的错误推理，导致查询结构不合理、连接方式错误或查询结果偏离预期。这种错误通常发生在模型错误地创建、遗漏或误用表连接，具体表现包括：
1. 多余的 JOIN（幻觉 JOIN）：模型连接了本不需要的表，可能导致数据重复或查询变慢。模型过度解读输入文本中的语义，在无需连接表时强行添加JOIN操作。  
   示例：用户问"统计销售额"，模型错误连接了与问题无关的`用户表`和`商品表`。
2. 缺失的 JOIN（遗漏 JOIN）：模型未能识别文本中隐含的多表关联需求，导致生成的SQL缺少关键JOIN。  
   示例：用户问"北京客户的订单详情"，模型仅查询`订单表`却未JOIN`客户表`以筛选"北京"客户。
3. 错误的连接条件：模型对ON子句的关联键选择错误（如用`user.id`连接`order.name`）。  
   示例：用户问"员工和其部门信息"，模型用`employee.age = department.id`错误关联。
4. 错误的连接类型：INNER JOIN、LEFT JOIN、RIGHT JOIN、FULL JOIN 误用。

下面是一些例子：
1. 多余的 JOIN（幻觉 JOIN）
	* **问题**：模型错误地连接了一个不相关的表，导致数据重复或错误。
	- **NLQ**: _Find all employees in the IT department._
	- **错误 SQL**:
	    ```sql
	    SELECT employees.name 
	    FROM employees 
	    JOIN projects ON employees.id = projects.employee_id 
	    WHERE employees.department = 'IT';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name 
	    FROM employees 
	    WHERE department = 'IT';
	    ```
	- **错误分析**：
	    - `projects` 表与 `employees` 表无关，不需要 JOIN。
	    - 多余的 JOIN 可能导致结果行数增加（如员工在多个项目中，导致多行重复）。
2. 缺失的 JOIN（遗漏 JOIN）
	* **问题**：查询需要的数据分布在多个表中，但模型没有正确连接它们，导致查询失败或返回错误结果。
	- **NLQ**: _Find the department name of employee ‘Alice’._
	- **错误 SQL**:
	    ```sql
	    SELECT department_name 
	    FROM departments 
	    WHERE employee_name = 'Alice';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT departments.department_name 
	    FROM employees 
	    JOIN departments ON employees.department_id = departments.id 
	    WHERE employees.name = 'Alice';
	    ```
	- **错误分析**：
	    - `departments` 表没有 `employee_name` 字段，查询会失败。
	    - 需要通过 `employees.department_id` 连接 `departments`。
3. 错误的连接条件
	* **问题**：`ON` 子句的条件错误，导致数据匹配错误或查询结果为空。
	- **NLQ**: _Find all employees and their department names._
	- **错误 SQL**:
	    ```sql
	    SELECT employees.name, departments.department_name 
	    FROM employees 
	    JOIN departments ON employees.id = departments.id;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT employees.name, departments.department_name 
	    FROM employees 
	    JOIN departments ON employees.department_id = departments.id;
	    ```
	- **错误分析**：
	    - `employees.id` 是员工 ID，而 `departments.id` 是部门 ID，二者无关，错误的连接条件会导致匹配失败或数据错误。
4. 错误的 JOIN 类型
	* **问题**：错误使用 INNER JOIN 或 LEFT JOIN，导致数据丢失或冗余。
	- **NLQ**: _List all departments and the number of employees in each._
	- **错误 SQL**:
	    ```sql
	    SELECT departments.name, COUNT(employees.id) 
	    FROM employees 
	    INNER JOIN departments ON employees.department_id = departments.id 
	    GROUP BY departments.name;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT departments.name, COUNT(employees.id) 
	    FROM departments 
	    LEFT JOIN employees ON employees.department_id = departments.id 
	    GROUP BY departments.name;
	    ```
	- **错误分析**：
	    - `INNER JOIN` 只返回有员工的部门，而没有员工的部门不会出现在结果中。
	    - `LEFT JOIN` 才能确保所有部门都被列出，即使没有员工。


### 1.4 Violating Value Specification（违反值规范）
定义：指 Text-to-SQL 模型在生成 SQL 查询时，对字段值的格式、语义或实际存储内容理解错误，导致生成的查询条件值与数据库实际存储值不一致，从而返回空结果或错误结果。其本质是模型未能准确理解自然语言描述与数据库字段值之间的映射关系。这种错误通常发生在 WHERE 子句、INSERT 语句或 GROUP BY 条件中，可能会导致：
1. 值格式错误 
	   - 示例：用户要求查询 `gender_code = 'Female'`，但模型生成 `gender_code = 'F'`，而数据库中实际存储的是完整单词 `'Female'` 而非缩写 `'F'`。  
2. 值语义偏差  
	   - 示例：用户要求查询 `order_status_code = '已发货'`，但模型生成 `order_status_code = 'SHIPPED'`（假设为英文代码）。
3.  数据库存储值与查询值不匹配
	   - 示例：用户要求查询 `country = 'USA'`，但数据库中实际存储为小写 `'usa'`，而模型未统一大小写处理。  
4. 值范围错误：查询值超出数据库字段存储的实际范围，导致查询无效。 
5. 多值混淆  
	   - 示例：用户要求查询 `payment_method_code = '信用卡'`，但数据库中实际存储为 `'CREDIT_CARD'`，导致条件不匹配。

下面是一些例子：
1. 值格式错误
	* **问题**：SQL 语句中使用的值格式与数据库存储格式不匹配，导致查询无法返回正确的结果。
	- **NLQ**: _Find all orders placed on January 1, 2023._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM orders WHERE order_date = '01-01-2023';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM orders WHERE order_date = '2023-01-01';
	    ```
	- **错误分析**：
	    - 数据库中的 `order_date` 可能存储为 `YYYY-MM-DD` 格式，而 SQL 查询中错误地使用了 `MM-DD-YYYY` 格式，导致查询结果为空。
2. 值范围错误
	* **问题**：查询值超出数据库字段存储的实际范围，导致查询无效。
	- **NLQ**: _Find employees who earn more than 1,000,000 per month._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM employees WHERE monthly_salary > 1000000;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM employees WHERE annual_salary > 12000000;
	    ```
	- **错误分析**：
	    - 数据库可能存储的是 **年薪**（annual_salary），而查询错误地将 **月薪** 直接用于比较，导致查询不到正确的数据。
3. 数据存储值与查询值不匹配
	* **问题**：数据库存储的值与查询中使用的值存在不同的命名规范或缩写，导致查询不到正确数据。
	- **NLQ**: _Find all customers from the United States._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM customers WHERE country = 'United States';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM customers WHERE country IN ('USA', 'United States', 'US');
	    ```
	- **错误分析**：
	    - 数据库可能存储的是 `'USA'` 而不是 `'United States'`，导致查询不到数据。
	    - 解决方案是考虑所有可能的存储格式，如 `IN ('USA', 'United States', 'US')`。
### 1.5 Schema Misinterpretation Error（模式误解错误）
定义：Schema Misinterpretation Error（模式误解错误）是 Text-to-SQL 任务中的一种逻辑错误，指的是 模型在解析数据库模式（Schema）时，错误地理解表、列、主键、外键、数据类型或表之间的关系，导致生成的 SQL 查询结构错误或查询结果不符合预期。这种错误通常表现为以下几种情况：
1. 错误的表引用：使用了不存在或不相关的表。
2. 错误的列引用：选择了不存在或错误的列名。
3. 误解表之间的关系：模型错误地推断了表之间的关系，导致 `JOIN` 方式错误，使用了错误的外键，或者查询结构不合理。
4. 误解数据类型：模型在 SQL 语句中使用了与数据库字段 数据类型不匹配的值，可能导致错误的查询结果或 SQL 语法错误。
5. 误解主键/外键约束：错误地使用外键查询，导致数据不完整或重复。


下面是一些例子：
1. 错误的表引用
	- **NLQ**: _Find all orders placed by customer John Doe._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM transactions WHERE customer_name = 'John Doe';
	    ```
	- **正确 SQL**
	    ```sql
	    SELECT * FROM orders WHERE customer_id = 
	    (SELECT customer_id FROM customers WHERE name = 'John Doe');
	    ```
	* 错误分析
		- `transactions` 可能是用于存储付款信息的表，而 `orders` 才是存储订单的正确表。
		- 由于 `orders` 可能不直接存储 `customer_name`，需要 `JOIN` 或子查询从 `customers` 表获取 `customer_id`。

2. 错误的列引用
	- **NLQ**: _Find the name of the employee with ID 1001._
	- **错误 SQL**:
	    ```sql
	    SELECT full_name FROM employees WHERE id = 1001;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name FROM employees WHERE employee_id = 1001;
	    ```
	* 错误分析
		- `employees` 表中可能没有 `full_name`，正确的列可能是 `name` 或 `employee_name`。
		- `id` 可能不是 `employees` 表的主键，正确的列可能是 `employee_id`。
3. 误解表之间的关系
	- **NLQ**: _List all orders along with the names of the customers who placed them._
	- **错误 SQL**:
	    ```sql
	    SELECT orders.id, customers.name 
	    FROM orders, customers 
	    WHERE orders.customer_id = customers.id;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT orders.id, customers.name 
	    FROM orders 
	    JOIN customers ON orders.customer_id = customers.customer_id;
	    ```
	* 错误分析
		- `customers.id` 可能是错误的列，正确的外键是 `customers.customer_id`。
		- 使用 `,` 进行隐式连接可能导致 **笛卡尔积**，而 `JOIN` 才能正确连接表。
4. 误解数据类型
	- **NLQ**: _Find all products cheaper than $100._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM products WHERE price < '100';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM products WHERE price < 100;
	    ```
	* 错误分析
		- `price` 可能是 `DECIMAL` 或 `FLOAT` 类型，而 `'100'` 是字符串，可能导致 SQL 隐式转换问题。
		- 直接使用数值 `100` 进行比较能避免数据类型错误。
5. 误解主键/外键约束
- **NLQ**: _Find all employees who have managed at least one project._
- **错误 SQL**:
    ```sql
    SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL;
    ```
- **正确 SQL**:
    ```sql
    SELECT DISTINCT employees.name 
    FROM employees 
    JOIN projects ON employees.employee_id = projects.manager_id;
    ```
* 错误分析
	- `employees` 表可能没有 `manager_id`，而 `projects` 表存储了项目经理的 `manager_id`。
	- 需要 `JOIN` `projects` 表以正确查找管理过项目的员工。

### 1.6 Column Selection Error（列选择错误）
定义：Column Selection Error指的是 SQL 查询在 `SELECT` 子句或查询条件中，错误地选择了列、遗漏列、选择多余列，导致查询结果不符合用户意图或查询逻辑不成立。常见的类型包括：
1. Extraneous Column Selection Error（冗余列选择错误)：SQL 查询中包含了不必要的额外列，使查询返回过多信息或导致逻辑错误。
2. Missing Column Selection Error（缺失列选择错误）：SQL 查询中遗漏了必须的列，使查询无法正确返回用户需要的信息。
3. Incorrect Column Selection Error（错误列选择错误）：SQL 查询包含了错误的列，导致查询逻辑不符合用户意图或返回错误数据。

下面是一些示例：
1. SQL 查询中 **额外包含了不必要的列**，这些列不符合查询意图，可能导致：
	- **NLQ**: _Find the names of all employees who work in the Sales department._
	- **错误 SQL**:
	    ```sql
	    SELECT employee_id, name, salary FROM employees WHERE department = 'Sales';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name FROM employees WHERE department = 'Sales';
	    ```
	* 错误分析
		- `employee_id` 和 `salary` 在用户的问题中 **没有被请求**，但错误查询中仍然选择了这些列。
		- 这些额外列可能会影响后续的数据处理，例如 `DISTINCT` 可能因 `employee_id` 变化而失效。
2. Missing Column Selection Error（缺失列选择错误）
	- **NLQ**: _Find the names and salaries of all employees in the Sales department._
	- **错误 SQL**:
	    ```sql
	    SELECT name FROM employees WHERE department = 'Sales';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name, salary FROM employees WHERE department = 'Sales';
	    ```
	* 错误分析
		- `salary` 是查询所需的信息，但错误 SQL **遗漏了 `salary` 列**，使得查询返回的结果不完整。
3. Incorrect Column Selection Error（错误列选择错误）
	- **NLQ**: _Find the names of customers who placed orders worth more than $500._
	- **错误 SQL**:
	    ```sql
	    SELECT customer_name FROM customers WHERE balance > 500;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT customers.name 
	    FROM customers 
	    JOIN orders ON customers.customer_id = orders.customer_id 
	    WHERE orders.total_amount > 500;
	    ```
	* 错误分析**
		- `balance` 可能指的是客户账户余额，而用户问题要求的是 **订单总金额**，所以 `orders.total_amount` 才是正确的列。
		- 由于 `customer_name` 可能存储在 `customers` 表，而订单金额在 `orders` 表，查询需要 `JOIN` 两个表。


### 1.7 Condition Logic Hallucination（条件逻辑幻觉）
指在自然语言到 SQL 查询的转换过程中，模型错误地生成与原始文本无关、覆盖不完全、覆盖超出或矛盾的条件逻辑。这种错误表现为：
1. 无中生有条件：模型添加了用户未提及的过滤条件。  
   *例*：用户要求 "显示销售额超过 1 万的订单"，模型生成 `WHERE amount > 10000 AND region = 'East'`（凭空添加 `region` 限制）。  
2. 条件逻辑矛盾：生成的条件与用户意图冲突。  
   *例*：用户要求 "排除已取消的订单"，模型生成 `WHERE status = 'cancelled'`（逻辑反向）。
3. 条件覆盖不完全（Incomplete Condition Filtering）：指的是 SQL 查询中的筛选条件没有正确覆盖所有可能的情况，导致部分不符合预期的结果被包含或排除。常见于：



下面是一些例子：
1. 无中生有条件（Spurious Condition Generation）
	- **NLQ**: _Find all orders where the total amount exceeds $10,000._
	- **错误 SQL**:
	    ```sql
	    SELECT * FROM orders WHERE amount > 10000 AND region = 'East';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM orders WHERE amount > 10000;
	    ```
	* 错误分析
		- 用户 **没有要求按 `region` 过滤**，但 SQL 查询错误地添加了 `AND region = 'East'`，导致只返回东部地区的订单。
		- 这种错误可能导致 **查询结果缺失** 或 **业务决策错误**。
2. 条件逻辑矛盾（Contradictory Condition Logic）
	- **NLQ**: _Exclude all cancelled orders._
	- **错误 SQL**: 
	    ```sql
	    SELECT * FROM orders WHERE status = 'cancelled';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT * FROM orders WHERE status != 'cancelled';
	    ```
	* 错误分析
		- 生成的 SQL 逻辑 **与用户意图相反**，导致 **仅返回被取消的订单**，而不是排除它们。
3. 条件覆盖不完全（Incomplete Condition Filtering）
	- **NLQ**: _Find cities where the maximum temperature never exceeded 70°F._
	- **错误 SQL**:
	    ```sql
	    SELECT DISTINCT city FROM weather WHERE max_temperature_f < 70;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT DISTINCT city FROM weather 
	    EXCEPT 
	    SELECT DISTINCT city FROM weather WHERE max_temperature_f >= 70;
	    ```
	* 错误分析
		- **错误 SQL**：查询仅包含某些天气记录温度低于 70°F 的城市，但无法保证 **该城市在所有记录中都符合要求**。
		- **正确 SQL**：使用 `EXCEPT` 先筛选出 **任何一天高于 70°F 的城市**，然后排除它们，确保剩下的城市符合要求。

4. 条件覆盖不完全（Incomplete Condition Filtering）
- **NLQ**: _Find customers who have placed an order every day for the past 7 days._
- **错误 SQL**:
    ```sql
    SELECT DISTINCT customer_id FROM orders 
    WHERE order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY);
    ```
- **正确 SQL**:
    
    ```sql
    SELECT customer_id FROM orders 
    WHERE order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 7 DAY)
    GROUP BY customer_id
    HAVING COUNT(DISTINCT order_date) = 7;
    ```
* 错误分析
	- **错误 SQL**：仅筛选了最近 7 天内下单的客户，但没有保证 **每天都有订单**。
	- **正确 SQL**：使用 `HAVING COUNT(DISTINCT order_date) = 7` 确保 **过去 7 天每天都有订单**。

### 1.8 Aggregation Function Misuse（聚合函数误用）
定义：聚合函数误用（Aggregation Function Misuse）指的是 SQL 查询中错误地使用了聚合函数（如 `SUM()`、`COUNT()`、`AVG()`、`MAX()`、`MIN()`），导致查询结果不符合预期。具体来说，可能的错误包括：
- 误用聚合函数（Incorrect Aggregation Usage）：选择了错误的聚合函数，例如使用 `COUNT(*)` 统计行数，而正确的做法是 `SUM(order_quantity)` 计算总数。
- 缺失聚合函数（Missing Aggregation Function）
- 对错误的数据列应用了聚合函数，导致查询逻辑错误。
- 聚合与非聚合混用错误（Aggregation and Non-Aggregation Mixing Error）：未正确使用 `GROUP BY`，导致数据聚合方式不正确。

下面是一些例子：
1. 误用聚合函数（Incorrect Aggregation Usage）
	- **NLQ**: _Find the average salary of employees._
	- **错误 SQL**:
	    ```sql
	    SELECT SUM(salary) FROM employees;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT AVG(salary) FROM employees;
	    ```
	* 错误分析
		- 用户需要的是 **平均工资**，但模型错误地使用了 `SUM(salary)`，返回的是总工资，而不是平均值。
2. 缺失聚合函数（Missing Aggregation Function）
	- **NLQ**: _Find the total sales for each product._
	- **错误 SQL**:
	    ```sql
	    SELECT product_id, sales FROM orders GROUP BY product_id;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT product_id, SUM(sales) FROM orders GROUP BY product_id;
	    ```
	* 错误分析
		- **错误 SQL** 缺少 `SUM(sales)`，导致 `sales` 只是单个订单的销售额，而不是该产品的总销售额。
		- **正确 SQL** 使用 `SUM(sales)` 计算按 `product_id` 分组的总销售额。
3. 聚合与非聚合混用错误（Aggregation and Non-Aggregation Mixing Error）
	- **NLQ**: _Find the highest salary and the employee who earns it._
	- **错误 SQL**:
	    ```sql
	    SELECT name, MAX(salary) FROM employees;
	    ```
	- **正确 SQL**:
	    ```sql
		SELECT name, salary 
		FROM employees
		GROUP BY name, salary
		HAVING salary = MAX(salary);
	    ```
	* 错误分析
		- **错误 SQL** 试图在 `SELECT` 中同时使用 `name`（非聚合列）和 `MAX(salary)`（聚合列），但没有 `GROUP BY`，导致 SQL 语法错误。
### 1.9 Missing Distinct Error（缺失去重错误）

定义：指 SQL 查询缺少 `DISTINCT` 关键字，导致查询结果包含重复数据，而用户实际期望的是唯一值。该错误通常出现在查询要求 唯一实体（如不同类别、唯一名称等）时，但 SQL 查询未正确去重，从而返回了冗余行。

下面是一些例子：
1. 查询唯一值但未去重
	- **NLQ**: _Find all unique job titles in the company._
	- **错误 SQL**:
	    ```sql
	    SELECT job_title FROM employees;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT DISTINCT job_title FROM employees;
	    ```
	* 错误分析
		- **错误 SQL** 返回了 `employees` 表中所有员工的 `job_title`，如果多个员工的职位相同，就会出现重复行。
		- **正确 SQL** 通过 `DISTINCT` 关键字去重，确保结果中每种职位只出现一次。
2. 统计不同用户数但未去重
	- **NLQ**: _Count the number of different customers who made purchases._
	- **错误 SQL**:
	    ```sql
	    SELECT COUNT(customer_id) FROM orders;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT COUNT(DISTINCT customer_id) FROM orders;
	    ```
	* 错误分析
		- **错误 SQL** 计算的是 `orders` 表中的所有 `customer_id`，如果一个客户有多个订单，就会被重复计数。
		- **正确 SQL** 使用 `COUNT(DISTINCT customer_id)`，确保每个客户只被计数一次。
3. 连接多个表时未正确去重
	- **NLQ**: _List all distinct product categories sold in 2023._
	- **错误 SQL**:
	    ```sql
	    SELECT category_name 
	    FROM products 
	    JOIN orders ON products.product_id = orders.product_id
	    WHERE orders.order_date >= '2023-01-01';
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT DISTINCT category_name 
	    FROM products 
	    JOIN orders ON products.product_id = orders.product_id
	    WHERE orders.order_date >= '2023-01-01';
	    ```
	* 错误分析
		- **错误 SQL** 没有 `DISTINCT`，如果同一产品类别的商品被多次购买，会导致类别重复出现。
		- **正确 SQL** 通过 `DISTINCT` 确保结果集中每个类别只出现一次。
### 1.10 OrderBy Error(排序错误)
定义：`OrderBy Error` 指 SQL 查询中的 `ORDER BY` 语句未能正确执行排序，导致查询结果的顺序与用户意图不符。这种错误通常由以下几种情况引起：
1. 缺失 `ORDER BY`：用户希望查询结果按某个字段排序，但 SQL 语句未包含 `ORDER BY`，返回的结果顺序是随机或默认的。
2. 错误的排序字段：`ORDER BY` 使用了错误的列，导致排序逻辑不符合用户意图。
3. 排序方向错误：ASC（升序）与 DESC（降序）方向与用户期望相反。
4. 多重排序逻辑错误：涉及多个 `ORDER BY` 字段时，字段的排列顺序不正确，影响最终的排序结果。

下面是一些例子：
1. 缺失 `ORDER BY`
	- **NLQ**: _Find the top 5 highest-paid employees._
	- **错误 SQL**:
	    ```sql
	    SELECT name, salary FROM employees LIMIT 5;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5;
	    ```
	* **错误分析**
		- **错误 SQL** 仅使用 `LIMIT 5`，但未排序，返回的可能是任意 5 名员工，而不是薪资最高的 5 人。
		- **正确 SQL** 使用 `ORDER BY salary DESC` 确保薪资最高的 5 名员工被返回。
2. 错误的排序字段
	- **NLQ**: _List all products from highest to lowest price._
	- **错误 SQL**:
	    ```sql
	    SELECT product_name, price FROM products ORDER BY quantity DESC;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT product_name, price FROM products ORDER BY price DESC;
	    ```
	*  **错误分析**
		- **错误 SQL** 按 `quantity` 进行排序，而用户期望的是按 `price` 排序。
		- **正确 SQL** 改用 `ORDER BY price DESC`，确保产品按照价格从高到低排列。
3. 排序方向错误
	- **NLQ**: _Show the 10 newest orders._
	- **错误 SQL**:
	    ```sql
	    SELECT order_id, order_date FROM orders ORDER BY order_date ASC LIMIT 10;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT order_id, order_date FROM orders ORDER BY order_date DESC LIMIT 10;
	    ```
	* **错误分析**
		- **错误 SQL** 采用 `ASC`（升序），返回的是最早的 10 个订单，而用户希望查看的是最新订单。
		- **正确 SQL** 采用 `DESC`（降序），确保最新订单排在前面。
4. 多重排序逻辑错误
	- **NLQ**: _Sort employees by department, then by salary from highest to lowest._
	- **错误 SQL**:
	    ```sql
	    SELECT name, department, salary FROM employees ORDER BY salary DESC, department ASC;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT name, department, salary FROM employees ORDER BY department ASC, salary DESC;
	    ```
	* **错误分析**
		- **错误 SQL** 先按 `salary DESC` 排序，再按 `department ASC` 排序，导致不同部门的员工按薪资乱序排列。
		- **正确 SQL** 先按 `department` 升序，再按 `salary` 降序，确保同一部门的员工按薪资从高到低排列。
### 1.11 GroupBy Misuse(GroupBy 误用)

定义：`GroupBy Misuse` 指 SQL 查询在使用 `GROUP BY` 进行分组时，未能正确匹配查询需求，导致错误的聚合行为或查询结果不符合用户期望。这种错误通常包括以下几种类型：
1. 缺失 `GROUP BY`：当查询需要按某列进行分组但 SQL 语句未使用 `GROUP BY`，导致错误的汇总结果。
2. 错误的分组列：`GROUP BY` 使用了错误的列，导致数据被错误地聚合。
3. 聚合列未正确使用：`SELECT` 语句中包含未聚合的列，且这些列未在 `GROUP BY` 中，导致 SQL 语法错误或错误的查询逻辑。
4. 不必要的 `GROUP BY`：在不需要分组的查询中错误地使用 `GROUP BY`，导致意外的结果。


下面是一些例子：
1. 缺失 `GROUP BY`
	- **NLQ**: _Find the total salary for each department._
	- **错误 SQL**:
	    ```sql
	    SELECT department, SUM(salary) FROM employees;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT department, SUM(salary) FROM employees GROUP BY department;
	    ```
	* **错误分析**
		- **错误 SQL** 试图计算每个部门的总工资，但由于缺少 `GROUP BY department`，会导致 SQL 语法错误（在某些 SQL 方言中）。
		- **正确 SQL** 添加了 `GROUP BY department`，确保 `SUM(salary)` 正确按部门分组。

2. 错误的分组列
	- **NLQ**: _Find the total number of orders for each customer._
	- **错误 SQL**:
	    ```sql
	    SELECT product_id, COUNT(order_id) FROM orders GROUP BY product_id;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT customer_id, COUNT(order_id) FROM orders GROUP BY customer_id;
	    ```
	* **错误分析**
		- **错误 SQL** 按 `product_id` 分组，而用户实际需要的是按 `customer_id` 统计订单数量。
		- **正确 SQL** 修改 `GROUP BY` 列，确保订单数量是按客户而非产品计算的。
3. 聚合列未正确使用
	- **NLQ**: _Find the highest salary in each department._
	- **错误 SQL**:
	    ```sql
	    SELECT department, name, MAX(salary) FROM employees GROUP BY department;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT department, name, salary 
	    FROM employees 
	    WHERE (department, salary) IN (
	        SELECT department, MAX(salary) 
	        FROM employees 
	        GROUP BY department
	    );
	    ```
	* **错误分析**
		- **错误 SQL** 在 `SELECT` 语句中包含 `name`，但 `name` 既不在 `GROUP BY` 里，也没有被聚合，导致 SQL 语法错误（在大多数 SQL 方言中）。
		- **正确 SQL** 需要使用子查询，确保 `name` 关联到最高工资的员工。
4. 不必要的 `GROUP BY`
	- **NLQ**: _Find the total number of orders._
	- **错误 SQL**:
	    ```sql
	    SELECT COUNT(order_id) FROM orders GROUP BY order_date;
	    ```
	- **正确 SQL**:
	    ```sql
	    SELECT COUNT(order_id) FROM orders;
	    ```
	* **错误分析**
		- **错误 SQL** 在查询总订单数量时，错误地按 `order_date` 进行分组，导致返回的是每天的订单数，而非总订单数。
		- **正确 SQL** 移除 `GROUP BY`，直接计算所有订单的总数。

### 1.12 Others





