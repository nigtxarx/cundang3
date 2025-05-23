# 功能描述

## 语法扩展

对 SillyTavern 的宏语法进行扩展，以支持更复杂的语法，例如条件判断、循环、读取更多信息等.

兼容 SillyTavern 原有的宏，扩展的语法基于 [Embedded JavaScript templating](https://ejs.co/) 实现，能够在提示词中使用 `JavaScript`。

能够在**世界/知识书**、**预设**中的**提示词**、**角色相关内容**、**消息**中执行.

仅需要将提示词中使用`<% ... %>`语句块即可。

例如

```javascript
<% print('hello world!') %>
```

> 完整语法说明：[EJS Syntax Reference](https://github.com/mde/ejs/blob/main/docs/syntax.md)
>
> 可以函数列表：[Reference](reference_cn.md)

此功能将会在**将提示词发生给LLM**和**渲染到 SillyTavern 中**时执行

---

## 处理模板

此扩展将会在**开始生成**时将 **SillyTavern** 构建的提示词进行处理，执行`<% ... %>`语句块中的所有`JavaScript`代码，然后将其替换为相应的执行结果（如果有输出）

执行顺序：

1. SillyTavern 准备生成时的提示词（合并**预设**、**世界/知识书**、**角色定义**、**消息**等内容）
2. **此扩展**对提示词中的所有`<% ... %>`语句块进行处理
3. 将处理后的提示词发生到**LLM**
4. 接收**LLM**输出的内容，将其渲染到 SillyTavern 的消息当中
5. 待SillyTavern接收完全部的LLM输出后，**此扩展**开始对接收的内容进行处理（即处理可见的`<% ... %>`语句块）



例如，准备的提示词为：

```javascript
当前好感度：<%- variables.好感度 %>/100
```

此扩展会读取`variables.好感度`的值，并将语句块替换为实际的值

```javascript
当前好感度：50/100
```

然后，该内容将会发送到LLM并开始生成

当LLM生成接受后，如果输出中包含以下内容：

```javascript
<% setvar('好感度', 60) -%>
新的好感度: <%- variables.好感度 %>/100
```

此扩展将会对输出结果进行处理，最终显示结果为：

```javascript
新的好感度: 60/100
```

---

## 提示词注入

在某些情况下，**世界/知识书**执行顺序无法被控制，为了确保提示词能够放置在指定的位置，此功能允许将特定提示词插入到**开头**以及**结束**位置。

只需要为**世界/知识书**条目中的**标题（备忘）**添加前缀即可将该条目的内容注入到相应顺序，**必须设置为未激活才会生效**。

此功能会受到**触发策略**、**顺序**、**包含组**、**确定优先级**、**触发概率**、**组权重**、**主要关键字**、**逻辑**以及**可选过滤器**影响。

> 黏性、冷却和延迟未实现。

- `[GENERATE:BEFORE]`：将此条目注入到**发送给LLM**的提示词的开头（仅限🔵）

- `[GENERATE:AFTER]`：将此条目的内容注入到**发送给LLM**的提示词的末尾（🔵和🟢）

- `[RENDER:BEFORE]`：将此条目注入到**接收的LLM的输出**的内容开头（仅限🔵）

- `[RENDER:AFTER]`：将此条目注入到**接收的LLM的输出**的内容结尾（🔵和🟢）

  > `[RENDER:BEFORE]`与`[RENDER:AFTER]`仅用于渲染，不会发送到**LLM**
  >
  > 因此，也会遵循[楼层渲染](#楼层渲染)的设定

- `[GENERATE:{idx}:BEFORE]`：将此条目注入到**发送给LLM**的第`{idx}`条消息的开头（仅限🔵）

- `[GENERATE:{idx}:AFTER]`：将此条目的内容注入到**发送给LLM**的第`{idx}`条消息的末尾（🔵和🟢）

	> 以发送到 LLM 中的 `messages` 内容顺序为准，**`{idx}`从 0 开始**
	>
	> 例如 `[GENERATE:1:BEFORE]`为将提示词注入到第1条messages中（首条为0）

---

## 楼层渲染

在渲染聊天消息时，处理方式会与纯提示词有一些不同

- 渲染时直接使用楼层内已经渲染出来的内容进行处理，即使用楼层内的**HTML**代码，处理后也是直接输出到**DOM内的HTML**代码

> 即 `#chat > div.mes > div.mes_block > div.mes_text`的HTML代码

- 使用`<%=`格式来输出时会对内容进行格式化，而使用`<%-`输出时直接视为`HTML`代码

> 格式化包括：转义`HTML`标记、处理**宏定义**、处理**正则表达式**、处理**Markdown**语法
>
> 在通常情况下，`<%=`的功能与`<%-`相同，而只有在渲染时才会表现出不同行为

- 在[提示词注入](#提示词注入)中的`[RENDER:BEFORE]`和`[RENDER:AFTER]`条目也会遵循此设定

- 渲染时会将`&lt;%`替换为`<%`，`%&gt;`替换为`%>`，以支持渲染显示输出

> 由于酒馆会自动将不认识的HTML标记自动进行转义，因此需要将其取消转义才能顺利执行

- 仅修改显示的**HTML**代码，不修改原始消息内容

> 因此，为避免楼层内的`<% ... %>`语句块在发送到LLM时重复执行，需要通过**正则表达式**来隐藏内容
>
> ```json
> {
>     "id": "a8ff1bc7-15f2-4122-b43b-ded692560538",
>     "scriptName": "楼层函数调用过滤",
>     "findRegex": "/<%.*?%>/g",
>     "replaceString": "",
>     "trimStrings": [],
>     "placement": [
>         1,
>         2
>     ],
>     "disabled": false,
>     "markdownOnly": false,
>     "promptOnly": true,
>     "runOnEdit": true,
>     "substituteRegex": 0,
>     "minDepth": null,
>     "maxDepth": null
> }
> ```
>
> 可以使用此**正则表达式**来隐藏聊天消息内的`<% ... %>`语句块

- 代码高亮与此扩展发生冲突

> 由于代码高亮会修改实际的HTML代码，在`<`、`>`以及`%`之间插入额外的HTML标记，导致此扩展无法正确地处理内容，因此会导致代码块内的`<% ... %>`无法执行

---

## 词符计数

由于此扩展会改变实际的 **词符（tokens）** 数量，因此酒馆内置的**词符计数**会与**实际词复数**不同

因此，此扩展会在每次生成开始时，以及接收LLM相应后，设置一些**全局变量**表示处理后的**词符（tokens）**

- `LAST_SEND_TOKENS`上次生成发送的词符（tokens）数量

- `LAST_SEND_CHARS`上次生成发送的文本长度

	**以下并非实际输出消耗的词符（tokens）数量，应该以酒馆内置的词符计数为准**

- `LAST_RECEIVE_TOKENS`上次生成输出的词符（tokens）数量

- `LAST_RECEIVE_CHARS`上次生成输出的文本长度

### 上下文词符预算

由于此扩展会改变实际的**词符（tokens）**数量，因此会导致**世界/知识书**的**上下文百分比**、**Token预算上限**以及**上下文长度（以词符数计）**无法正确计算

会导致预测**词符（tokens）**数量比实际**词符（tokens）**数量要大得多，导致预算不足，部分**世界/知识书**被丢弃

而使用`getwi`、`getvar`、`getchr`等方式导入的**提示词**也不会被计数，也可能导致超出预算

[Context % / Budget](https://docs.sillytavern.app/usage/core-concepts/worldinfo/#context---budget)

---

## 范围转义

在`<#escape-ejs>...<#/escape-ejs>`内的 `<%`和`%>`将会被自动替换为`<%%`和`%%>`

例如，输入：

```html
<%= 'line 1' %>
<#escape-ejs>
<%= 'line 2' %>
<#/escape-ejs>
<%= 'line 3' %>
```

进行处理后，将会输出

```html
line 1

<%= 'line 2' %>

line 3
```

---

## 设置选项

各个设置选项的说明

- Enable Prompt template


总开关，控制是否开启此扩展

- Generate-time evaluation

是否开启对LLM生成时的提示词进行处理

世界书、预设、角色卡数据等功能均在此处进行处理

- [GENERATE] evaluation
- [RENDER] evaluation

开启对应的[提示词注入](#提示词注入)功能

- Chat message evaluation

对聊天消息（显示内容）进行处理

- Evaluate inside a code block

对聊天消息（显示内容）中的`<pre>`标签内部进行处理

非必要建议保持关闭，因为与其他扩展存在冲突

- Evaluate raw message (AND SAVE)

是否允许直接对原始聊天消息内容进行处理

**原始消息内容未经过正则、宏处理**

顺序：

1. 生成结束（并显示出来）/编辑消息后/滑动消息/打开角色卡
2. 执行此功能（`Evaluate raw message (AND SAVE)`）
3. 对显示内容进行处理（`Chat message evaluation`）

> 此功能需要开启`Chat message evaluation`才会生效

- Enable activewi to take effect this time

是否允许 `activewi`以及其别名函数能够在同一次生成时生效

禁用后可减少生成时间消耗，但是`activewi`及其别名函数只会在下次生成时才生效

- Save variables after updating

当此扩展处理完毕并更新了变量后对聊天消息进行保存

- Preload world info

在打开角色卡时立即加载（预热）需要的世界书

并立即对世界书进行处理

> 预先加载可以减少首次生成的消耗时间，但是会导致打开角色卡变慢

这个功能主要是用来加载`define`函数的

- Use strict mode

在处理模板代码时使用 Javascript 的「严格模式」

具体变化见[严格模式](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Strict_mode)

- Enable debug logging

开启调试日志，开启后此扩展将会在控制台输出大量调试日志





























