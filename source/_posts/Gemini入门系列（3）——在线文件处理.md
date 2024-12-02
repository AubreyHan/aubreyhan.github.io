---
title: Gemini入门系列（3）——在线文件处理
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 7a9c8f23
date: 2024-11-17 11:02:51
---

&emsp;&emsp;上一篇谈了本地文件处理。本地文件处理虽然方便，但收到很多方面制约，例如文件大小、文件内容等，另外网络传输也可能导致不成功。
<!-- more -->
&emsp;&emsp;为了解决此问题，Gemini提供了多种在线文件的处理方式：
![](https://home.aubreyhan.net:9000/blogpic/17318093833731.jpg)

&emsp;&emsp;特别是002模型开始，可以直接处理公开url和Youtube，大大方便了使用。
# 从Cloud Storage导入
&emsp;&emsp;Cloud Storage（GCS）是GCP提供的对象服务，和Gemini有良好的整合，需要处理的文件放入GCS，在Gemini处理是最方便的方式，如下在console选择从Cloud Storage导入处理一张GCS的图片：
![](https://home.aubreyhan.net:9000/blogpic/17318103135664.jpg)
![](https://home.aubreyhan.net:9000/blogpic/17318103487733.jpg)

&emsp;&emsp;同样，可以获取处理Cloud Storage文件的代码：
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
        [image1, """请描述上图内容"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
        print(response.text, end="")

image1 = Part.from_uri(
    mime_type="image/jpeg",
    uri="gs://gemini-demo-bucket-001/1.jpg",
)

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
&emsp;&emsp;此代码中，用了如下部分定义Cloud Storage导入文件：
```
image1 = Part.from_uri(
    mime_type="image/jpeg",
    uri="gs://gemini-demo-bucket-001/1.jpg",
)
```

# Youtube视频链接
&emsp;&emsp;001模型虽然支持Youtube视频，但是限制必须是自己组织上传的内容，002放开了这个限制，只要是公开视频，都可放入Gemini处理。
&emsp;&emsp;Youtube视频链接处理很简单，点解选项然后填入链接url即可：
![Xnip Helper 2024-11-17 10.36.23](https://home.aubreyhan.net:9000/blogpic/Xnip%20Helper%202024-11-17%2010.36.23.png)

&emsp;&emsp;根据视频长短，处理时间不一样：
![](https://home.aubreyhan.net:9000/blogpic/17318112560881.jpg)

&emsp;&emsp;同样，在获取的代码里：
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
        [video1, """请描述视频内容"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
        print(response.text, end="")

video1 = Part.from_uri(
    mime_type="video/*",
    uri="https://www.youtube.com/watch?v=r5HFLkNrbIU",
)

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
&emsp;&emsp;如下代码定义了使用Youtube链接的方法：
```
video1 = Part.from_uri(
    mime_type="video/*",
    uri="https://www.youtube.com/watch?v=r5HFLkNrbIU",
)
```

# 通用网址
&emsp;&emsp;如果上传到GCS或者Youtube不方便，希望能处理非Google环境的文件，从002开始，Gemini可以处理通用网址。客户可将文件上传至自己使用方便的网络服务，然后将网址传递给Gemini即可。
&emsp;&emsp;当然这个方式会有些限制，例如一些视频网站，公开网址是播放网址而非文件url，或者一些网盘，提供的网址并非直接文件访问的url，这种方式就无法处理。必须获得真实的文件url。
&emsp;&emsp;如下，将第一个例子中的图片放入我在家创建的一个存储，并获取公开访问的url作为通用网址：
![](https://home.aubreyhan.net:9000/blogpic/17318121396571.jpg)

&emsp;&emsp;让Gemini处理：
![](https://home.aubreyhan.net:9000/blogpic/17318122387886.jpg)

&emsp;&emsp;代码如下：
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
        [image1, """请用中文总结以上图片的内容及使用场景"""],
        generation_config=generation_config,
        safety_settings=safety_settings,
        stream=True,
    )

    for response in responses:
        print(response.text, end="")

image1 = Part.from_uri(
    mime_type="image/jpeg",
    uri="https://ahts873a.myqnapcloud.com:8010/v1/AUTH_Cloud/demo/1.jpg",
)

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
&emsp;&emsp;通用网址的处理部分：
```
image1 = Part.from_uri(
    mime_type="image/jpeg",
    uri="https://ahts873a.myqnapcloud.com:8010/v1/AUTH_Cloud/demo/1.jpg",
)
```

# 小结
&emsp;&emsp;从以上例子可以看到，Gemini可以通过多种方式处理在线文件，Part.from_uri类是处理在线文档的基础方法。
&emsp;&emsp;合理使用在线文档，可大大方便Gemini的使用。
