---
title: 在Vertex使用服务账号授权
tags:
  - GCP
  - AI
categories:
  - GCP
  - AI
abbrlink: 4d424ba1
date: 2024-09-20 08:39:57
---

&emsp;&emsp;使用AI Studio或者其他LLM平台时，很多用户习惯使用api key授权。但在Vertex需要通过服务账号（Service Account）key的方式授权。这个过程并不复杂，但是没有使用过服务账号的用户，可能对服务账号、key、IAM、Role的概念及操作流程有些疑惑。下边就通过简单demo进行说明。
<!-- more -->
# 关于授权的基本概念
&emsp;&emsp;在GCP平台，使用IAM对相关账户权限进行管理，涉及的相关概念如下：
* 权限（Permission）：权限定义了对资源可执行的操作（资源即GCP的组织、文件夹、项目、服务等用户需要操作管理的设施）
* 角色（Role）：权限并不能直接使用（赋予用户），而是通过将一个或者多个权限组成集合（角色），再赋予用户；GCP平台内置若干预定义角色，可直接使用，用户也可以根据需要将若干权限组合创建自定义角色
* 账号：使用GCP的用户，通常用邮箱（user@example.com）形式表示
* 服务账号：服务账号是一种特殊的账号，一般不由真人使用，而是提供给服务获得调用api的权限
&emsp;&emsp;IAM的基本逻辑如下：
![](https://home.aubreyhan.net:9000/blogpic/17267942796984.jpg)

# 在Vertex使用服务账号授权
## 创建服务账号并授权
&emsp;&emsp;服务账号的创建很简单，在console点击‘IAM和管理’-‘服务账号’进入服务账号的管理界面：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267951597877.jpg)

&emsp;&emsp;点击‘创建服务账号’开始创建：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267969791409.jpg)

&emsp;&emsp;第一步为服务账号命名及确定ID，ID是服务账号的唯一属性，确定后不可修改：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267975540722.jpg)

&emsp;&emsp;然后向服务账号授权（绑定角色），可以在这里授权也可以以后授权，或者以后也可以修改。这里因为要使用Vertex AI，所以绑定Vertex AI User角色：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267978361842.jpg)

&emsp;&emsp;下一步可以先忽略，直接完成即可。
&emsp;&emsp;现在到IAM里，可以看到创建的服务账号和绑定的角色：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267980753177.jpg)

&emsp;&emsp;此处点击后边🖊️可以对该账号的角色进行增删改操作。

## 获取服务账号的key
&emsp;&emsp;新建的服务账号已具备所需权限，在开发环境里我们需要通过该服务账号的key在开发环境赋权。
&emsp;&emsp;回到服务账号界面，点击已建好的服务账号，进入其管理界面：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267988461281.jpg)

&emsp;&emsp;点击密钥并创建新密钥（也可以上传现有密钥授权）：
![Xnip Helper 2024-09-20 10.21.03](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/Xnip%20Helper%202024-09-20%2010.21.03.png)

&emsp;&emsp;保持JSON格式：
![Xnip Helper 2024-09-20 10.21.41](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/Xnip%20Helper%202024-09-20%2010.21.41.png)

&emsp;&emsp;创建后会自动将私钥保存到本地，这就是下一步说需要的key：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17267990183075.jpg)

## 使用服务账号key授权Vertex
&emsp;&emsp;在客户端运行如下sample code：
```
import base64
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting


def generate():
    vertexai.init(project="hy-20240914-001", location="us-central1")
    model = GenerativeModel(
        "gemini-1.5-flash-001",
    )
    responses = model.generate_content(
        ["""请描述一个宠物"""],
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
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
]

generate()
```
&emsp;&emsp;输出没有授权的错误：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17268094499025.jpg)

&emsp;&emsp;修改code如下，加入通过key认证：
```
from google.oauth2 import service_account #导入服务账号认证所需的库
import base64
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting

#通过服务账号key文件创建凭据
cred = service_account.Credentials.from_service_account_file('/home/aubreyhan/hy-20240914-001-3d2739517190.json')

def generate():
    vertexai.init(project="hy-20240914-001", location="us-central1", credentials=cred) #在Vertex环境配置中引入凭据
    model = GenerativeModel(
        "gemini-1.5-flash-001",
    )
    responses = model.generate_content(
        ["""请描述一个宠物"""],
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
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
    SafetySetting(
        category=SafetySetting.HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=SafetySetting.HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE
    ),
]

generate()

```
&emsp;&emsp;参见code中注释有三处需要修改：
* 导入GCP认证库中服务账号的库
* 通过服务账号key创建凭据
* 在Vertex AI初始化时添加凭据
&emsp;&emsp;运行修改后code，可正常运行：
![](https://gcore.jsdelivr.net/gh/AubreyHan/blogimg/17268101867684.jpg)

# 总结
&emsp;&emsp;以上Demo简单演示了如何通过服务账户key在代码中授权Vertex AI。操作过程比较简单，但需要注意以下几点：
1. 同一个服务账号可以授权多个角色，所以如果客户端还需使用其他服务（例如GCS），可使用一个服务账号
2. 遵循最小权限原则，不要给服务账号不必要的权限
3. 一个服务账号可以创建多个key，并且设置过期时间，必要时通过轮换key实现安全性
4. 一定不要把key上传到公网！一定不要把key上传到公网！一定不要把key上传到公网！（重要事情说三遍）
5. 服务账号虽然归宿一个项目，但可以跨项目赋予权限，例如A项目的服务给B项目的服务账号授权；大型复杂项目里，可以有一个项目里专门创建多个服务账号，然后其他项目授权，便于服务账号的统一管理。

&emsp;&emsp;以上就是服务账号key在Vertex AI中的授权使用，希望对大家有帮助。
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;
&emsp;&emsp;