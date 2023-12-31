有的时候 llm 会生成结构尚可的 json，大差不差，但在最后一步还是会出现一些问题，比如我们现在有下面的原始语料：

```
ChatGPT的通用性为何做得如此之好？

涌现是复杂系统中的概念，如果一项能力没有在较小的模型中出现，但出现在较大的模型中，则称为涌现。涌现能力是通用性的根源。Google做了一个关于instruction tuning的非常关键的工作叫做FLAN**[1]**，其中针对“新意图识别能力”的涌现条件是：模型大小到达一定规模（比如65B），instruction任务的类型到达一定数量（比如40）。这篇文章的一作已经去OpenAI了。实际现在的一种主流观点是，不再对instruction任务的类型数量作限制，将模型大小达到10B及以上作为涌现的条件。更好地参考文章见这里**[6]**。

为什么面向对话的微调没有遭遇灾难性遗忘问题？  

对于ChatGPT而言，基于GPT-3.5，分别在SFT阶段和RLHF阶段完成了两次微调，前者是对问答任务的微调，后者是完成答案排序。但是微调之后的模型没有拟合在问答任务上，依然具备各项通用能力。比较容易想到的是，基座模型足够大，但是微调数据较少，故不会显著影响基座模型的通用能力。从另外一个角度，在微调阶段，除了问答任务，还有其他的比如代码生成，翻译，摘要等多样性的任务。这同样启发我们，针对ChatGLM-6B完成SFT阶段之后的对话能力遗忘的问题，也可以通过类似的方式缓解。

ChatGPT的大范围上下文连续对话能力是如何做到的？

可能来自两个方面的原因。SFT阶段的高质量多轮对话数据，RLHF阶段提升了回复质量，用间接的方式提升了多轮的一致性。在张家俊老师看来，还存在对较长Token的显式建模能力，比如8192，人类在一次对话过程中很难超出这个长度。但是实际测试过程中，对于超过这个范围的上下文也能得到还不错的理解。

ChatGPT的交互修正能力是如何炼成的？

在与ChatGPT交互过程中会发现，无论是用户更改自己之前的说法还是指出ChatGPT的回复中存在的问题，ChatGPT都能够捕捉到修改意图，并准确识别出需要修改的部分，最后能够做出正确的修正。ChatGPT不可能具备实时在线学习能力，一方面是模型太重，学不动。另一方面，由于是来自C端用户的反馈，在无法绝对保证准确的反馈输入前，模型在学习上要保守一些，万一被教坏了呢？针对这个现象，张老师给出了3点解释：

（1）OpenAI人工构建的对话数据中包含一些交互修正的案例，微调后拥有了这样的能力；

（2）人工反馈的强化学习使得模型输出更加符合人类偏好，从而在信息修正这类对话中表现得更加遵循人类的修正意图；

（3）可能大模型达到一定规模（e.g. 60B）之后，原始训练数据中的交互修正案例就被学到了，模型交互修正的能力自然就涌现出来了

ChatGPT的逻辑推理能力是如何学到的？

目前的主流观点是来自代码学习。通过混合代码和文本训练模型，也许代码注释建立了代码和文本之间的联系，使得模型习得强大的推理能力。而对于ChatGPT展现出来的multilingual能力，比如在中文任务上的能力，可能训练数据中存在的中英文对照在发挥着巨大的作用。

ChatGPT是否针对不同下游任务采用不同的解码策略？

张家俊老师给出的一个观察是：“对比不同类型的任务时，我们会发现ChatGPT的回复多样性针对不同下游任务差别比较大。针对“如何”、“为什么”等“How”、“Why”型任务时，重新生成的回复与之前的回复无论是表达方式还是具体内容具有较大差异，针对机器翻译、数学应用题等“What”型任务时，不同回复之间的差异非常细微，有时几乎没有变化。”，这里，我们倾向于相信ChatGPT能够学习到任务相关的非常理想的概率分布，也就是说，基于采样的解码策略就可以适用于所有任务。

ChatGPT能否解决事实可靠性问题？

目前不能。如果希望ChatGPT解决事实回答的可靠性问题，可能需要进一步提升模型的拒识能力，也就是过滤掉模型确定无法回答的那些问题，同时还需要事实验证模块来验证ChatGPT回复的正确性。实际上，目前的一个做法是借助搜索引擎的能力，基于搜索引擎返回的知识，ChatGPT做总结。在解决事实可靠性的同时，也能一定程度上解决知识和信息的时效性问题。不过，这里的问题是，不是所有的知识都能被搜索引擎检索到，这是另外一个话题了。

ChatGPT能否实现实时信息的学习？

难也没必要。比较实际的策略是，一方面每隔一段时间用最新的数据更新模型，另一方面，基于voting机制实现模型的在线实时学习能力，不过技术挑战依然非常大。

整体上梳理下来会发现在底层基础知识上会存在非常多有意思的问题，同时没有解决的问题也非常多。对于大模型，对于ChatGPT，我们似乎依然不甚理解。这也许就是这件事情的迷人之处～
```

出自 [一个有意思的关于ChatGPT的问题清单](https://mp.weixin.qq.com/s/L9nxZ1P0hu450DIJXDGprw)

我们现在要求 OpenAI 做 json 提炼。

`现在我有如下的文档 {content} ，请你给我总结一下其中的问题，以json形式返回，json格式要求如下,只有一个字段questions，它的值是一个字符串数组，数组中的每个元素就是对应的问题`

或者这样写也行

`请你给我总结一下其中的问题，以json形式返回，json格式要求如下

```json
{  
"questions": ["问题1", "问题2"...]  
}
```
`

……一个尴尬的事实是，现在是 2023年8月21日，曾经的 OpenAI 不老老实实给 json 的问题我试了几次都不复现了，现在给的就是我预期的格式……

```json
{
  "questions": [
    "ChatGPT的通用性为何做得如此之好？",
    "为什么面向对话的微调没有遭遇灾难性遗忘问题？",
    "ChatGPT的大范围上下文连续对话能力是如何做到的？",
    "ChatGPT的交互修正能力是如何炼成的？",
    "ChatGPT的逻辑推理能力是如何学到的？",
    "ChatGPT是否针对不同下游任务采用不同的解码策略？",
    "ChatGPT能否解决事实可靠性问题？",
    "ChatGPT能否实现实时信息的学习？"
  ]
}
```

好吧，曾经的 OpenAI 经常会犯这样一个错误

```text
\`\`\`json: {
  "questions": [
    "ChatGPT的通用性为何做得如此之好？",
    "为什么面向对话的微调没有遭遇灾难性遗忘问题？",
    "ChatGPT的大范围上下文连续对话能力是如何做到的？",
    "ChatGPT的交互修正能力是如何炼成的？",
    "ChatGPT的逻辑推理能力是如何学到的？",
    "ChatGPT是否针对不同下游任务采用不同的解码策略？",
    "ChatGPT能否解决事实可靠性问题？",
    "ChatGPT能否实现实时信息的学习？"
  ]
}

 ```


它会输出标准 json 的同时，一定要给你多加一点字符串，来表明自己的结构，我可以理解它从网上都学了些什么，而且这个问题其实很好解决，我们完全可以在获得数据后手动处理一下，做一点简单的正则就行（如果你说结果不符合正则怎么办？那就打回重练）。

其实我们也可以还是在提示词里做一点点小小的手脚就行，一点点。

我个人常用的 prompt 如下：

```text
....输出结果为纯json，不带任何其他字符，总计不超过 800字.你的返回结果应该直接以json的'{'格式开始，前面不需要带任何东西，以json的最后一个'}'作为结束。
```

就这样就可以了，在2023年8月份之前 OpenAI 经常有大量的 json 输出的错误，上面给的这个方法虽然简单粗暴，但很明确的给了一个确定无比的约束，我试下来目前觉得效果还不错。
