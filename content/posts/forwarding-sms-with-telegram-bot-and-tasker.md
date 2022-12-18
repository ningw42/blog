---
title: "使用 Telegram Bot 和 Tasker 转发短信"
date: 2018-06-20T20:53:45+08:00
tags: [Android, Tasker, Telegram]
---

要求：

1. 接收短信的设备必须是 Android 设备
2. 需要购买 Tasker
3. 接收短信的设备能正常访问 Telegram Bot API

主要分为以下几步：

1. 新建和使用 Telegram Bot 
	- 建立一个 Telegram Bot，获得 Telegram Bot Token
	- 调用 API
2. 配置 Tasker 来转发短信
	- 在 Tasker 中配置 Profile
	- 为该 Profile 配置对应的 Task
	- 调试

## 新建和使用 Telegram Bot

### 建立一个 Telegram Bot，获得 Telegram Bot Token

参考 [Telegram Bot 的文档](https://core.telegram.org/bots#3-how-do-i-create-a-bot)，新建一个 Bot，并获得 Token。

### 调用 API
参考 [Telegram Bot API 文档中关于鉴权的描述](https://core.telegram.org/bots/api#authorizing-your-bot)，调用任何 API 的格式都是 `https://api.telegram.org/bot<token>/METHOD_NAME`，同时接受 HTTP GET 和 HTTP POST。

转发短信所需要用到的 API 是 [Send Message](https://core.telegram.org/bots/api#sendmessage)。需要用到的参数只有 `chat_id`，`text` 和 `parse_mode`。其中 `text` 就是 Bot 向 Telegram 用户发送的内容，`parse_mode` 用于表示 `text` 中内容的解析方式，`chat_id` 是对话 ID。

`chat_id` 是唯一需要自己通过额外途径获得的值，可以使用通过[这个 Telegram Bot](https://telegram.me/get_id_bot) 获得。

在浏览器中键入这样的 URL 就可以正常调用 Telegram Bot API 来发送消息给自己了。`https://api.telegram.org/bot<TOKEN>/sendMessage?text=hello_world&chat_id=<YOUR_CHAT_ID>`。此时你将收到发自刚刚新建的 Bot 的消息 “hello_world”。

## 配置 Tasker 来转发短信

至此我们已经能够通过 Bot 给自己发送任何消息了，接下来就是如何获得短信内容，并调用以上 API 的过程。[Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm) 是一款 Android 上的软件，可以监听各种系统事件，并且执行相应任务的软件。

### 在 Tasker 中配置 Profile

在 Tasker 的 Profiles tab 中新建一个 Event 类型的 Profile，类型选择 Received Text，按需求配置是否需要过滤类型，发送者和内容。

### 为该 Profile 配置对应的 Task

1. 返回后 Tasker 会提示为当前 Profile 配置对应的任务，这里选择 New Task，并输入任务的名字。
2. 输入名字后会自动进入任务编辑，点击右下角的按钮新建 Action，选择 Action 类型为 HTTP Post。
3. 在 Server:Port 这一栏填写 Telegram 的 API Host `https://api.telegram.org`。
4. 在 Path 这一栏按上文填写 `/bot<TOKEN>/sendMessage`
5. 在 Data/File 这一栏填写以下 Code Block 中的内容
6. 在 Content Type 这一栏填写 `application/json`

```json
{
    "chat_id": <YOUR_CHAT_ID>,
    "parse_mode": "HTML",
    "text": "<b>%SMSRF(%SMSRN)</b> \n\n%SMSRB",
}
```

其中有 `%SMSRF`，`%SMSRN` 和 `%SMSRB` 这三个字符串是 Tasker 的变量，分别用于表示 Received Text 事件中短信的发件号码，发件人（如果在通讯录中就会是名字，否则一样是号码）和短信内容。

编辑完成后需要点击右上角的按钮应用更改。

### 调试

Tasker 的任务提供了直接运行的功能。进入刚刚新建的 Task，点击左下角的“Run”按钮，此时 Bot 就会发送上文 `text` 中的内容。如果不成功，详细对照上文步骤。

## 其他

通过这样的模式还可以实现通知未接来电等等一系列功能。

其实更方便的方法是下载 [Remote Bot for Telegram](https://play.google.com/store/apps/details?id=com.alexandershtanko.androidtelegrambot)，这个应用以同样的原理实现了更多的功能，只不过它不需要 Tasker，而是自己来处理。

我自己的目标是将 Android 上的短信转发到 iOS 上，按短信接收平台和转发目标平台来分，可以用下面的方法。

| 短信接收平台(Row)/转发目标平台(Column) | iOS | Android |
| --- | --- | --- |
| iOS | 请用 Messages 自带的 Text Message Forwarding | 据我所知无解 |
| Android | 本文方法可行 | 文本方法可行，不排除有更好的方法 |





