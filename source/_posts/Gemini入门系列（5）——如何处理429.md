---
title: Gemini入门系列（5）——如何处理429
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 7b70a115
date: 2024-12-26 06:55:08
---

&emsp;&emsp;429是Gemini访问中经常会出现的一个报错，原因是quota（qpm）不足导致。如果是生产环境，最好的解决方案是根据购买PT独享的方式。但有些场景，例如PoC、或者并发忽高忽低等，可能PT并不适合，这时候就要通过其他技术手段解决。
<!-- more -->
&emsp;&emsp;解决429的思路也很简单，通过retry的方式，如果出现429，就反复重试，直到执行成功。为了提供成功率，可以多选择几个region，自动随机挑选，提高执行效率。
&emsp;&emsp;Gemini2.0开始，使用新的API，捕捉异常代码的方式和之前API不一样，下边分别举例说明。
# google-cloud-aiplatform
&emsp;&emsp;google-cloud-aiplatform是Gmini1.5及之前版本用的API，捕捉异常代码通过google-api-core这个API进行，代码如下：
```
from vertexai.preview.generative_models import GenerativeModel
import vertexai
from google.api_core.exceptions import ResourceExhausted
import random

locationList = [
    'us-central1',
    'us-east1',
    'us-east4',
    'us-west1'
]

def GenAI(location: str):
    vertexai.init(project='ai-demo-440003', location=location)  
    model = GenerativeModel(
        "gemini-1.5-flash-002",
    )
    generation_config = {
        "max_output_tokens": 8192,
        "temperature": 1,
        "top_p": 0.95,
    }
    responses = model.generate_content(
        ["""hello"""],
        generation_config=generation_config,
        stream=True,
    )
    for response in responses:
        print(response.text, end="")

while True:
    try:
        location = random.choice(locationList)
        GenAI(location)
        break
    except ResourceExhausted as e:
        if e.code == 429:
            continue
```
&emsp;&emsp;这段代码，除了GenAI所用到的库，还额外导入两个库
* from google.api_core.exceptions import ResourceExhausted：异常代码的库
* import random： 随机库
&emsp;&emsp;如下代码定义函数使用GenAI生成回复，其中Location作为函数参数传递：
```
def GenAI(location: str):
    vertexai.init(project='ai-demo-440003', location=location)  
    model = GenerativeModel(
        "gemini-1.5-flash-002",
    )
    generation_config = {
        "max_output_tokens": 8192,
        "temperature": 1,
        "top_p": 0.95,
    }
    responses = model.generate_content(
        ["""hello"""],
        generation_config=generation_config,
        stream=True,
    )
    for response in responses:
        print(response.text, end="")
```
&emsp;&emsp;这段代码定义了一个region的列表，即在那些region去调用Gemini的API，如果需要限制在一个Region，可以只放需要的region：
```
locationList = [
    'us-central1',
    'us-east1',
    'us-east4',
    'us-west1'
]
```
&emsp;&emsp;这部分是真正调用Gemini的地方，这里并不是直接调用预先定义的GenAI函数，而是用了一个while循环：
```
while True:
    try:
        location = random.choice(locationList)
        GenAI(location)
        break
    except ResourceExhausted as e:
        if e.code == 429:
            continue
```
&emsp;&emsp;进入循环首先随机从region list选择一个region，并作为参数传递给GenAI函数。如果执行成功，字节break循环跳出。
&emsp;&emsp;如果执行不成功，在ecept代码段捕获的异常代码如果是429，继续从头执行循环，再次随机挑选一个region执行，直到成功。
&emsp;&emsp;通过这种简单粗暴的方式，可以实现出现429错误时自动retry，避免人工干预。

# google-genai
&emsp;&emsp;从Gemini2.0开始，GCP将vertexai和ai studio的API整合到一起，采用了新的google-genai。其中一个改变就是异常捕获的方式，不再使用google-api-core（2.0GA后是否会变未知）。
&emsp;&emsp;更改后的code如下：
```
from google import genai
from google.genai import types
from google.genai.errors import ClientError
import random

locationList = [
    "us-central1"
]

def GenAI(location):
  client = genai.Client(
      vertexai=True,
      project="ai-demo-440003",
      location=location
  )


  model = "gemini-2.0-flash-exp"
  contents = [
    types.Content(
      role="user",
      parts=[
        types.Part.from_text("""hello""")
      ]
    )
  ]
  generate_content_config = types.GenerateContentConfig(
    temperature = 1,
    top_p = 0.95,
    max_output_tokens = 8192,
    response_modalities = ["TEXT"],
    safety_settings = [types.SafetySetting(
      category="HARM_CATEGORY_HATE_SPEECH",
      threshold="OFF"
    ),types.SafetySetting(
      category="HARM_CATEGORY_DANGEROUS_CONTENT",
      threshold="OFF"
    ),types.SafetySetting(
      category="HARM_CATEGORY_SEXUALLY_EXPLICIT",
      threshold="OFF"
    ),types.SafetySetting(
      category="HARM_CATEGORY_HARASSMENT",
      threshold="OFF"
    )],
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    print(chunk.text, end="")

while True:
  try:
    location = random.choice(locationList)
    GenAI(location)
    break
  except ClientError as e:
    if e.code == 429:
      continue
```
&emsp;&emsp;和之前对比，可以看到，捕获异常的库变为了：
```
from google.genai.errors import ClientError
```
&emsp;&emsp;其它使用方法基本一样。
&emsp;&emsp;另外要注意的是，目前Genmini2.0只在us-central1提供，所以在region list里只有这一个。以后部署的region多了，可以随时添加。

# 小结
&emsp;&emsp;通过捕获异常的方式解决429不难，这里提供简单的代码供参考。希望Gemini越用越好。