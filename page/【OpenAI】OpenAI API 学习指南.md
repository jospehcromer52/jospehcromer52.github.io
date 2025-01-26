## 获取 OpenAI API Key

### 1. 创建 OpenAI 账号

如果您之前使用过 ChatGPT，那么您可以直接使用该账号作为您的 OpenAI 账号。若您尚未注册，可以前往 OpenAI 官网进行注册。

> 由于国内网络限制，注册可能较为复杂，网上有许多相关教程，具体操作这里不再详细说明。您也可以选择通过购买账号或已有额度的 API Key 来快速获取。

### 2. 获取 OpenAI API Key

登录到 OpenAI 后，鼠标移动到页面左侧，您会看到一个弹出的侧边栏。

1. 点击侧边栏中的“API Keys”进入 API Keys 页面。
2. 在新页面，您可以管理（创建、删除等）API Key。

点击“Create new secret key”创建新 API Key，填写名称并确认。此时会弹出包含您刚创建的 Key 的对话框，请务必立即保存此 Key，因为关闭后将无法再查看。

完成后，点击 "Done"，您将在页面上看到新创建的 API Key。

## 查询 API 使用额度

### 额度查询

在侧边栏中点击“Usage”可以进入使用页面。该页面左侧显示了每日花费，右侧则显示额度信息。在 Credit Grants 区域，显示有三种颜色：灰（未使用）、绿（已使用）、红（已过期）。只有在额度处于未使用状态（灰色）时，您才可以成功调用 API。

### 额度充值

点击侧边栏的“Setting”下的“Billing”进入账单页面。在此页面您可以管理充值相关事项。

首先需要添加付款方式，点击 Payment methods 可以管理自己的支付方式。由于网络限制，可能国内的 Visa 卡无法使用，您可以考虑使用国外的卡或网络卡。

添加付款方式后，返回 Overview 页面点击 Add to credit balance 就可以进行充值。充值后，回到 Usage 页面查看可用额度的变化。

## Python 使用测试

### 1. 环境配置

确保 Python 版本在 3.7.1 以上。为了便于使用，我推荐使用 Anaconda 创建一个虚拟环境。

### 2. 安装 OpenAI 库

### 3. 设置 API Key

OpenAI 默认会查找环境变量中的 "OPENAI_API_KEY" 作为您的 Key。您可以通过以下两种方式设置：

#### - 为所有项目设置

在系统环境变量中添加 "OPENAI_API_KEY"。可通过在 Windows 中搜索环境变量来打开设置页面。

添加完成后，可以通过命令行使用 `echo %OPENAI_API_KEY%` 确认设置是否成功。

python
from openai import OpenAI

client = OpenAI()


#### - 为单个项目设置

在项目文件夹中创建 `.env` 文件（在使用 git 时，需将其添加到 gitignore 文件中），输入 `OPENAI_API_KEY=`（您的 Key）。

plaintext
# 确保您的 API Key 保密
OPENAI_API_KEY=abc123


完成以上步骤后，运行测试，确保没有报错。

如果您需要使用其他环境变量名，可以使用如下代码：

python
client = OpenAI(
    api_key=os.environ.get("CUSTOM_ENV_NAME"),
)


### 发送请求测试

下面是一个简单的 gpt-3.5 chat 请求示例：

python
import os
import dotenv
from openai import OpenAI

dotenv.load_dotenv()

client = OpenAI(
    api_key=os.environ.get("OPENAI_API_KEY"),
)

# 发送请求
completion = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."},
        {"role": "user", "content": "Compose a poem that explains the concept of recursion in programming."}
    ]
)

# 输出回复
print(completion.choices[0].message.content)


执行请求后，您可以在 [Usage 页面](https://platform.openai.com/usage) 查看请求的花费和 token 数量等信息（可能有延迟）。

## 功能介绍（Python 举例）

### Text Generation

您可以通过 GPT-4 理解自然语言（视图功能支持图像理解），并返回文本。主要输入是 messages，以下是一个示例：

python
response = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Who won the world series in 2020?"},
        {"role": "assistant", "content": "The Los Angeles Dodgers won the World Series in 2020."},
        {"role": "user", "content": "Where was it played?"}
    ]
)


每个回复中有一个 `finish_reason` 字段，可能的值包括：

- `stop`：信息完成或触发了 stop 参数。
- `length`：达到最大 token 或模型限制。
- `tool_call`：模型决定调用工具。
- `content_filter`：由于内容过滤，信息被隐藏。
- `null`：尚未完成。

请注意，GPT-4 的 vision 版本支持图像输入，您只需在 user message 的 content 中添加图像的 URL。

### Image Generation

您可以参考官方文档了解更多功能：
👉 [野卡 | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)