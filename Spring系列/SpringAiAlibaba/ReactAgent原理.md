ReactAgent原理：

Spring AI Alibaba 中的`ReactAgent` 基于 Graph 运行时构建。Graph 由节点（steps）和边（connections）组成，定义了 Agent 如何处理信息。Agent 在这个 Graph 中移动，执行如下节点：

*   Model Node (模型节点)：调用 LLM 进行推理和决策
*   Tool Node (工具节点)：执行工具调用
*   Hook Nodes (钩子节点)：在关键位置插入自定义逻辑



这里的节点（steps）可以理解成activiti审批流里的action，节点实例，真正干活的实例

边（connections）可以理解成activiti审批流里的连线，表示从谁到谁，source -> target

内部实现将任务拆解成一个个明确的步骤（节点），然后用代码把这些步骤“编排”成一个流程图，使用代码描述如下：

```java
package com.demo.test.graph;

import static com.alibaba.cloud.ai.graph.action.AsyncNodeActionWithConfig.node_async;

import com.alibaba.cloud.ai.graph.CompiledGraph;
import com.alibaba.cloud.ai.graph.KeyStrategy;
import com.alibaba.cloud.ai.graph.OverAllState;
import com.alibaba.cloud.ai.graph.RunnableConfig;
import com.alibaba.cloud.ai.graph.StateGraph;
import com.alibaba.cloud.ai.graph.agent.exception.AgentException;
import com.alibaba.cloud.ai.graph.exception.GraphStateException;
import com.alibaba.cloud.ai.graph.state.strategy.AppendStrategy;
import com.alibaba.cloud.ai.graph.state.strategy.ReplaceStrategy;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.messages.Message;

/**
 * 理解 Spring Ai Alibaba 的流程编排
 *
 * @author wuzhenhong
 * @date 2026/3/31 15:56
 */
public class TestGraph {

    public static void main(String[] args) throws GraphStateException {
        // 1. 定义全局状态
        OverAllState state = new OverAllState();
        state.registerKeyAndStrategy("topic", new ReplaceStrategy());
        state.input(Map.of("topic", "undefined"));

        // 2. 定义流程中的各个节点
        StateGraph stateGraph = new StateGraph("Research Workflow", () -> {
            HashMap<String, KeyStrategy> keyStrategyHashMap = new HashMap<>();
            keyStrategyHashMap.put("messages", new AppendStrategy());
            return keyStrategyHashMap;
        })
            // 节点1：信息搜集节点，内部封装了调用搜索工具的LLM
            .addNode("research_node", node_async(((s, config) -> {
                System.out.println("research_node........");
                // 如果 Windows 中文乱码，设置 -Dfile.encoding=GBK 到启动参数中
                return Map.of("topic", "您好！hello world!");
            })))
            // 节点2：报告撰写节点，内部封装了调用文档工具的LLM
            .addNode("writing_node", node_async((s, config) -> {
                Object topic = s.data().getOrDefault("topic", "undefined");
                System.out.println(String.format("invoke LLM with topic %s........", topic));
                String javaStr = """
                    public class HelloWorld {
                                        
                        public static void main(String[] args) {
                            System.out.println("%s");
                        }
                                        
                    }
                    """.formatted(topic);
                List<Message> messageList = new ArrayList<>();
                AssistantMessage assistantMessage = AssistantMessage.builder()
                    .content(javaStr)
                    .build();
                messageList.add(assistantMessage);
                return Map.of("messages", messageList);
            }))
            // 节点3：结束节点
            .addNode("finish_node", node_async((s, config) -> {
                System.out.println("finish_node........");
                return Map.of();
            }))

            // 3. 编排节点间的执行路径
            .addEdge(StateGraph.START, "research_node")
            .addEdge("research_node", "writing_node")
            .addEdge("writing_node", "finish_node")
            .addEdge("finish_node", StateGraph.END);

        RunnableConfig config = RunnableConfig.builder()
            .threadId("testsssssssssss1111")
            .build();

        // 4. 执行工作流
        CompiledGraph compiledGraph = stateGraph.compile();
        AssistantMessage assistantMessage = extractAssistantMessage(compiledGraph.invoke(state, config));
        System.out.println(assistantMessage.getText());
    }

    private static AssistantMessage extractAssistantMessage(Optional<OverAllState> state) {
        return state.flatMap(s -> s.value("messages"))
            .stream()
            .flatMap(messageList -> ((List<?>) messageList).stream()
                .filter(msg -> msg instanceof AssistantMessage)
                .map(msg -> (AssistantMessage) msg))
            .reduce((first, second) -> second)
            .orElseThrow(() -> new AgentException("No AssistantMessage found in 'messages' state"));
    }

}


```

启动 main 方法执行结果如下：

```java
research_node........
invoke LLM with topic hello world!........
finish_node........
public class HelloWorld {

    public static void main(String[] args) {
        System.out.println("hello world!");
    }

}
```



ReactAgent请求流程可以描述成下面这样：

框内的id相同的表示同一个对象


<img src="../../images/SpringAiAlibaba/ReactAgentExecFlow.png">

从图中可以看到 AGENT_HOOK 全程只会调用一次，而 MODEL_HOOK 可能会调用多次，直到模型认为推理结果满意为止



用官方的图表示如下：

*   总览图

    <img src="../../images/SpringAiAlibaba/ReactAgentOverview.png">

*   勾子和拦截器的调用

    <img src="../../images/SpringAiAlibaba/ReactAgentHook.png">


