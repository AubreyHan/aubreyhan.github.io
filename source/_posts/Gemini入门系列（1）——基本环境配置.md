---
title: Gemini入门系列（1）——基本环境配置
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 7040f2d1
date: 2024-10-28 19:39:17
---

&emsp;&emsp;最近使用Gemini的用户越来越多。国内Gemini的资料相对较少，很多用户遇到问题很难快速找到资料，所以打算记录一下Gemini的基本使用，方便查找。
&emsp;&emsp;本系列假定已对GCP（Google Cloud Platform）有基础使用经验并可以科学上网。
<!-- more -->
# GCP环境准备
&emsp;&emsp;使用Gemini，首先需要一个GCP项目，登录GCP Console（cloud.google.com）并记录项目的相关信息（Project Num和Project ID）：
![](https://home.aubreyhan.net:9000/blogpic/17300858734628.jpg)

&emsp;&emsp;如下进入Vertex AI的Gemini界面：
![](https://home.aubreyhan.net:9000/blogpic/17300859341735.jpg)

&emsp;&emsp;如果是该项目第一次使用Vertex AI，会提示激活API：
![](https://home.aubreyhan.net:9000/blogpic/17300859785617.jpg)

&emsp;&emsp;激活后点击左侧Freeform，即可输入Prompt使用Genimi：
![](https://home.aubreyhan.net:9000/blogpic/17300862817401.jpg)


# 开发环境准备
&emsp;&emsp;当然，首先得要有IDE，下边以最常用的VS Code为例。Python要求3.8以上版本，为了与其他环境隔离，建议创建venv，并在VS Code安装Python解释器方便调试：
![](https://home.aubreyhan.net:9000/blogpic/17300943143734.jpg)

&emsp;&emsp;安装GCP AI的库：
```
pip install google-cloud-aiplatform
```
![](https://home.aubreyhan.net:9000/blogpic/17300944610561.jpg)

&emsp;&emsp;现在复制Gemini Sample Code测试是否可以正常运行。在GCP Console的Gemini界面右上角点击Get Code，可以看到对应的Sample Code：
![](https://home.aubreyhan.net:9000/blogpic/17300960467511.jpg)


&emsp;&emsp;复制到IDE：
```
import base64
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting


def generate():
    vertexai.init(project="ai-demo-440003", location="us-central1")
    model = GenerativeModel(
        "gemini-1.5-flash-002",
    )
    responses = model.generate_content(
        ["""Please tell me how to lear python"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
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

generate()
```
![](https://home.aubreyhan.net:9000/blogpic/17300960789703.jpg)


&emsp;&emsp;运行，出现错误：
![](https://home.aubreyhan.net:9000/blogpic/17300962382160.jpg)

&emsp;&emsp;这个错误的原因是，运行环境未找到用作授权的service account。
&emsp;&emsp;这是是Gemini和其他LLM差异较大的一点。其他LLM一般通过API Key认证，很多用户都在这一步不知道如何处理。下边就看看操作流程：

&emsp;&emsp;首先还是要在GCP Console进入Service Account的管理界面：
![](https://home.aubreyhan.net:9000/blogpic/17300965790448.jpg)

&emsp;&emsp;点击Create Service Account，首先给创建的Service Account一个名字：
![](https://home.aubreyhan.net:9000/blogpic/17300966432280.jpg)

&emsp;&emsp;第二步，因为要通过这个Service Account使用Gemini，所以需要赋予相应权限（Vertex AI User）：
![](https://home.aubreyhan.net:9000/blogpic/17300967344294.jpg)

&emsp;&emsp;点击Done创建完成。
&emsp;&emsp;下一步，需要给Service Account创建一个Key，这个Key具备Service Account权限，用作开发环境鉴权。在Service Account列表界面点击刚创建的Service Account，选择KEYS页面，并点击ADD KEY：
![](https://home.aubreyhan.net:9000/blogpic/17300969184218.jpg)

&emsp;&emsp;选择Create new key，创建一个JSON格式的key：
![](https://home.aubreyhan.net:9000/blogpic/17300970675561.jpg)

&emsp;&emsp;创建完成，私钥会自动下载到本地。将key上传到开发环境并记录其路径，然后修改Sample Code如下：
```
import base64
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting
from google.oauth2 import service_account    #导入google认证库的service account模块

#通过service account key创建credential
cred = service_account.Credentials.from_service_account_file(
    '/home/gcpvm/ai-demo-440003-7b8cf6bf07d5.json'
)


def generate():
    vertexai.init(project="ai-demo-440003", location="us-central1", credentials=cred)  #在初始化环境配置credential
    model = GenerativeModel(
        "gemini-1.5-flash-002",
    )
    responses = model.generate_content(
        ["""Please tell me how to lear python"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
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

generate()
```

&emsp;&emsp;再次运行成功：
![](https://home.aubreyhan.net:9000/blogpic/17300980649223.jpg)

# 小结
&emsp;&emsp;本篇文档描述如何配置Gemini的开发环境，内容比较简单，希望对不熟悉GCP Vertex AI的朋友有所帮助。
&emsp;&emsp;配置过程中如果有问题，可以检查是否权限方面的配置不当。
