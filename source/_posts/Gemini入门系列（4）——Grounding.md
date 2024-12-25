---
title: Gemini入门系列（4）——Grounding
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 8670afbc
date: 2024-12-25 20:33:28
---
&emsp;&emsp;如果把AI看作一个学生，那么有个最大的缺点，就是只掌握了学习过的知识（Trainning的材料），如果问AI一些知识范围外的问题，或者具有实时性的问题，AI无法回答。
<!-- more -->
&emsp;&emsp;例如，问AI当天的天气情况，AI不能给出答案：
![](https://home.aubreyhan.net:9000/blogpic/17332250771581.jpg)

&emsp;&emsp;但是，我们可以给AI接入一个搜索引擎，让AI可以自主去网上搜索相关知识，即AI的Grounding功能。
# 基于google search的Grounding
&emsp;&emsp;GCP直接提供基于google search的Grounding，可以在console直接启用，也可以在code中使用。
## console
&emsp;&emsp;Console使用非常简单，在右边工具栏打开google search开关即可：
![](https://home.aubreyhan.net:9000/blogpic/17351230599090.jpg)

&emsp;&emsp;现在提问，AI就可以给出适时信息的回答，以及答案的来源：
![](https://home.aubreyhan.net:9000/blogpic/17351231620105.jpg)

## code
&emsp;&emsp;Sample Code如下：
```
import base64
import vertexai
from vertexai.preview.generative_models import GenerativeModel, Part, SafetySetting, Tool
from vertexai.preview.generative_models import grounding


def generate():
    vertexai.init(project="ai-demo-440003", location="us-central1")
    model = GenerativeModel(
        "gemini-1.5-flash-002",
        tools=tools,
    )
    responses = model.generate_content(
        ["""今天北京的天气怎么样"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
        if not response.candidates or not response.candidates[0].content.parts:
            continue
        print(response.text, end="")


generation_config = {
    "max_output_tokens": 8192,
    "temperature": 1,
    "top_p": 0.95,
}

safety_settings = [
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HATE_SPEECH,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
]

tools = [
    Tool.from_google_search_retrieval(
        google_search_retrieval=grounding.GoogleSearchRetrieval()
    ),
]

generate()
```
&emsp;&emsp;Grounding相关的code如下：
1. 定义一个tools，内容为Grounding
```
tools = [
    Tool.from_google_search_retrieval(
        google_search_retrieval=grounding.GoogleSearchRetrieval()
    ),
]
```
1. 在模型定义中除了基础模型，加上tools
```
model = GenerativeModel(
        "gemini-1.5-flash-002",
        tools=tools,
    )
```

# 自定义Grouding
&emsp;&emsp;有些场景下，我们希望AI能根据指定内容回答，把Grounding的范围封闭在一个自定义的数据集，而不是开放的搜索引擎（例如客服）。Gemini同样也提供此类型的Grounding。
## 建立数据集
&emsp;&emsp;Gemini支持多种数据集（包括结构化、非结构化数据，网站、app应用等）。本文以非结构化数据为例，示范一个简单的pdf文件（demo.pdf），内容为姓名和学号的对应：
![](https://home.aubreyhan.net:9000/blogpic/17351254971874.jpg)

&emsp;&emsp;将此文件上传到一个GCS的bucket：
![](https://home.aubreyhan.net:9000/blogpic/17351284473822.jpg)

&emsp;&emsp;进入Agent Builder，选择Data Store：
![](https://home.aubreyhan.net:9000/blogpic/17351284848781.jpg)

&emsp;&emsp;点击Create Data Store开始创建：
![](https://home.aubreyhan.net:9000/blogpic/17351285254988.jpg)

&emsp;&emsp;选择Cloud Storage，根据上传的数据类型做对应选择并定位到对应的bucket，如果数据会时常更新，可以选择Periodic方式（周期可选择1、3、5天），这里为了测试直接选择One Time：
![](https://home.aubreyhan.net:9000/blogpic/17351287309068.jpg)

&emsp;&emsp;最后选择区域和给Data Store命名即可：
![](https://home.aubreyhan.net:9000/blogpic/17351287786239.jpg)

&emsp;&emsp;完成创建如下，此处需要记下Location和ID共AI调用：
![](https://home.aubreyhan.net:9000/blogpic/17351288468117.jpg)

&emsp;&emsp;需要稍等后台处理数据（embedding），根据数据量会需要不同的时间。
&emsp;&emsp;
## 使用私有数据集进行Grounding
&emsp;&emsp;首先文档里有一处错误：
![](https://home.aubreyhan.net:9000/blogpic/17351290096361.jpg)

&emsp;&emsp;实际在console还没有整合此功能：
![](https://home.aubreyhan.net:9000/blogpic/17351290388268.jpg)

&emsp;&emsp;仍然需要通过code来实现，如下：
```
import base64
import vertexai
from vertexai.preview.generative_models import GenerativeModel, Part, SafetySetting, Tool
from vertexai.preview.generative_models import grounding


def generate():
    vertexai.init(project="ai-demo-440003", location="us-central1")
    model = GenerativeModel(
        "gemini-1.5-flash-002",
        tools=tools,
    )
    responses = model.generate_content(
        ["""李四的学号是多少"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
        if not response.candidates or not response.candidates[0].content.parts:
            continue
        print(response.text, end="")


generation_config = {
    "max_output_tokens": 8192,
    "temperature": 1,
    "top_p": 0.95,
}

safety_settings = [
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HATE_SPEECH,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=SafetySetting.HarmBlockThreshold.OFF
    ),
]

tools = [
    Tool.from_retrieval(
        grounding.Retrieval(
            grounding.VertexAISearch(
                datastore='grounding-ds-001_1735128749407',
                project="ai-demo-440003",
                location='global'           
            )
        )
    )
]

generate()
```
&emsp;&emsp;运行结果如下：
![](https://home.aubreyhan.net:9000/blogpic/17351292573046.jpg)

&emsp;&emsp;AI根据创建的数据集正确回答出问题。
&emsp;&emsp;对比使用google search的Grounding，可以看到主要的区别在tools的定义：
```
tools = [
    Tool.from_retrieval(
        grounding.Retrieval(
            grounding.VertexAISearch(
                datastore='grounding-ds-001_1735128749407',
                project="ai-demo-440003",
                location='global'           
            )
        )
    )
]
```
&emsp;&emsp;这里不再是直接引入google search，而是用自己创建的数据集替代，可以更自由的掌控AI生成的结果。

# 小结
&emsp;&emsp;通过Grounding可以让AI更加个性化，在一些特定场景，能帮用户打造自己的AI。Grounding的使用很简单，希望能有帮助
