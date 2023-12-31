<p align="center">
   <picture>    
    <img alt="Byzer-Agent" src="./images/byzer-agent.png" width=55%>
  </picture>
</p>

<h3 align="center">
Easy, fast, and distributed agent framework for everyone
</h3>

<p align="center">
| <a href="#"><b>Documentation</b></a> | <a href="#"><b>Blog</b></a> | | <a href="#"><b>Discord</b></a> |

</p>

---

*Latest News* 🔥

- [2023/12] Byzer-Agent released in Byzer-LLM 0.1.22

---

Byzer-Agent is an distributed agent framework for LLM. It is designed to be easy to use, easy to scale, and easy to debug. It is built on top of Ray, developed from [autogen](https://github.com/microsoft/autogen), a high-performance distributed execution framework.

The code of Byzer-Agent is under the project [Byzer-LLM](https://github.com/allwefantasy/byzer-llm). So this project is just a document project.

---

* [Installation](#installation)
* [Architecture](#Architecture)
* [DataAnalysis (multi-agent)](#DataAnalysis-(multi-agent))
* [RAG Example](#rag-example)
* [DataAnalysis Example](#DataAnalysis-Example)
* [Custom Agent](#custom-agent)
* [Remote Agent](#remote-agent)

## Architecture

<p align="center">
  <img src="./images/byzer-tools.jpg" width="600" />
</p>

---


##  Installation

Install the following projects step by step.

1. [Byzer-LLM](https://github.com/allwefantasy/byzer-llm) 
2. [Byzer-Retrieval](https://github.com/allwefantasy/byzer-retrieval)

---

## DataAnalysis (multi-agent)

<p align="center">
  <img src="./images/multi-agent.jpg" width="600" />
</p>

---

## RAG Example

After you install the Byzer-LLM and Byzer-Retrieval,  make sure you have started a Byzer-Retrieval Cluster:

```python
if not retrieval.is_cluster_exists("data_analysis"):
    builder = retrieval.cluster_builder()
    builder.set_name("data_analysis").set_location("/tmp/data_analysis").set_num_nodes(2).set_node_cpu(1).set_node_memory("3g")
    builder.set_java_home(env_vars["JAVA_HOME"]).set_path(env_vars["PATH"]).set_enable_zgc()
    builder.start_cluster()  
```

you can use the following code to create a RAG agent:


```python
user.initiate_chat(
retrieval_agent,
message={
    "content":"介绍下 Gluten",
    "metadata":{
        "file_path":"/home/byzerllm/projects/jupyter-workspace/aritcle.txt"
    }
},)

```

The output:

```
user (to retrieval_agent):

介绍下 Gluten

--------------------------------------------------------------------------------
retrieval_agent (to user):

 Gluten 是一个开源项目，其目标是通过本地化 Spark 项目，以提高 Spark 在 ETL 批处理领域的性能。Gluten 项目由深耕于 Spark 加速的 Intel OAP 团队和 Kyligence 公司共建，BIGO 大数据团队于 2022 年 9 月加入研发。Gluten 项目的核心思想是使用本地的 Native 运行时库作为 Spark 插件，结合了 Spark 的可扩展性框架、优化器、容错能力和 Native 执行库的高性能，预期可以带来数倍的性能提升。

Gluten 项目主要支持 Velox 和 ClickHouse 两种 Native 执行库。Velox 是 Meta 开源的本地执行库，其目标是构建统一的查询执行层，提供了数据类型、向量数据结构、Pipeline 框架、算子、表达式计算、IO 和资源管理等功能。而 ClickHouse 是 2016 年 Yandex 公司开源的一种用于联机分析(OLAP)的列式数据库，以其高性能近年来备受关注。

BIGO 大数据团队在 2022 年 9 月加入 Gluten 项目后，已经在生产环境逐步灰度 Gluten，开始替换 Spark 的 ETL 工作负载，目前灰度 SQL 上获得了总体 40%+ 的成本节省。

```

You can continue to chat with the agent:

```python
user.initiate_chat(
retrieval_agent,
message={
    "content":"BIGO 大数据团队是什么时候加入 Gluten 项目的",
    "metadata":{        
    }
},clear_history=False)
```

Notice that, we set `clear_history` to False, so the previous conversation will be kept.

The output:

```text
user (to retrieval_agent):

BIGO 大数据团队是什么时候加入 Gluten 项目的

--------------------------------------------------------------------------------
retrieval_agent (to user):

BIGO大数据团队于2022年9月加入Gluten项目。

--------------------------------------------------------------------------------
```

The code of this example is [here](./notebooks/quick_rag.ipynb).

---

## DataAnalysis Example

* Connect to Ray Cluster

```python
import os
os.environ["RAY_DEDUP_LOGS"] = "0" 

import ray
from byzerllm.utils.client import ByzerLLM,LLMRequest,LLMResponse,LLMHistoryItem,InferBackend
from byzerllm.records import SearchQuery

ray.init(address="auto",namespace="default")   

llm = ByzerLLM()
chat_model_name = "chat"

if not llm.is_model_exist("chat"):
    llm.setup_gpus_per_worker(2).setup_num_workers(1).setup_infer_backend(InferBackend.VLLM)
    llm.deploy(
        model_path="/home/byzerllm/models/openbuddy-zephyr-7b-v14.1",
        pretrained_model_type="custom/auto",
        udf_name=chat_model_name,
        infer_params={}
    )


def show_code(lang,code_string):
    from IPython.display import display, Markdown    
    display(Markdown("```{}\n{}\n```".format(lang,code_string)))


def show_text(msg):
    from IPython.display import display, Markdown
    display(Markdown("```{}\n{}\n```".format("text",msg))) 

def show_image(content):
    from IPython.display import display, Image
    import base64             
    img = Image(base64.b64decode(content))
    display(img)    
    
```

* Create a DataAnalysis agent group


```python
import ray
from byzerllm.apps.agent import Agents
from byzerllm.apps.agent.user_proxy_agent import UserProxyAgent
from byzerllm.apps.agent.extensions.data_analysis import DataAnalysis


ray.init(address="auto",namespace="default")  

llm = ByzerLLM()
chat_model_name = "chat"
llm.setup_default_model_name(chat_model_name)  

user = Agents.create_local_agent(UserProxyAgent,"user",llm,None,
                                human_input_mode="NEVER",
                                max_consecutive_auto_reply=0)

data_analysis = DataAnalysis("chat4","william","/home/byzerllm/projects/jupyter-workspace/test.csv",
                             llm,None)


```

Notice that the chat_name and owner in DataAnalysis will be combined as the session key. When you execute the code, will trigger the 
file preview agent to load the csv file. 

Here is the conversations:

```text
use_shared_disk: False file_path: /home/byzerllm/projects/jupyter-workspace/test.csv new_file_path: /home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to privew_file_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) We have a file, the file path is: /home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv , please preview this file
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) privew_file_agent (to python_interpreter):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) Here's the Python code that meets your requirements:
(DataAnalysisPipeline pid=2134293) ```python
(DataAnalysisPipeline pid=2134293) import pandas as pd
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) file_path = "/home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv"
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) try:
(DataAnalysisPipeline pid=2134293)     # Read the file based on its suffix
(DataAnalysisPipeline pid=2134293)     if file_path.endswith(".csv"):
(DataAnalysisPipeline pid=2134293)         df = pd.read_csv(file_path)
(DataAnalysisPipeline pid=2134293)     elif file_path.endswith(".xlsx") or file_path.endswith(".xls"):
(DataAnalysisPipeline pid=2134293)         df = pd.read_excel(file_path)
(DataAnalysisPipeline pid=2134293)     else:
(DataAnalysisPipeline pid=2134293)         raise ValueError(f"Unsupported file type: {file_path}")
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293)     # Set the flag to indicate successful loading
(DataAnalysisPipeline pid=2134293)     loaded_successfully = True
(DataAnalysisPipeline pid=2134293)     
(DataAnalysisPipeline pid=2134293)     # Show the first 5 rows of the file
(DataAnalysisPipeline pid=2134293)     file_preview = df.head()
(DataAnalysisPipeline pid=2134293) except Exception as e:
(DataAnalysisPipeline pid=2134293)     # Set the flag to indicate failed loading
(DataAnalysisPipeline pid=2134293)     loaded_successfully = False
(DataAnalysisPipeline pid=2134293)     
(DataAnalysisPipeline pid=2134293)     # Show an error message
(DataAnalysisPipeline pid=2134293)     file_preview = f"Error occurred while loading the file: {str(e)}"
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) # Print the result
(DataAnalysisPipeline pid=2134293) print(file_preview)
(DataAnalysisPipeline pid=2134293) ```
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) In this code, we first define the `file_path` variable to store the path of the file to be previewed. Then, we use a `try`-`except` block to handle possible exceptions during file loading.
(DataAnalysisPipeline pid=2134293) Inside the `try` block, we check the suffix of the file path to determine how to read the file. If the file is a CSV file, we use the `pd.read_csv()` function to load it into a DataFrame. If the file is an Excel file (either .xlsx or .xls format), we use the `pd.read_excel()` function to load it. If the file has an unsupported suffix, we raise a `ValueError` exception.
(DataAnalysisPipeline pid=2134293) If the file is loaded successfully, we set the `loaded_successfully` flag to `True`, and use the `head()` method of the DataFrame to get the first 5 rows of the file, which is stored in the `file_preview` variable.
(DataAnalysisPipeline pid=2134293) If any exception occurs during file loading, we set the `loaded_successfully` flag to `False`, and store an error message in the `file_preview` variable.
(DataAnalysisPipeline pid=2134293) Finally, we print the contents of the `file_preview` variable to show the result of file previewing.
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) python_interpreter (to privew_file_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) exitcode: 0 (execution succeeded)
(DataAnalysisPipeline pid=2134293) Code output:    ID   Deaths  Year                 Entity
(DataAnalysisPipeline pid=2134293) 0   1  1267360  1900  All natural disasters
(DataAnalysisPipeline pid=2134293) 1   2   200018  1901  All natural disasters
(DataAnalysisPipeline pid=2134293) 2   3    46037  1902  All natural disasters
(DataAnalysisPipeline pid=2134293) 3   4     6506  1903  All natural disasters
(DataAnalysisPipeline pid=2134293) 4   5    22758  1905  All natural disasters
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) privew_file_agent (to data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) ID,Deaths,Year,Entity
(DataAnalysisPipeline pid=2134293) 1,1267360,1900,All natural disasters
(DataAnalysisPipeline pid=2134293) 2,200018,1901,All natural disasters
(DataAnalysisPipeline pid=2134293) 3,46037,1902,All natural disasters
(DataAnalysisPipeline pid=2134293) 4,6506,1903,All natural disasters
(DataAnalysisPipeline pid=2134293) 5,22758,1905,All natural disasters
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) sync the conversation of preview_file_agent to other agents
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to assistant_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) We have a file, the file path is: /home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv , please preview this file
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to assistant_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) ID,Deaths,Year,Entity
(DataAnalysisPipeline pid=2134293) 1,1267360,1900,All natural disasters
(DataAnalysisPipeline pid=2134293) 2,200018,1901,All natural disasters
(DataAnalysisPipeline pid=2134293) 3,46037,1902,All natural disasters
(DataAnalysisPipeline pid=2134293) 4,6506,1903,All natural disasters
(DataAnalysisPipeline pid=2134293) 5,22758,1905,All natural disasters
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to visualization_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) We have a file, the file path is: /home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv , please preview this file
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to visualization_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) ID,Deaths,Year,Entity
(DataAnalysisPipeline pid=2134293) 1,1267360,1900,All natural disasters
(DataAnalysisPipeline pid=2134293) 2,200018,1901,All natural disasters
(DataAnalysisPipeline pid=2134293) 3,46037,1902,All natural disasters
(DataAnalysisPipeline pid=2134293) 4,6506,1903,All natural disasters
(DataAnalysisPipeline pid=2134293) 5,22758,1905,All natural disasters
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
```

2. Chat

```python
data_analysis.analyze("根据文件统计下1901年总死亡人数")
```

Here is the console output:

```text
(DataAnalysisPipeline pid=2134293) user_data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 根据文件统计下1901年总死亡人数
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) Select agent: assistant_agent to answer the question: 根据文件统计下1901年总死亡人数
(DataAnalysisPipeline pid=2134293) data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac (to assistant_agent):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) 根据文件统计下1901年总死亡人数
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
(DataAnalysisPipeline pid=2134293) assistant_agent (to python_interpreter):
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) ```python
(DataAnalysisPipeline pid=2134293) # filename: stats.py
(DataAnalysisPipeline pid=2134293) import pandas as pd
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) def get_total_deaths_year(year):
(DataAnalysisPipeline pid=2134293)     df = pd.read_csv("/home/byzerllm/projects/jupyter-workspace/data_analysis_pp_e61639d1e6e6504af87495b8bf80ecac.csv")
(DataAnalysisPipeline pid=2134293)     total_deaths = df[df["Year"] == year]["Deaths"].sum()
(DataAnalysisPipeline pid=2134293)     return total_deaths
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) total_deaths_1901 = get_total_deaths_year(1901)
(DataAnalysisPipeline pid=2134293) print(f"The total number of deaths in 1901 is {total_deaths_1901}.")
(DataAnalysisPipeline pid=2134293) ```
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) Run the above Python script to calculate the total number of deaths in 1901.
(DataAnalysisPipeline pid=2134293) 
(DataAnalysisPipeline pid=2134293) --------------------------------------------------------------------------------
```

3. Output

You can also use `output` function to get the result:

```python
o = data_analysis.output()
show_image(o["content"])
```

Here is value:

```text
exitcode: 0 (execution succeeded)
Code output: The total number of deaths in 1901 is 400036.
```

If the session is terminated, you can use the following code release the resource:

```python
data_analysis.close()
```

You can update the system message if it's not perfect for you model:

```python
data_analysis.update_pipeline_system_message.remote('''
You are a helpful data analysis assistant.
You don't need to write code, or anwser the question. The only thing you need to do 
is plan the data analysis pipeline.

You have some tools like the following:

1. visualization_agent, 这个 Agent 可以帮助你对数据进行可视化。
2. assistant_agent, 这个 Agent 可以帮你生成代码对数据进行分析，统计。
3. common_agent, 这个Agent 只会根据对话来帮助用户分析数据。他不会生成任何代码去分析数据。


Please check the user's question and decide which tool you need to use. And then reply the tool name only.
If there is no tool can help you, 
you should reply exactly `UPDATE CONTEXT`.
''')
```

You can get all agents in pipeline:

```python
data_analysis.get_agent_names()
# ['assistant_agent', 'visualization_agent', 'common_agent', 'privew_file_agent', 'python_interpreter']
```

Or get the agent system message by name:

```python
data_analysis.get_agent_system_message("assistant_agent")
```

Or update the agent system message by name:

```python
data_analysis.update_agent_system_message("assistant_agent","hello")
```

## Custom Agent

How to create a custom agent?

<p align="center">
  <img src="./images/CustomAgent.jpg" width="600" />
</p>

Try to extend the class `from byzerllm.apps.agent.conversable_agent import ConversableAgent`.

The agent provides two key communication funciton.

### send

The function signature:

```python
def send(
        self,
        message: Union[Dict, str],
        recipient: Union[ClientActorHandle,Agent,str], #"Agent"
        request_reply: Optional[bool] = None,
        silent: Optional[bool] = False,
    ) -> bool:
```

You can use this function to send a message to the other agent. You can set request_reply to False, then we just 
send the message to the self/agent's messagebox, but will not trigger the actual reply from the recipient.


### reply

You can create any function which accept the following parameters:

```python
def generate_xxxx_reply(
        self,
        raw_message: Optional[Union[Dict,str,ChatResponse]] = None,
        messages: Optional[List[Dict]] = None,
        sender: Optional[Union[ClientActorHandle,Agent,str]] = None,
        config: Optional[Any] = None,
        ) -> Tuple[bool, Union[str, Dict, None,ChatResponse]]:

        if messages is None:
            messages = self._messages[get_agent_name(sender)]  
```

You can register the `generate_xxxx_reply` to the agent in the `__init__` function:

```python
self._reply_func_list = []        
self.register_reply([Agent, ClientActorHandle,str], SparkSQLAgent.generate_xxxx_reply) 
self.register_reply([Agent, ClientActorHandle,str], ConversableAgent.check_termination_and_human_reply) 
```

Then when a message comes, the function `generate_xxxx_reply` will be invoke.
No matter the agent are local or remote, you can use the same `send` and `reply` function.

If you want to terminate the chat: you can finally return like this:

```python
return True, {"content":reply,"metadata":{"TERMINATE":True}}    
```

Or make sure the `content` the last line is "TERMINATE". Then the `ConversableAgent.check_termination_and_human_reply` will check it
every chat happens.

There are a lot of agent examples in [agent-extenions](https://github.com/allwefantasy/byzer-llm/tree/master/src/byzerllm/apps/agent/extensions).

## Remote Agent

You can use `from byzerllm.apps.agent import Agents` Agents to create a local or remote agent, 
for example, you can create a remote agent using the following code:

```python
privew_file_agent = Agents.create_remote_agent(PreviewFileAgent,"privew_file_agent",llm,retrieval,
                                   max_consecutive_auto_reply=3,
                                   code_agent = python_interpreter
                                   )
```

Notice that, remote agent can only talk to remote agent, and local agent can only talk to local agent.


Here are a code example show you how to create and invoke the remote agent:

```python
from byzerllm.apps.agent import Agents
from byzerllm.apps.agent.assistant_agent import AssistantAgent
from byzerllm.apps.agent.user_proxy_agent import UserProxyAgent
from byzerllm.apps.agent.extensions.python_codesandbox_agent import PythonSandboxAgent
from byzerllm.apps.agent.extensions.preview_file_agent import PreviewFileAgent

python_interpreter = Agents.create_remote_agent(PythonSandboxAgent,"python_interpreter",
                                                llm,retrieval,
                                                max_consecutive_auto_reply=1000,
                                                system_message="you are a code sandbox")

privew_file_agent = Agents.create_remote_agent(PreviewFileAgent,"privew_file_agent",llm,retrieval,
                                   max_consecutive_auto_reply=1000,
                                   code_agent = python_interpreter
                                   )

user = Agents.create_remote_agent(UserProxyAgent,"user",llm,retrieval,
                                human_input_mode="NEVER",
                                max_consecutive_auto_reply=0)

```

Then you can begin the talk:

```python
import ray

ray.get(privew_file_agent._prepare_chat.remote(python_interpreter, True))

file_path = "/home/byzerllm/projects/jupyter-workspace/test.csv"
file_ref = ray.put(open(file_path,"rb").read())

ray.get(
user.initiate_chat.remote(
privew_file_agent,
message={
    "content":"",
    "metadata":{
        "file_path": file_path,
        "file_ref": file_ref
    }
})
)
```

Since the agent is `remote`, so you should call the function in agent with `agent.function.remote` style.

Here is the console output:

```text
(PreviewFileAgent pid=2135038) user (to privew_file_agent):
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) --------------------------------------------------------------------------------
(PreviewFileAgent pid=2135038) python_interpreter (to privew_file_agent):
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) exitcode: 0 (execution succeeded)
(PreviewFileAgent pid=2135038) Code output:    ID   Deaths  Year                 Entity
(PreviewFileAgent pid=2135038) 0   1  1267360  1900  All natural disasters
(PreviewFileAgent pid=2135038) 1   2   200018  1901  All natural disasters
(PreviewFileAgent pid=2135038) 2   3    46037  1902  All natural disasters
(PreviewFileAgent pid=2135038) 3   4     6506  1903  All natural disasters
(PreviewFileAgent pid=2135038) 4   5    22758  1905  All natural disasters
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) 
(PreviewFileAgent pid=2135038) --------------------------------------------------------------------------------
(PythonSandboxAgent pid=2135037) privew_file_agent (to python_interpreter):
(PythonSandboxAgent pid=2135037) 
(PythonSandboxAgent pid=2135037) ```python
(PythonSandboxAgent pid=2135037) import pandas as pd
(PythonSandboxAgent pid=2135037) 
(PythonSandboxAgent pid=2135037) file_path = "/home/byzerllm/projects/jupyter-workspace/test.csv"
(PythonSandboxAgent pid=2135037) 
(PythonSandboxAgent pid=2135037) try:
(PythonSandboxAgent pid=2135037)     if file_path.endswith(".csv"):
(PythonSandboxAgent pid=2135037)         df = pd.read_csv(file_path)
(PythonSandboxAgent pid=2135037)     elif file_path.endswith(".xlsx") or file_path.endswith(".xls"):
(PythonSandboxAgent pid=2135037)         df = pd.read_excel(file_path)
(PythonSandboxAgent pid=2135037)     else:
(PythonSandboxAgent pid=2135037)         raise ValueError("Unsupported file format")
(PythonSandboxAgent pid=2135037)         
(PythonSandboxAgent pid=2135037)     loaded_successfully = True
(PythonSandboxAgent pid=2135037)     
(PythonSandboxAgent pid=2135037) except Exception as e:
(PythonSandboxAgent pid=2135037)     print(f"Failed to load file: {e}")
(PythonSandboxAgent pid=2135037)     loaded_successfully = False
(PythonSandboxAgent pid=2135037)     
(PythonSandboxAgent pid=2135037) if loaded_successfully:
(PythonSandboxAgent pid=2135037)     file_preview = df.head()
(PythonSandboxAgent pid=2135037) else:
(PythonSandboxAgent pid=2135037)     file_preview = "Error: Failed to load file"
(PythonSandboxAgent pid=2135037)     
(PythonSandboxAgent pid=2135037) print(file_preview)
(PythonSandboxAgent pid=2135037) ```
(PythonSandboxAgent pid=2135037) 
(PythonSandboxAgent pid=2135037) This code reads the file located at `file_path` using either the `pd.read_csv()` or `pd.read_excel()` function depending on the file type (determined by the file extension). If the file is loaded successfully, the first 5 rows of the DataFrame are stored in the `file_preview` variable and printed. If there is an error loading the file, an error message is stored in the `file_preview` variable and printed instead.
(PythonSandboxAgent pid=2135037) 
(PythonSandboxAgent pid=2135037) --------------------------------------------------------------------------------
(UserProxyAgent pid=2135039) privew_file_agent (to user):
(UserProxyAgent pid=2135039) 
(UserProxyAgent pid=2135039) ID,Deaths,Year,Entity
(UserProxyAgent pid=2135039) 1,1267360,1900,All natural disasters
(UserProxyAgent pid=2135039) 2,200018,1901,All natural disasters
(UserProxyAgent pid=2135039) 3,46037,1902,All natural disasters
(UserProxyAgent pid=2135039) 4,6506,1903,All natural disasters
(UserProxyAgent pid=2135039) 5,22758,1905,All natural disasters
(UserProxyAgent pid=2135039) 
(UserProxyAgent pid=2135039) 
(UserProxyAgent pid=2135039) --------------------------------------------------------------------------------
```

If you want to get agent's messagebox, try use the following code:

```python
ray.get(
   user.get_chat_messages.remote() 
)
```

The output:

```text
defaultdict(list,
            {'privew_file_agent': [{'content': '',
               'metadata': {'file_path': '/home/byzerllm/projects/jupyter-workspace/test.csv',
                'file_ref': ObjectRef(00ffffffffffffffffffffffffffffffffffffff1800000002e1f505)},
               'role': 'assistant'},
              {'content': 'ID,Deaths,Year,Entity\n1,1267360,1900,All natural disasters\n2,200018,1901,All natural disasters\n3,46037,1902,All natural disasters\n4,6506,1903,All natural disasters\n5,22758,1905,All natural disasters\n',
               'metadata': {'TERMINATE': True},
               'role': 'user'}]})
```







