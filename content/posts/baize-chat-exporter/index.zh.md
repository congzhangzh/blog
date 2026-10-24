---
title: "白泽：把 AI 对话导出为 Markdown，以及跟上聊天界面改版要付出什么"
date: 2026-10-24
summary: "白泽是一个小巧的 Chromium 扩展，在本地把 ChatGPT、Claude、Gemini 等十三个站点的对话导出为干净的 Markdown。本文介绍它能做什么，以及 ChatGPT 的一次改版如何在一天之内让它坏了两次。"
tags: ["baize", "browser-extension", "markdown", "chatgpt", "claude", "gemini", "debugging"]
---

{{< video src="baize-trailer.mp4" poster="baize-trailer.jpg" caption="白泽把你和各个 AI 的对话，化作一页属于你自己的文档交还给你。由 Gemini（Veo）生成；完整的四个镜头见项目 README。" >}}

我每天都会和好几个 AI 对话，有价值的内容散落在 ChatGPT、Claude、Gemini 等各个地方，
而它们都没有提供好用的方式把对话留下来。[白泽](https://github.com/congzhangzh/baize-chat-exporter)
就是我的答案：一个小巧的 Chromium 扩展，一键把一段对话导出为一个干净的 Markdown 文件。
一切都在你的浏览器里完成：不用注册，没有服务器，没有遥测。

## 名字的由来

白泽是《山海经》传统中通晓万物的神兽。相传它向黄帝讲述了天下一万一千五百二十种精怪的知识，
黄帝命人记录成《白泽图》，这可以说是历史上第一次"知识导出"。这个扩展为你的 AI 对话做同样的事：
**汲百家所长，长出你自己的智慧。**

## 实际效果

{{< video src="demo.mp4" poster="demo.jpg" caption="真实的 Chrome 和真实的扩展在导出一段对话。只有聊天网站是模拟的，不会调用任何 AI 服务。" >}}

你能得到：

- **十三个站点，均已测试**：ChatGPT、Claude、Gemini、Grok、Kimi、豆包、智谱清言、
  腾讯元宝、Perplexity、HuggingChat、Meta AI、WhatsApp Web，以及自托管的 OpenWebUI。
- **可读的 Markdown**：数学公式还原为 `$...$` / `$$...$$`，代码保留为代码块，
  Claude 的扩展思考折叠进 `<details>`。
- **图片一并保存**：以 data URI 嵌入文件（一个文件，粘贴到哪里都行），
  或者打包成 ZIP，Markdown 旁边带一个 `images/` 文件夹。
- **隐私优先**：只读取页面已经在你自己浏览器里渲染出来的内容。只有在你添加自托管站点时，
  才会逐个来源申请额外的站点权限。

## 它是怎么构建的

每个站点对应一个小小的适配器文件 `content/adapters/<site>.js`。适配器只回答几个问题：
对话在哪里、每一轮从哪里开始、是谁说的、怎么滚动才能加载剩下的内容。其余的事情都由框架完成：
HTML 转 Markdown、图片、数学公式、下载。

这样拆分很重要，因为适配器会"腐烂"。聊天网站不停地改 DOM，往往悄无声息，而且常常按账号灰度。
某个类名一变，`querySelectorAll` 返回空列表，用户只看到"未找到对话内容"，完全不知道原因。
所以整个项目围绕一个目标来设计：让这类故障大声报错，并且能很快修好。

- **每个关键锚点至少两层后备选择器**，设置 `localStorage.AI_EXPORTER_DEBUG = '1'` 后，
  每一层都会打印命中数量。
- **结构探针**：一段贴进页面控制台的脚本，打印每个候选选择器各匹配到多少节点。
  不管眼下哪个选择器有效，"还有什么能匹配上？"这个问题永远有答案。
  探针才是真正的资产，选择器是一次性的。
- **故障历史写在源码里。** 每个不简单的修复都会在适配器文件头部加一条带日期的记录：
  症状、根因、修复。调试时没人去翻 `git log`，但每个人都会读自己要改的那个文件的开头。

## 案例：一天之内，ChatGPT 坏了两次

2026 年 9 月 28 日收到一个问题报告："ChatGPT 上不能用了。"可在我的测试账号上一切正常。

**同一个网站，不同的页面。** 报告者用的是 Plus 账号，ChatGPT 给这个账号推送了一套全新的
app-shell 布局，Chat 和 Work 两种模式都是；免费账号仍然是旧布局。在一个 Work 对话上运行探针，
结果一目了然：所有旧锚点都是 0，适配器依赖的滚动容器也不见了。
第一条经验：问题复现不了时，先怀疑布局不同，而不是用户操作有误，并问清楚是哪个账号、哪种套餐。

**一个倒着滚的滚动条。** 新的对话列表是 `flex-direction: column-reverse`。
它的 `scrollTop` 在**底部**为 0，越往上越负。旧代码用 `Math.max(0, scrollTop - step)`
往上滚，结果永远不会小于 0，所以根本没动过。修复方法是为新布局加一套锚点，
并让滚动边界知道哪边是"上"。线上验证：一个 Work 对话导出 16/16 轮，一个 Chat 模式对话
10/10 轮，免费账号的旧布局照常工作。

**然后它停不下来了。** 同一天又来了第二个报告：导出"在顶部无限重试"。原因只是零点几个像素。
在 1.40625 的显示缩放下，滚动条停在 `scrollTop` −4989.87，而计算出的边界是 −4991。
于是判断 `scrollTop > min + 1` 永远认为"还没到顶"，把整个时间预算都耗在了那里。
更糟的是，ChatGPT 按需加载更早的消息，一次加载如果错过了触发时机就再也不会重试：
"Loading older messages…"连续 60 个周期纹丝不动，只有真实的鼠标滚轮事件才能加载更多。
修复：2 像素的容差，加上"滚动条到底动没动"的判断；基于时间的空闲退出；
等待时从顶部轻推一点；向上和向下两遍各用独立的时间预算。

一旦证据摆在屏幕上，这些问题都不难。没有探针和调试日志的话，每一个都要靠猜上一下午。

## 试一试

白泽在 [congzhangzh/baize-chat-exporter](https://github.com/congzhangzh/baize-chat-exporter) 开源。
Chrome 网上应用店的上架正在准备中，在此之前：

1. `git clone https://github.com/congzhangzh/baize-chat-exporter`
2. 打开 `chrome://extensions`（或 `edge://extensions`），打开**开发者模式**。
3. 点击**加载已解压的扩展程序**，选择包含 `manifest.json` 的文件夹。

支持 Chrome、Edge、Brave、Opera、Vivaldi 和 Arc，暂不支持 Firefox。

## 谁维护什么

先把预期说清楚：我维护的是我自己在用的站点和使用场景，以及大家共用的部分：框架、探针，
还有对贡献的评审。**其他站点都归社区所有。** DeepSeek、通义千问、Mistral Le Chat、
Microsoft Copilot，以及 [README](https://github.com/congzhangzh/baize-chat-exporter#coming-soon)
里"想要支持"清单上的其余站点，要等用它们的人来写适配器才会有；对于我不用的站点，
已有适配器的日常维护也是一样。

这件事没有听起来那么难。一个适配器大约 100 行，而每天用某个站点的人，总是最先发现它坏了的人。
探针存在的意义，就是让这个人不必读懂整个代码库，也能找到原因。

## 你可以怎样帮忙

- **新增或认领一个站点。** 一个新站点就是一个适配器文件；
  [贡献指南](https://github.com/congzhangzh/baize-chat-exporter/blob/main/CONTRIBUTING.md)
  会一步步带你完成，第一步就是跑探针。认领一个站点，意味着它坏了的时候由你来检查。
- **导出出问题时开一个 issue**，写明站点、账号类型，以及导出时标签页是否在前台。
  把探针输出的表格贴进 issue，通常就能直接指向原因。
- **自己写不了？** 可以[赞助一个站点](https://github.com/congzhangzh/baize-chat-exporter#sponsor-a-site)，
  让它在我的队列里排到前面。赞助只影响先后顺序，不影响许可证：白泽始终是 MIT，没有付费功能。
