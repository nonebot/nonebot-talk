---
theme: seriph
highlighter: shiki
lineNumbers: true
---

<!-- markdownlint-disable MD003 MD033 MD022 -->

# NoneBot2 开发研讨会

## *NoneBot 2.0.0-beta1 is coming s∞n*

主持人: @yanyongyu

---

# 主要内容

- 👀 2.0.0-beta1 发布前瞻
  - 🎉 依赖注入
  - 🔧 OneBot Adapter 内置工具函数
  - 🕹️ 拓展格式化控制符
  - 🚚 项目文件结构更改

- 📝 全新文档框架: `nonepress`

- 💬 讨论议题
  - 📛 `nonebot.adatper.cqhttp`改名
  - 🕸️ Adapter连接层抽象

- ❓自由提问环节

---

# 依赖注入

TODO: 添加一个整活的副标题 [#588](https://github.com/nonebot/nonebot2/pull/588)

## 什么是依赖注入?

[**"依赖注入"**](https://zh.m.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC)意思是，在编程中，有一种方法可以让你的代码声明它工作和使用所需要的东西, 即"**依赖**"。

系统(在这里是指`NoneBot`)将负责做任何需要的事情，为你的代码提供这些必要依赖(即"**注入**"依赖性)

这在你有以下情形的需求时非常有用:

- 这部分代码拥有共享的逻辑(同样的代码逻辑多次重复)
- 共享数据库以及网络请求连接会话
  - 比如`httpx.AsyncClient`, `aiohttp.ClientSession`和`sqlalchemy.Session`
- 用户权限检查以及认证
- 还有更多...

它在完成上述工作的同时，还能尽量减少代码的耦合和重复

---

- 事实上, 在NoneBot中, 我们已经有了一种类似的思想, 即[事件处理函数重载](https://v2.nonebot.dev/advanced/overloaded-handlers.html)

```python {2,7}
@matcher.handle()
async def _(bot: Bot, event: GroupMessageEvent):
    await matcher.send("群聊消息事件响应成功！")


@matcher.handle()
async def _(bot: Bot, event: PrivateMessageEvent):
    await matcher.send("私聊消息事件响应成功！")
```

- 观察上面这段代码, 我们可以用两个`@matcher.handle`来接受来自两个不同类型的消息, 如果用依赖注入的观点来看待, 那么它可以这样分解:
  - `event: GroupMessageEvent`: 它注解了`GroupMessageEvent`类型, 即代表这个事件处理函数**依赖**一个`GroupMessageEvent`事件类. 当我们接受到群聊消息的时候, 事件满足该依赖, 就会调用这个函数
  - 当然, 在代码实现上, 这还不能被称作"依赖注入", 因为它缺乏一个自定义依赖的方法

接下来, 让我们用一个实际的例子, 来看看即将引入的依赖注入会为Bot的编写带来哪些好处

---

## 依赖注入示例

- 很多人都会写一款以图搜图插件, 它的大概逻辑一般来讲是这样:
  1. 用户发送`/搜图`指令, 触发图片搜索功能
  2. 机器人回复提示, 请求用户发送一张等待搜索的图片
  3. 用户发送图片进行搜索, 或者发送取消关键词取消搜索
  4. 根据用户输入进行搜索或者取消搜索

那么, 在目前的版本中, 我们可能需要这样去写:

```python
matcher = on_command('搜图')

@matcher.handle()
async def _(bot:Bot, event: MessageEvent):
    image = next( # 获取接受到消息中的第一张图片链接, 如果没有, 则为None
        [segment.data['url'] for segment in event.message if segment.type == 'image'], None) # 其实是一种装逼写法
    if image is not None:
        await matcher.reject('请发送一张图片进行搜索')
    result = await search_image(image) #可能是你自己的搜图函数实现
    await matcher.send(result)
```

接下来, 我们将用依赖注入的思想来实现这段代码

---

```python {2|6-12|17|all}
... #之前的import省略了
from nonebot.dependencies import Depends 

matcher = on_command('搜图')

async def first_image(event: MessageEvent):
    # 获取接受到消息中的第一张图片链接, 如果没有, 则为None
    image = next(
        [segment.data['url'] for segment in event.message if segment.type == 'image'], None) # 其实是一种装逼写法
    if image is not None:
        await matcher.reject('请发送一张图片进行搜索')
    return image

@matcher.handle()
async def handler(bot:Bot, 
            event: MessageEvent, 
            image: str=Depends(first_image) #这里就是依赖注入的核心魔法
          ):
    result = await search_image(image) #可能是你自己的搜图函数实现
    await matcher.send(result)
```

可以看到, 我们引入了一个新的东西, `Depends`, 它向`matcher.handle()`声明这个参数是一个依赖. 此时, `first_image`就是**依赖**, 而`handler`函数就是被**注入**的依赖方

> 可是, 它有什么用捏?

---

有些~~聪明的小朋友们~~可能就会想了

> 那既然这样, 我直接在`handler`函数里面执行`first_image`函数不行嘛, 就像这样:

```python {3}
@matcher.handle()
async def handler(bot:Bot, event: MessageEvent):
    image = first_image(event)
    result = await search_image(image) #可能是你自己的搜图函数实现
    await matcher.send(result)
```

好, 假设现在我们的要求图片`first_image`现在有了一个新的功能需求: 在接受到图片错误次数过多后, 直接结束当前指令, 这又该怎么写呢?

很显然, 我们现在需要操作`state`来处理当前指令的状态, 也就是说, 我们需要为`first_image`添加一个新的参数`state: T_State`

如果复用了`first_image`这个函数, 你可能就需要在所有调用该函数的地方都添加一个`state`参数, 这样会导致代码需要做很多改动

如果我们采用`Depends`的方式, 我们不会需要添加任何参数, 因为`state`将会直接作为一个依赖被注入到`first_image`函数中, 避免了代码的改动, 同时提高了代码的可读性

---

## One more thing

随着依赖注入的发布, 之前的`StateFactory`和`Rule`也将会使用`Depends`作为底层实现, 并逐步弃用

如果大家熟悉`FastAPI`的依赖注入的话, `NoneBot`实现的依赖注入和`FastAPI`实现的部分非常相似(~~直接抄的~~), 你甚至可以看看[FastAPI文档](https://fastapi.tiangolo.com/tutorial/dependencies/)

```python {monaco}
#这里留一段代码块给低调佬freestyle

handler = on_command('command')
```

---

# OneBot Adapter Helpers

帮助用户完成特定需求的工具函数 [#571](https://github.com/nonebot/nonebot2/pull/571)

在之前的搜图例子中, 我们自己造了一个函数, 叫做`first_image`

但是这其实是OneBot Adapter将要提供的一项被封装的能力

列个清单来讲, 它提供了以下内容:

- 原NoneBot1中[`nonebot.command.argfilter`包括的功能](https://docs.nonebot.dev/api.html#nonebot-command-argfilter-%E6%A8%A1%E5%9D%97)
  - `extract_image_urls`: 提取用户消息中所有图片的URL
  - `extract_numbers`: 提取用户消息中的所有数字, 包括带符号的浮点数和整数
  - `convert_chinese_to_bool`: 将中文（`好`, `不行` 等）转换成`bool`
  - `is_cancellation`: 判断用户是否有意图取消当前指令
    - 提供依赖注入版本`handle_cancellation`

- `CommandDebounce`: 一个[命令防抖/命令冷却实现](https://github.com/mixmoe/IzumiBot/blob/b13416d34acd0695091f131252f0aaad5c3840cb/IzumiBot/plugins/bot_utils/controllers.py#L44-L95)

---

因为前面几个移植的函数没什么好说的, 这里给一个`CommandDebounce`的示例

```python {3-9|10|all}
setu = on_command('setu')

debounce = CommandDebounce(setu, # 要应用防抖的matcher
                # 防抖隔离等级, 有全局/用户/群聊/群聊+用户模式可选
                isolate_level=CommandDebounce.IsolateLevel.GROUP,
                # 防抖超时事件(aka冷却时间/cool down)
                debounce_timeout=11.4514,
                # 当频率过高时发送的消息, 可选
                cancel_message='冲太快辣, 别冲辣')
debounce.apply() #应用防抖到matcher
```

这个时候, 如果群友冲太快(即一个群在11.4514秒内调用了两次`setu`指令), bot就会给群友发`冲太快辣, 别冲辣`, 保护了群友的身心健康

---

# 拓展格式化控制符

使用`MessageSegment`的工厂方法作为格式化控制符 [#555](https://github.com/nonebot/nonebot2/pull/555)

在NoneBot`2.0.0-alpha14`中, 我们引入了消息模板这一项功能, 具体介绍可以移步[discussions#41](https://github.com/nonebot/discussions/discussions/41)

> 我们计划在下个版本中加入类似`{link:image}`这样的支持! 直接把链接转化为图片! 听起来很棒是不是!

忽略掉某人的发病语言风格, 来看这个功能

这个功能抽象来说还是非常简单的, 就是可以使用`Message`对象对应的`MessageSegment`的工厂方法作为字符串的**格式化控制符**(*format spec*, 没找到正式翻译, 自己瞎翻的)

我们可以看下面一段代码作为例子

---

现在我们有一段这样从某个API获取的数据:

```json
{ 
  "avatar": "https://stdrc.cc/static/favicons/favicon-32x32.png"
  "type": "美少女",
  "name": "RichardChien",
  "relationship": "老婆",
}
```

来写段代码

```python {3-7|9-12|all}
data = await get_relationship(...)

# 原来需要这样写
data['avatar'] = MessageSegment.image(data['avatar']) # 将其中特定字段转换为segment
template = Message.template('{avatar} {type} @{name} 是我{relationship}')
template.format(data)
# >>> '[图片] 美少女 @RichardChien 是我老婆'

# 现在可以这样写
template = Message.template('{avatar:image} {type} @{name} 是我{relationship}')
template.format(data)
# >>> '[图片] 美少女 @RichardChien 是我老婆'
```

---

# 全新文档NonePress

基于docusaurus的一个新的文档页面 [nonebot/docusaurus-theme-nonepress](https://github.com/nonebot/docusaurus-theme-nonepress)

<table>
  <tr>
    <td><img src="/1st/new-doc-1.png"></td>
    <td><img src="/1st/new-doc-2.png"></td>
  </tr>
  <tr>
    <td><img src="/1st/new-doc-3.png"></td>
  </tr>
</table>

---

# `nonebot.adapter.cqhttp`改名

由于`OneBot v12`标准的推出, 讨论是否修改当前Adapter名称

目前, 有以下几个可选方案

- 重命名`nonebot.adapter.cqhttp`为`nonebot.adapter.onebot`
  - 其中`.v11`处理v11协议(即目前协议), `.v12`处理v12新协议
  - 好处: 命名清晰明确
  - 坏处: 要改现有代码, 是大型破坏性修改
  
- 保留`nonebot.adapter.cqhttp`来处理v11协议, 新建`nonebot.adapter.onebot`处理v12协议
  - 好处: 兼容性强
  - 坏处: `cqhttp`命名容易混淆, 而且无法推动用户使用新协议
  
- 或者还有其他方案? Orz

---

# Adapter连接层抽象

独立出Adapter的连接层来单独处理事件

目前, NoneBot2的网络连接层是和Bot耦合的, 基本类似于一个连接对应一个Bot, 这会造成跨平台兼容性不足

而注册的Adapter实际上为Bot对象, 这也会造成概念上的混淆

现在正在考虑抽象出Adapter的网络连接为单独一层, 解除部分Bot对象的耦合功能

可能对第三方Adapter开发者负担较大

---
layout: center
---

# 自由提问

问, 都可以问 ~~能答上算我输~~

![今回はここまで](/1st/the-end.png)
