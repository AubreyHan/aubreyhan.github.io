---
title: Gemini入门系列（7）——GlobalEndpoint
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: bd1e66d7
date: 2025-02-19 18:25:01
---

&emsp;&emsp;随着Gemini-2.0-Flash的GA，一些新Feature也陆续公布。其中一个比较比较有用的就是Global Endpoint。
<!-- more -->
&emsp;&emsp;如大家所知，Gemini使用时需要指定Region，如果该Region资源紧张，就会出现429错误。要想使用多个Region，只能在code里轮训，使用起来比较麻烦。现在Gloabl EndPoint推出后，直接使用Global Endpoint，无需客户端的复杂配置。
&emsp;&emsp;目前支持Global Endpoint的模型有：
&emsp;&emsp;* Gemini-1.5-Flash-002
&emsp;&emsp;* Gemini-1.5-Pro-002
&emsp;&emsp;* Gemini-2.0-Flash-001

# 在Console使用Global Endpoint
&emsp;&emsp;在Console使用Global Endpoint很简单，直接在Region选择的地方选择Gloabl即可，以Gemini-1.5-Flash-002，如下选择：
![](https://home.aubreyhan.net:9000/blogpic/17399616960353.jpg)

&emsp;&emsp;
&emsp;&emsp;如果模型选择Gemini-2.0-Flash-001，看不到该选项，但可以通过code里location的设置实现。

# 在Ccode里使用Global Endpoint
&emsp;&emsp;code的修改很简单，只需要把location设置为global即可，如下：
```
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"), 
                      vertexai=True, 
                      project="ai-demo-440003", 
                      location="global")
response = client.models.generate_content(
    model="gemini-2.0-flash-001",
    contents="How does AI work?",
)
print(response.text)
```
&emsp;&emsp;运行结果：
![](https://home.aubreyhan.net:9000/blogpic/17399631148985.jpg)

&emsp;&emsp;可以正确运行。
# 小结
&emsp;&emsp;Global Endpoint集中了全球所有Region资源，在一定程度上降低了429的出错概率，也无需客户在code里实现Region轮询，大大简化了code编写。
&emsp;&emsp;目前，Global Endpoint还在Public Preview阶段，所以只支持3个模型（测试版模型不支持），另外如下功能目前不支持：
&emsp;&emsp;* Tuning
&emsp;&emsp;* Batch
&emsp;&emsp;* Context Caching
&emsp;&emsp;* RAG
&emsp;&emsp;* DRZ/VPC-SC/ML-Processing
&emsp;&emsp;* Provision Thoughput
&emsp;&emsp;客户可以根据实际需求决定是否选择Gloabl Endpoint。
