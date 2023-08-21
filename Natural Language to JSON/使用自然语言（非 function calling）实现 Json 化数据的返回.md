OpenAI 提供的 function calling 功能，很多时候被我用来实现 NER 实体提取，但除此之外还有很多种方法来实现一个 json 的输出。

这些方法很难保证 100% 的有效，针对不同的 llm 模型也要使用不同的提示词才能得到相对最好的结果。不过这里面还是有一些技巧的。

通常来说如果想要获得一个稳定的 json 输出，我们可以遵循如下步骤

1. 用结构化的方式来描述出你需要的 json 结构是什么样，包括每个字段的名字 key 应该叫什么，它的含义是什么。
2. 可以配合 yaml 等格式强化对你预期的结构的描述。
3. 给 llm 打样，教会它。
4. 强制约定格式输出。
5. 在获得输出后进行校验和筛查过滤，简单的问题直接处理，复杂的，或者回答牛头不对马嘴的，打回重来。

为什么要用 Json？
可以通过自然语言从大量文档中进行实体提取，获取数据。
和现有系统对接， Json 是几乎唯一的也是事实上的通用数据格式标准（我知道还有其他的）。

适用场景
使用自然语言控制系统：通过自然语言发布指令来取代手动操作
自然语言产生结构化数据： 例如从 PRD → 流程图。


# 大致步骤

1. 使用类似下面的 prompt 来提问：

你的任务是将一段自然语言描述的工作转化为一个可以完成该工作的步骤列表，可用的步骤类型是有限的，包括：

```python
- name: 添加节点
  key: add_node
  params:
    - name: <节点名称>
    - type: <节点类型:start|end|eval|condition>
    - condition: <分支条件>
- name: 删除节点
  key: delete_node
  params:
    - name: <要删除的节点的名称>
- name: 修改节点
    key: edit_node
    params:
        - name: <要修改的节点的名称>
        - modifyKey: <要修改的节点的参数名>
        - modifyValue: <要修改为的参数值>
```

你需要将转化后的步骤列表以 json 数组格式输出，数组中每一个元素为一个步骤，包括步骤 key 和必要的 params 。 例如当自然语言描述为：

`请添加一个开始节点，名字为 entry 。`

你的输出应该是：

```javascript
[{
  key: add_node,
  params: {
    name: "entry"
    type: "start"
  }
}]
```

自然语言任务描述为：

`修改 condition 节点，将分支条件修改为 c == a + 1。然后添加一个新的运行节点，名字是 gogogo 。`

你的输出：

2. 此时我们会得到这样的输出：

```javascript
[{
  key: edit_node,
  params: {
    name: "condition",
    modifyKey: "condition",
    modifyValue: "c == a + 1"
  }
}, {
  key: add_node,
  params: {
    name: "gogogo",
    type: "eval"
  }
}]
```

3. 解析这个结构化输出并调用 App 的 API 完成工作

# 扩展

- prompt 还有迭代调优空间，例如明确的告诉 ChatGPT 当无法完成任务时，输出 error
- 当系统可枚举的行为数量太多无法一次放进 prompt 中时有两种办法
    
    1. 将 copilot 功能按场景分开，例如编辑文档时和编辑流程图时使用分开的入口。
        
        2. 将结构化选项描述向量化，从任务描述经过向量相关性计算后仅放入部分结构化选项。但这种方式恐怕精度会有问题，未必可用，可能可以考虑 Claude 的 100K 上下文。
        
    
- 结构化输出后可以嵌套调用 ChatGPT 。例如 “为第二张 PPT 补充具体内容” 的指令，可在转化为类似 `{task: "fill_content", index: 2}` 这样的结构后，在执行 fill_content 任务的时候再次调用 ChatGPT 来实际产生需要补充的内容
- 得到结构化输出之后的两个附带的好处
    
    1. 可以给用户预览接下来要实际执行的动作，由用户 confirm 或调整后再执行
        
        2. 可以 undo / redo 单步动作