## 1 标记任务描述
* 分析error type和root cause，表象可能是用错某个函数，根本原因是LLM对某个操作理解不当
* gpt-3.5-turbo_3.0：spider-train-merged-info https://docs.qq.com/sheet/DTG9La1VpbXd6TFhT?tab=000001
* gpt-3.5-turbo_2.0：spider-test-merged-info https://docs.qq.com/sheet/DTENnVHBZbHZDcHJ3?tab=000001
### 1.1 Error Type Enumeration
#### 1.1.1 Set Operator Misuse（集合运算符误用）

**这个错误通常出现在 SQL 查询中错误地使用了集合运算符（如 `UNION`、`INTERSECT`、`EXCEPT`）。** 可能的原因包括：

1. **列数不匹配**：集合运算符两侧的 `SELECT` 语句返回的列数不同，例如：
    
    ```sql
    SELECT id, name FROM users
    UNION
    SELECT age FROM employees; -- 错误：列数不匹配
    ```
    
2. **数据类型不兼容**：集合运算符要求对应列的数据类型必须兼容，否则会报错，例如：
    
    ```sql
    SELECT id FROM users
    UNION
    SELECT name FROM employees; -- 错误：id 可能是整数，而 name 是字符串
    ```
    
3. **错误的子查询使用方式**：某些 SQL 语法要求集合运算符不能直接用于子查询或者特定的 SQL 语句中，比如 `INSERT INTO` 时不能直接使用 `UNION` 等。
    
#### 1.1.2 Mis Limited Condition（缺失的限制条件）

该错误通常与 `LIMIT` 语句的使用不当有关，可能由于 `LIMIT` 缺失（查询要求选取最大或最小值）、语法错误、不合理的参数、或者在不支持 `LIMIT` 的 SQL 语句中使用 `LIMIT`，导致查询逻辑错误。

#### 1.1.3 Join Logic Hallucination（连接逻辑幻觉）
这是Text-to-SQL模型（将自然语言转化为SQL查询的AI模型）中一种典型的逻辑错误，指模型在生成SQL查询时对表连接（JOIN）逻辑的错误推理。具体表现为：
1. **错误添加JOIN**：  
   模型过度解读输入文本中的语义，在无需连接表时强行添加JOIN操作。  
   *示例*：用户问"统计销售额"，模型错误连接了与问题无关的`用户表`和`商品表`。
2. **遗漏必要JOIN**：  
   模型未能识别文本中隐含的多表关联需求，导致生成的SQL缺少关键JOIN。  
   *示例*：用户问"北京客户的订单详情"，模型仅查询`订单表`却未JOIN`客户表`以筛选"北京"客户。
3. **JOIN条件错误**：  
   模型对ON子句的关联键选择错误（如用`user.id`连接`order.name`）。  
   *示例*：**用户问"员工和其部门信息"，模型用`employee.age = department.id`错误关联。**
**本质原因**：  
模型未能准确理解自然语言中的实体关系，或对数据库模式（Schema）中的外键约束、表间关联缺乏精准映射能力。
#### 1.1.4 Violating Value Specification（违反值规范）

指 Text-to-SQL 模型在生成 SQL 查询时，对字段值的格式、语义或实际存储内容理解错误，导致生成的查询条件值与数据库实际存储值不一致，从而返回空结果或错误结果。
这是 Text-to-SQL 模型中的一种典型逻辑错误，表现为模型对字段值的具体存储形式（如性别代码、日期格式、分类标签等）做出错误假设。**其本质是模型未能准确理解自然语言描述与数据库字段值之间的映射关系。**
1. **值格式错误**  
   - 示例：用户要求查询 `gender_code = 'Female'`，但模型生成 `gender_code = 'F'`，而数据库中实际存储的是完整单词 `'Female'` 而非缩写 `'F'`。  
2. **值语义偏差**  
   - 示例：用户要求查询 `order_status_code = '已发货'`，但模型生成 `order_status_code = 'SHIPPED'`（假设为英文代码）。  
3. **大小写敏感性问题**  
   - 示例：用户要求查询 `country = 'USA'`，但数据库中实际存储为小写 `'usa'`，而模型未统一大小写处理。  
4. **多值混淆**  
   - 示例：用户要求查询 `payment_method_code = '信用卡'`，但数据库中实际存储为 `'CREDIT_CARD'`，导致条件不匹配。

#### 1.1.5 Schema Misinterpretation Error（模式误解错误）
**Schema Misinterpretation Error**（模式误解错误）是指 LLM 生成的 SQL 查询未能正确理解数据库模式（schema），导致：
1. 选择了错误的表或字段：例如gpt-3.5-turbo_2.0的100。
2. **误解了表与表之间的关系：例如gpt-3.5-turbo_2.0的101。**
3. 生成的 SQL 语义与问题要求不匹配。

#### 1.1.6 Extraneous Column (Selection) Error（多余列错误）

该错误表示 SQL 查询中包含了不必要的额外列，导致查询逻辑不成立。
#### 1.1.7 Missing Column Selection Error（缺失列错误）

该错误表示 SQL 查询中缺失了必要的列，导致查询逻辑不成立。
#### 1.1.8 Incorrect Column Selection Error（不正确列错误）

该错误表示 SQL 查询中包含了错误的，不符合question要求的额外列，导致查询逻辑不成立。
#### 1.1.9 Condition Logic Hallucination（条件逻辑幻觉）
指在自然语言到 SQL 查询的转换过程中，**模型错误地生成与原始文本无关、覆盖不完全、覆盖超出或矛盾的条件逻辑**。这种错误表现为：
1. **无中生有条件**：模型添加了用户未提及的过滤条件。  
   *例*：用户要求 "显示销售额超过 1 万的订单"，模型生成 `WHERE amount > 10000 AND region = 'East'`（凭空添加 `region` 限制）。  
2. **条件逻辑矛盾**：生成的条件与用户意图冲突。  
   *例*：用户要求 "排除已取消的订单"，模型生成 `WHERE status = 'cancelled'`（逻辑反向）。
3. **条件覆盖不完全（Incomplete Condition Filtering）**：指的是 SQL 查询中的筛选条件没有正确覆盖所有可能的情况，导致部分不符合预期的结果被包含或排除。常见于：
	- “至少有一天符合条件” vs. “所有天数都必须符合” 的混淆。
	- **“找出最高气温从未超过 70°F 的城市”** 🚫 (`WHERE max_temperature_f < 70`) ❌
	- **正确方式** ✅ (`EXCEPT SELECT DISTINCT city WHERE max_temperature_f >= 70`)
#### 1.1.10 Subquery logic error（子查询逻辑错误）

**子查询逻辑错误（Subquery Logic Error）** 指的是 SQL 语句在使用子查询（`SUBQUERY`）时，逻辑上不符合查询目标，导致错误的结果或空结果集。该错误通常发生在 **`IN`、`EXISTS`、`NOT IN`** 等子查询条件使用不当的情况下，具体表现为：
- **错误地筛选了不相关的数据**，导致返回的数据不正确。
- **遗漏应当包含的数据**，导致返回的数据为空（空集）。
- **子查询的粒度（granularity）与外层查询不匹配**，导致查询逻辑错误。


#### 1.1.11 Aggregation Function Misuse（聚合函数误用）


**聚合函数误用（Aggregation Function Misuse）** 指的是 SQL 查询中错误地使用了聚合函数（如 `SUM()`、`COUNT()`、`AVG()`、`MAX()`、`MIN()`），导致查询结果不符合预期。具体来说，可能的错误包括：
- **选择了错误的聚合函数**，例如使用 `COUNT(*)` 统计行数，而正确的做法是 `SUM(order_quantity)` 计算总数。
- **未正确使用 `GROUP BY`**，导致数据聚合方式不正确。
- **对错误的数据列应用了聚合函数**，导致查询逻辑错误。

#### 1.1.12 OrderBy Error(排序错误)
**误解question的意图或者未结合database schema、客观事实**，出现排序错误。

#### 1.1.13 GroupBy Misuse(GroupBy 误用)
在 Text-to-SQL 任务中，LLM 生成的 SQL 语句错误地使用了 `GROUP BY`，导致查询逻辑出现偏差。此错误通常发生在 `GROUP BY` 语句被误用于本不需要聚合的查询中，使得查询结果可能丢失数据或返回不符合预期的行。

**错误原因可能包括如下：**
1. **LLM 泛化过度**：模型可能在训练过程中学到「日期通常需要 `GROUP BY`」的模式，而在此查询中，该模式是不适用的。
2. **SQL 逻辑理解不足**：模型未能区分「需要聚合」和「仅需排序筛选」的场景。
3. **缺乏数据依赖分析**：LLM 可能没有充分理解 `weather` 表的模式，没有意识到 `date` 是唯一标识每日记录的字段。
#### 1.1.14 正确答案有误

## 2 特殊备注（主要是测试集本身的问题，gpt-3.5-turbo_3.0）

### 2.0 正确答案有误
spider数据集中有很多错误的gold sql。
### 2.1 一个问题可对应多个解

在44中，question为Please show the different statuses, ordered by the number of cities that have each.该问题未指明ordered是升序还是降序（大模型默认降序）。
### 2.2 database data 错误
在64中，question为List the id of students who attended some courses?
得到的predict sql逻辑上是正确的：
``` sql
SELECT student_id FROM Students WHERE student_id IN (     SELECT student_id     FROM Student_Course_Attendance );
```

对应的standard sql是：
``` sql
SELECT student_id FROM student_course_attendance
```

predict sql之所以被判断为错误的，问题出在database data上面，Student_Course_Attendance中的student_id是Students的student_id的超集（也就是说Student_Course_Attendance表中存在Students表里没有的id）。

### 2.3 同义question->sql
在标记过程中，发现了很多同义question组，其text to sql的predict sql结果大部分相同，logic error type也很相似。

### 2.4 无伤大雅的多余列
在190，192中，标准答案希望查询语句只返回station name, longitude and average duration of trips，但predict sql查询结果多返回了station id。这一logic error不算严重，question没有明确指明返回列时，这种回答也比较正确。


### 2.5 database schema的实时性数据查询
在277，278，285，289等等中，标准答案对于follower这种动态数据的查询需要将用户信息表user_profiles 和关注表follows进行join表连接（以防止user_profiles表中followers字段的不准确）。而predict sql普遍性忽略这一点，但是这一点有没有在database schema中特殊强调。