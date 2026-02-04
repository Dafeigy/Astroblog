---
title: 内网环境下的AI生产力重构实践
published: 2026-02-04
description: '尝试重构生产力ing...'
image: 'https://pic1.imgdb.cn/item/66968691d9c307b7e95c57f0.png'
tags: ["AI工作流","AI编程","生产力"]
category: 'AI'
draft: false 
lang: 'zh-cn'
---



## 前言

2025年初Deepseek R1的横空出世把开源精神发挥到了极致，国产大模型以极高的性价比表现横扫了抢占了市面闭源模型的市场。Deepseek R1的开源是具有划时代意义的，这种意义可能不在于技术上，而是在个人发展影响上。它的出现确保了“普通人”能使用到的高新技术的下限，让“科技平权”不再是一个空谈的口号，每个人都可以基于极致性价比的大模型重塑自己的知识体系（前提是他们愿意花费时间去研究与学习），进一步提高自己的认知水平。虽然说开源模型和闭源模型的差别对日常接触开发、代码编写工作的我们来说是心知肚明的，但我希望自己还能保持一些折腾的热情：于私，这是填补我好奇心的手段；于公，是积累一些浅薄的经验供后来者参考。

说回正题吧。假设你在一个算力受限、网络访问受限的工作环境里，你要怎么去重塑你的生产力？我希望可以不断迭代这篇文章，并分享我找到的自认最佳的实践。

## 编程

码农离不开的工具除了键盘就是集成开发(Integrated Development Environment, IDE)了。我多年来都是使用Visual Studio Code，这个被大部分人认为是宇宙第一的编辑器。这个工具的优点是：

- **大厂背书，持续开源。**“大厂”一词形容现在的微软可能有点令人欲言又止，但是你不能否认这个缔造了几十亿用户的公司的技术积累与产品打磨。在这里，“大厂”代表了一种信任背书，大家都可以较为放心的使用，不用担心数据泄露的问题，这对涉及敏感信息的编码工作的人来说，非常重要。
- **生态活跃，插件丰富。** VSCode的插件可谓是琳琅满目，美化的、效率的、辅助的各式各样的都有。作为全世界使用人数最多的IDE，你不用质疑它的生态活跃程度，新技术降临时它也肯定是最先尝试结合重构生产力的IDE。
- **上手简单，轻松编码。** Visual Studio Code 包含一个交互式调试器，因此你可以逐步执行源代码、检查变量、查看调用堆栈并在控制台中执行命令，同时VS Code 还与构建和脚本工具集成，以执行常见任务，从而使日常工作流程更快。

因此，它是会作为一个关键的工具参与到接下来的编程工作流中的。事实上，很多的AI IDE工具，如字节跳动的TRAE、腾讯的Code Buddy等，他们都是基于VSCode进行二次开发的。在我的构想的实践中，将会使用较为成熟的AI辅助编程插件与VSCode结合完成AI编程工作流的编排。

> 你可能会困惑为什么不直接用TRAE的企业版本以实现本地部署，额...有些公司可能不太想那么快掏这笔钱吧！

另一种AI编程工具的载体则是CLI命令行式的，比如Claude Code，Open Code这种。这种我没有具体的使用经验，之后等二番战的时候再来研究一下！先把绝大多数人的应用场景给解决，再来研究如何提升上限。

### 模型选择

选择使用阿里再2026年2月4日推出的 [Qwen3-Coder-Next-80B](https://www.modelscope.cn/models/Qwen/Qwen3-Coder-Next) 作为核心的编程模型吧！这是千问推出的最新模型，从SWE-Benchmark的结果来看，似乎效果是和头部的开源模型坐一桌的：

![c1fd64511e27bb31629f31341c6a7eb7](https://s2.loli.net/2026/02/04/cibE1QO3m8TGB7u.png)

我研究了一下[技术报告](https://github.com/QwenLM/Qwen3-Coder/blob/main/qwen3_coder_next_tech_report.pdf)，似乎是对我选用的[Cline](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev)插件是有优化的。至于模型怎么调用，我推荐是使用ModelScope

### 插件选择

[Cline](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev)。基本上没有别的选择。我早年在读研的时候还尝试折腾过Continue和Roo Code，二者都不是Cline的水平。Cline是针对Claude的模型进行优化过的，但是Qwen3也针对这一插件进行了优化。

