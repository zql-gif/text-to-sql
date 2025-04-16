## version1.0

``` python

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
            f"Let's think step by step. 根据{error_type}的定义，判断下面提供的text to sql任务实例是否属于{error_type}，如果是则给出true，如果不是则给出false"  
            f"{error_type}定义如下：{logic_error_type_intro_lines[error_type]}\n"  
            "请严格按照下面json样例的格式输出答案，仅输出符合[]列表的json格式，不要输出任何额外内容："  
            "{\"judgement\": python boolean,true or false, \"explanation\": \"{{explanation string to answer why is or not}}\"}"  
            f"text to sql任务实例：{question_content}"  
        )  
    }  
]
```


## version2.0
``` python 
context_issue = [  
    {  
        "role": "system",  
        "content": "You are an expert in analyzing code issues and you will respond in Chinese."  
    },  
    {  
        "role": "user",  
        "content": (  
            f"Let's think step by step. 根据text to sql任务的logic error type：{error_type}的定义，判断下面提供的text to sql任务实例是否属于{error_type}这一类logic error type，如果是则给出true，如果不是则给出false"  
            f"{error_type}定义如下：{logic_error_type_intro_lines[error_type]}\n"  
            "请严格按照下面json样例的格式输出答案，仅输出符合[]列表的json格式，不要输出任何额外内容："  
            "{\"judgement\": python boolean,true or false, \"explanation\": \"{{explanation string to answer why is or not}}\"}"  
            f"text to sql任务实例：{question_content}"  
        )  
    }  
]
```

## version3.0
``` python
context_issue = [  
    {  
        "role": "system",  
        "content": "You are an expert in analyzing code issues and you will respond in Chinese."  
    },  
    {  
        "role": "user",  
        "content": (  
            f"Let's think step by step. "  
            f"根据text to sql任务的logic error type：{error_type}的定义，判断下面提供的text to sql任务实例中，‘LLM预测答案’相较于‘正确答案’是否存在{error_type}这一类text to sql logic error（不考虑其他类型的logic error type）。如果存在则给出true，如果不是则给出false"  
            f"{error_type}定义如下：{logic_error_type_intro_lines[error_type]}\n"  
            "请严格按照下面json样例的格式输出答案，仅输出符合[]列表的json格式，不要输出任何额外内容："  
            "{\"judgement\": python boolean,true or false, \"explanation\": \"{{explanation string to answer why is or not}}\"}"  
            f"text to sql任务实例：{question_content}"  
        )  
    }  
]
```
