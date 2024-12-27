---
title: Gemini入门系列（6）——新Gemini API
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 51fdaedf
date: 2024-12-27 19:53:54
---

&emsp;&emsp;Gemini1.5及之前版本，使用的API库为google-cloud-aiplateform。随着Gemini2.0的发布，GCP也发布了新的API库google-genai。虽然Gemini2.0的普通功能在google-cloud-aiplateform也可以调用，但一些新的如multimodal、real-time等，需要google-genai支持。以下边代码为例简单介绍一下使用的区别。
<!-- more -->
```
import os
from google import genai
from google.auth import default

os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = 'path to key'

credentials, project = default(scopes=["https://www.googleapis.com/auth/cloud-platform"])

client = genai.Client(
    vertexai=True, credentials=credentials, project='ai-demo-440003', location='us-central1'
)

response = client.models.generate_content(
    model='gemini-2.0-flash-exp', contents='What is your name?'
)
print(response.text)
```
# Vertex AI和AI Studio的API整合
&emsp;&emsp;初接触Gemini很容易有困惑，为什么有时候api key可以有时候又不行，为什么quota有时候可以提升有时候又不能提升？等等。
&emsp;&emsp;实际上Gemini提供了两个平台：
* [AI Studio](https://ai.google.dev/aistudio)：这是为开发者提供的一个免费试用Gemini的平台，里边有大量Demo等，可供开发者参考和免费试用
* [Vertex AI](https://console.cloud.google.com/vertex-ai/studio/)这是在GCP上为商业用户提供的VI平台，收费，有SLA，有support，可以更具需要提升Quota
&emsp;&emsp;两者的主要区别如下：
&emsp;&emsp;[AI Studio Vs Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/migrate/migrate-google-ai?hl=zh-cn)
&emsp;&emsp;Gemini1.5及之前AI Studio和Vertex AI分别有各自的库：
* AI Studio：google-generativeai
* Vertex AI：google-cloud-aiplatform
&emsp;&emsp;到Gemini2.0，统一成了一个库：google-genai。
&emsp;&emsp;如下client的定义时中，有一个参数vertexai的设置，如果设置为True，即使用vertextai；如果没有设置（默认值为False）或设置为False，则使用AI Studio：
```
client = genai.Client(
    vertexai=True, credentials=credentials, project='ai-demo-440003', location='us-central1'
)
```

# 鉴权
&emsp;&emsp;AI Studio仍然可以通过api key鉴权。但Vertex AI鉴权有很大的变化，不再通过google.oauth2直接load service account key。下边具体描述。
## gcloud环境
&emsp;&emsp;如果客户环境可以安装gcloud（GCP命令行工具），鉴权非常简单，直接通过如下命令即可：
```
gcloud auth application-default login
```
&emsp;&emsp;如此配置后无需在代码里鉴权，代码简化为：
```
from google import genai

client = genai.Client(
    vertexai=True, project='ai-demo-440003', location='us-central1'
)

response = client.models.generate_content(
    model='gemini-2.0-flash-exp', contents='What is your name?'
)

print(response.text)
```
&emsp;&emsp;运行可能会有警告没有设置quota- project和没有在gcloud配置默认project
，通过如下命令配置即可：
```
gcloud config set project project_id
gcloud auth application-default set-quota-project project_id
```
## 非gcloud环境
&emsp;&emsp;如果开发或使用环境中各种原因无法安装gcloud，也可通过service account key鉴权。方法如下：
1. 设置环境变量（key的路径）：
    ```
    export 'GOOGLE_APPLICATION_CREDENTIALS'='path to key'
    ```
2. 在代码里通过google.auth创建凭据：
    ```
    from google.auth import default
    credentials, project = default(scopes=["https://www.googleapis.com/auth/cloud-platform"])
    ```
3. 在client定义时指定凭据：
    ```
    client = genai.Client(
    vertexai=True, credentials=credentials, project='ai-demo-440003', location='us-central1'
    )
    ```
&emsp;&emsp;这样，就可以正常获得相关权限调用API了。
&emsp;&emsp;当然，更便捷的方式是通过os库把环境变量设置也放到代码里：
```
import os
from google import genai
from google.auth import default

os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = 'path to key'

credentials, project = default(scopes=["https://www.googleapis.com/auth/cloud-platform"])

client = genai.Client(
    vertexai=True, credentials=credentials, project='ai-demo-440003', location='us-central1'
)
```
&emsp;&emsp;需要注意的是，这里的环境变量设置是临时的，只在此代码段运行时有效。
# 错误捕获
&emsp;&emsp;google-genai内置了错误捕获，无需再通过其他库，相关信息可参见上一篇429问题处理的介绍

# 小结
&emsp;&emsp;Gemini2.0的库变化比较大，可能很多时候有点摸不着头脑，这里做个简单汇总，希望有帮助。