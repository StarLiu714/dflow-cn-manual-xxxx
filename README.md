# DFLOW

[Dflow](https://deepmodeling.com/dflow/dflow.html) 是一个用于构建科学计算工作流程（例如并发学习工作流程）的Python框架。 [Argo Workflows](https://argoproj.github.io/) as the workflow engine.

对于dflow的用户（例如机器学习应用开发者），dflow提供了用户友好的函数式编程界面，用于构建他们自己的工作流程。用户不需要关心流程控制、任务调度、可观测性和容灾性。用户可以通过API以及前端界面来跟踪工作流程的状态并处理异常。因此，用户可以专注于实施操作（OPs）和编排工作流程。

对于dflow的开发者来说，dflow封装了argo SDK，隐藏了计算和存储资源的细节，并提供了扩展能力。虽然argo是一个云原生的工作流引擎，但dflow使用容器来解耦计算逻辑和调度逻辑，并使用Kubernetes来使工作流程可观测、可复制和稳健。Dflow被设计为基于分布式、异构基础设施。科学计算中最常见的计算资源可能是HPC集群。用户可以使用执行器来使用[DPDispatcher](https://github.com/deepmodeling/dpdispatcher)插件管理HPC作业，也可以使用虚拟节点技术在Kubernetes框架下统一管理HPC资源（[wlm-operator](https://github.com/dptech-corp/wlm-operator)）。

在dflow中，操作模板（缩写为OP）可以在工作流程之间重复使用并在用户之间共享。Dflow提供了一个cookie cutter配方[dflow-op-cutter](https://github.com/deepmodeling/dflow-op-cutter)，用于模板化一个新的OP包。从以下命令开始开发一个OP包：

```python
pip install cookiecutter
cookiecutter https://github.com/deepmodeling/dflow-op-cutter.git
```

Dflow提供了一个调试模式，用于以纯Python实现的dflow后端运行工作流程，与Argo/Kubernetes独立。调试模式使用本地环境来执行OPs，而不是容器。它实现了默认模式的大多数API，以提供相同的用户体验。调试模式提供了在没有容器的情况下进行调试或测试的便利性。对于部署Docker和Kubernetes有问题并且难以从外部访问的集群，调试模式也可以用于生产，尽管稳定性和可观测性较低。

<!-- vscode-markdown-toc -->
* 1. [概览](#1-概览)
	* 1.1. [架构](#11-架构)
	* 1.2. [基础知识](#12-基础知识)
		* 1.2.1. [参数和工件](#121-参数和工件)
		* 1.2.2. [OP模板](#122-op模板)
		* 1.2.3. [步骤](#123-步骤)
		* 1.2.4. [工作流程](#124-工作流程)
* 2. [快速入门](#2-快速入门)
	* 2.1. [设置Argo服务器](#21-设置Argo服务器)
	* 2.2. [安装dflow](#22-安装dflow)
	* 2.3. [运行示例](#23-运行示例)
* 3. [用户指南](#3-用户指南)
	* 3.1. [通用层](#31-通用层)
		* 3.1.1. [工作流程管理](#311-工作流程管理)
		* 3.1.2. [上传/下载工件](#312-上传下载工件)
		* 3.1.3. [步骤](#313-步骤)
		* 3.1.4. [DAG](#314-DAG)
		* 3.1.5. [条件步骤、参数和工件](#315-条件步骤参数和工件)
		* 3.1.6. [使用循环生成并行步骤](#316-使用循环生成并行步骤)
		* 3.1.7. [超时](#317-超时)
		* 3.1.8. [失败时继续](#318-失败时继续)
		* 3.1.9. [在并行步骤成功数量/比例时继续](#319-在并行步骤成功数量比例时继续)
		* 3.1.10. [可选输入工件](#3110-可选输入工件)
		* 3.1.11. [输出参数的默认值](#3111-输出参数的默认值)
		* 3.1.12. [步骤的键](#3112-步骤的键)
		* 3.1.13. [重新提交工作流程](#3113-重新提交工作流程)
		* 3.1.14. [执行器](#3114-执行器)
		* 3.1.15. [通过调度程序插件提交HPC/Bohrium作业](#3115-通过调度程序插件提交HPCBohrium作业)
		* 3.1.16. [通过虚拟节点提交Slurm作业](#3116-通过虚拟节点提交Slurm作业)
		* 3.1.17. [在Kubernetes中使用资源](#3117-在Kubernetes中使用资源)
		* 3.1.18. [重要注意事项：变量名称](#3118-重要注意事项变量名称)
		* 3.1.19. [调试模式：与Kubernetes无关的dflow](#3119-调试模式与Kubernetes无关的dflow)
		* 3.1.20. [工件存储插件](#3120-工件存储插件)
	* 3.2. [接口层](#32-接口层)
		* 3.2.1. [切片](#321-切片)
		* 3.2.2. [重试和错误处理](#322-重试和错误处理)
		* 3.2.3. [进度](#323-进度)
		* 3.2.4. [上传Python开发包](#324-上传Python开发包)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## 1. 概述
###  1.1. 架构
dflow由一个**通用层**和一个**接口层**组成。接口层接受用户提供的各种操作模板，通常以Python类或函数的形式，然后将它们转换为通用层可以处理的基本操作模板。

<p align="center">
<img src="./docs/imgs/dflow_architecture.png" alt="dflow_architecture" width="400"/>
</p>

###  1.2. 基础知识

####  1.2.1. 参数和工件
参数和工件是工作流程存储并在工作流程中传递的数据。参数以文本形式保存，可以在用户界面中显示，而工件以文件形式保存。参数以其值的形式传递给操作（OP），而工件以路径形式传递。

####  1.2.2. 操作模板
操作模板（缩写为OP）是工作流程的基本构建块。它定义了在给定输入和期望输出的情况下要执行的特定操作。输入和输出都可以是参数和/或工件。最常见的操作模板是容器操作模板。支持两种类型的容器操作模板：`ShellOPTemplate`、`PythonScriptOPTemplate`。`ShellOPTemplate`通过shell脚本和脚本运行的容器镜像定义操作。`PythonScriptOPTemplate`通过Python脚本和容器镜像定义操作。

作为更适用于Python的操作模板类别，`PythonOPTemplate`以Python类或Python函数的形式定义操作（相应地称为类操作或函数操作）。由于Python是弱类型语言，我们对Python操作进行了严格的类型检查，以减轻歧义和意外行为。

对于类操作，操作的输入和输出结构在静态方法`get_input_sign`和`get_output_sign`中声明。每个方法都返回一个从参数/工件名称到其类型的字典映射。操作的执行在`execute`方法中定义。传递给操作的参数值的类型应与签名中声明的类型一致。类型检查在`execute`方法之前和之后实施。对于输入/输出工件，其签名应类似于`Artifact(type)`，其中`type`可以是`Path`、`List[Path]`、`Dict[str, Path]`或`dflow.python.NestedDict[Path]`。对于输入工件，`execute`方法将根据其签名接收路径、路径列表或路径字典。操作开发者可以直接处理路径中的文件或目录。对于输出工件，`execute`方法也应根据其签名返回路径、路径列表或路径字典。

```python
from dflow.python import OP, OPIO, OPIOSign, Artifact
from pathlib import Path
import shutil


class SimpleExample(OP):
    def __init__(self):
        pass

    @classmethod
    def get_input_sign(cls):
        return OPIOSign(
            {
                "msg": str,
                "foo": Artifact(Path),
            }
        )

    @classmethod
    def get_output_sign(cls):
        return OPIOSign(
            {
                "msg": str,
                "bar": Artifact(Path),
            }
        )

    @OP.exec_sign_check
    def execute(
        self,
        op_in: OPIO,
    ) -> OPIO:
        shutil.copy(op_in["foo"], "bar.txt")
        out_msg = op_in["msg"]
        op_out = OPIO(
            {
                "msg": out_msg,
                "bar": Path("bar.txt"),
            }
        )
        return op_out
```

上述示例定义了一个名为`SimpleExample`的操作模板（OP）。该操作的功能是将输入的工件`foo`复制到输出工件`bar`，并将输入参数`msg`复制到输出参数`msg`。

对于函数操作（function OP），输入和输出的结构更紧凑地在类型注释中声明，执行逻辑则在函数体中定义。类型检查也在函数前后进行。我们建议使用`python>=3.9`版本以使用这种语法糖。有关Python注释的更多信息，请参阅以下链接:
[Python official howtos](https://docs.python.org/3/howto/annotations.html).

```python
from dflow.python import OP, Artifact
from pathlib import Path
import shutil

@OP.function
def SimpleExample(
		msg: str,
		foo: Artifact(Path),
) -> {"msg": str, "bar": Artifact(Path)}:
    shutil.copy(foo, "bar.txt")
    out_msg = msg
    return {"msg": out_msg, "bar": Path("bar.txt")}
```

要从上述的类或函数定义一个操作模板（OP模板），我们需要指定容器镜像以及`PythonOPTemplate`的其他可选参数。在此镜像中，无需安装`pydflow`，因为默认情况下将本地的`pydflow`包上传到容器中。

```python
from dflow.python import PythonOPTemplate

simple_example_templ = PythonOPTemplate(SimpleExample, image="python:3.8")
```

一个例子:
- [Python OP example](examples/test_python.py)

#### 1.2.3. 步骤
`Step`是制定数据流规则的核心块。一个步骤是实例化操作模板的结果，其中必须在此处指定操作模板中声明的所有输入参数的值和所有输入工件的源。步骤的输入参数/工件可以在提交时是静态的，也可以是来自另一个步骤的输出的动态值。

```python
from dflow import Step

simple_example_step = Step(
    name="step0",
    template=simple_example_templ,
    parameters={"msg": "HelloWorld!"},
    artifacts={"inp_art": foo},
)
``` 

请注意，这里的`foo`是一个工件，可以从本地上传，也可以是另一个步骤的输出。

#### 1.2.4. 工作流程
`Workflow`将各个块连接在一起，以构建工作流程。通过按顺序添加步骤来创建简单的串行工作流程。将一系列步骤添加到工作流程中意味着这些步骤并行运行。

```python
from dflow import Workflow

wf = Workflow(name="hello-world")
wf.add(simple_example_step)
```

Submit a workflow by

```python
wf.submit()
```

一个例子:
- [Workflow example](examples/test_steps.py)


## 2. 快速入门

### 2.1. 设置 Argo 服务器

如果您已经有一个Argo服务器，可以跳过此步骤。否则，您可以按照[安装指南](tutorials/readme.md)进行设置。

### 2.2. 安装 dflow
确保您的Python版本不低于3.6，并安装dflow

```
pip install pydflow
```


### 2.3. 运行示例
有几个[notebook教程](tutorials/readme.md)可以帮助您开始使用dflow。此外，您可以从终端提交一个简单的工作流程

```
python examples/test_python.py
```

然后，您可以通过[argo的用户界面](https://127.0.0.1:2746)来检查已提交的工作流程。


## 3. 用户指南 ([dflow-doc](https://deepmodeling.com/dflow/dflow.html))
### 3.1. 通用层

#### 3.1.1. 工作流管理
在通过`wf.submit()`提交工作流程或通过`wf = Workflow(id="xxx")`获取历史工作流程后，可以使用以下API来实时跟踪其状态：

- `wf.id`: argo中的工作流ID
- `wf.query_status()`: 查询工作流程状态，返回`"Pending"`、`"Running"`、`"Succeeded"`等。
- `wf.query_step(name=None)`: 按名称查询步骤（支持正则表达式），返回一个argo步骤对象列表
    - `step.phase`: 步骤的阶段，`"Pending"`、`"Running"`、`Succeeded`等。
    - `step.outputs.parameters`: 输出参数的字典
    - `step.outputs.artifacts`: 输出工件的字典

#### 3.1.2. 上传/下载工件
Dflow提供了上传文件到工件存储库和从工件存储库下载文件的工具（默认工件存储库是在快速入门中设置的Minio）。用户可以上传一个文件/目录、一组文件/目录或一个文件/目录字典，并获取一个工件对象，该对象可以用作步骤的参数

```python
artifact = upload_artifact([path1, path2])
step = Step(
    ...
    artifacts={"foo": artifact}
)
```

用户还可以下载从工作流程中查询的步骤的输出工件（默认为当前目录）

```python
step = wf.query_step(name="hello")
download_artifact(step.outputs.artifacts["bar"])
```

Modify `dflow.s3_config` to configure artifact repository settings globally.

通过修改dflow.s3_config来全局配置工件存储库设置。

注意：dflow在上传过程中保留文件/目录相对于当前目录的相对路径。如果上传了位于当前目录之外的文件/目录，其绝对路径将用作工件中的相对路径。如果要在工件中使用不同的目录结构与本地目录结构，请创建软链接然后上传。

3.1.3. 步骤

`Steps` 是另一种操作模板，它是由其组成的步骤而不是容器定义的。它可以看作是一个子工作流或由一些较小的操作模板组成的超级操作模板。`steps` 包括一个包含 `step` 数组的数组，例如 `[[s00, s01], [s10, s11, s12]]`，其中内部数组表示并行步骤，而外部数组表示顺序步骤。用户可以声明 `steps` 的输入/输出参数/工件，如下所示：


```python
steps.inputs.parameters["msg"] = InputParameter()
steps.inputs.artifacts["foo"] = InputArtifact()
steps.outputs.parameters["msg"] = OutputParameter()
steps.outputs.parameters["bar"] = OutputArtifact()
```


如同为`workflow`添加`step`一样，将一个 `step` 添加到 `steps` 中


```python
steps.add(step)
```

`steps` 可以像脚本操作模板一样用作实例化更大 `step` 的模板。因此，可以构建具有嵌套结构的复杂工作流程。还允许在 `steps` 内部递归使用 `steps` 作为其自身内部的构建块的模板，以实现动态循环。

`steps` 的输出参数可以设置为来自其中一个步骤的参数，如下所示：

```python
steps.outputs.parameters["msg"].value_from_parameter = step.outputs.parameters["msg"]
```

在这里，`step` 必须包含在 `steps` 中。要为 `steps` 分配输出工件，使用以下方式：

```python
steps.outputs.artifacts["foo"]._from = step.outputs.parameters["foo"]
```

- [Recursive example](examples/test_recurse.py)

#### 3.1.4. DAG
`DAG` 是另一种由其组成的任务及其依赖关系定义的操作模板。`DAG` 的使用方式与 `Steps` 类似。要将一个 `task` 添加到 `dag` 中，请使用以下方式：

```python
dag.add(task)
```

`task` 的使用方式也与 `step` 类似。Dflow将自动检测`dag`中任务之间的依赖关系（从输入/输出关系中）。还可以通过以下方式声明额外的依赖关系：

```python
task_3 = Task(..., dependencies=[task_1, task_2])
```

- [DAG example](examples/test_dag.py)

#### 3.1.5. 有条件的步骤、参数和工件
通过`Step(..., when=expr)`将步骤设置为有条件的，其中`expr`是以字符串格式表示的布尔表达式，例如`"%s < %s" % (par1, par2)`。如果表达式被评估为true，则执行步骤，否则跳过。`when`参数通常用作递归步骤的中断条件。`steps`（类似于`dag`）的输出参数也可以被分配为有条件的，如下所示：

```python
steps.outputs.parameters["msg"].value_from_expression = if_expression(
    _if=par1 < par2,
    _then=par3,
    _else=par4
)
```

同样，可以通过以下方式将 `steps` 的输出工件分配为有条件的：


```python
steps.outputs.artifacts["foo"].from_expression = if_expression(
    _if=par1 < par2,
    _then=art1,
    _else=art2
)
```

- [Conditional outputs example](examples/test_conditional_outputs.py)

#### 3.1.6. 使用循环生成并行步骤
在科学计算中，通常需要生成一系列共享通用操作模板但仅在输入参数上不同的并行步骤列表。`with_param` 和 `with_sequence` 是 `Step` 的两个参数，用于自动生成并行步骤列表。这些步骤共享通用操作模板，仅在输入参数上不同。

使用 `with_param` 选项的步骤在列表上生成并行步骤（可以是常量列表或引用另一个参数的列表，例如，另一个步骤的输出参数或 `steps` 或 `DAG` 上的输入参数），并行度等于列表的长度。每个并行步骤通过 `"{{item}}"` 从列表中选择一个项目，例如

```python
step = Step(
    ...
    parameters={"msg": "{{item}}"},
    with_param=steps.inputs.parameters["msg_list"]
)
```

使用 `with_sequence` 选项的步骤在数值序列上生成并行步骤。通常，`with_sequence` 与返回Argo序列的 `argo_sequence` 协同使用。对于 `argo_sequence`，指定序列的起始位置的数字由 `start` 指定（默认值：0）。可以通过 `count` 指定序列中的元素数量，也可以通过 `end` 指定序列的结束位置的数字。可以通过 `format` 指定 printf 格式字符串，以格式化序列中的值。每个参数都可以通过参数传递，`argo_len` 返回列表的长度可能会有用。每个并行步骤通过 `"{{item}}"` 从序列中选择一个元素，例如

```python
step = Step(
    ...
    parameters={"i": "{{item}}"},
    with_sequence=argo_sequence(argo_len(steps.inputs.parameters["msg_list"]))
)
```

#### 3.1.7. 超时
通过 `Step(..., timeout=t)` 设置步骤的超时时间。单位是秒。

- [Timeout example](examples/test_error_handling.py)

#### 3.1.8. 失败时继续
通过 `Step(..., continue_on_failed=True)` 设置工作流在步骤失败时继续执行。

- [Continue on failed example](examples/test_error_handling.py)

#### 3.1.9. 并行步骤成功数量/比例时继续
对于由 `with_param` 或 `with_sequence` 生成的一组并行步骤，通过 `Step(..., continue_on_num_success=n)` 或 `Step(..., continue_on_success_ratio=r)` 设置工作流在特定数量/比例的并行步骤成功时继续执行。

- [Continue on success ratio example](examples/test_success_ratio.py)

#### 3.1.10. 可选输入工件
通过 `op_template.inputs.artifacts["foo"].optional = True` 设置输入工件为可选的。

#### 3.1.11. 输出参数的默认值
通过 `op_template.outputs.parameters["msg"].default = default_value` 为输出参数设置默认值。当 `value_from_expression` 中的表达式失败或步骤被跳过时，将使用默认值。

#### 3.1.12. 步骤的键
可以通过 `Step(..., key="some-key")` 为步骤分配一个键，以方便定位步骤。键可以被视为一个可能包含其他参数引用的输入参数。例如，步骤的键可以随着动态循环的迭代而更改。一旦为步骤分配了键，就可以通过 `wf.query_step(key="some-key")` 查询步骤。如果在工作流程内键是唯一的，`query_step` 方法将返回一个仅包含一个元素的列表。

- [Key of step example](examples/test_reuse.py)

#### 3.1.13. 重新提交工作流
工作流通常具有一些计算成本较高的步骤。可以重用先前运行步骤的输出以提交新的工作流程。例如，可以通过 `wf.submit(reuse_step=[step0, step1])` 提交一个包含重用步骤的工作流程。这里，`step0` 和 `step1` 是由 `query_step` 方法返回的先前运行的步骤。在新的工作流程运行步骤之前，它将检测是否存在一个与即将运行的步骤的键匹配的重用步骤。如果匹配成功，工作流程将跳过该步骤并将其输出设置为重用步骤的输出。在重用之前修改步骤的输出，可以使用 `step0.modify_output_parameter(par_name, value)` 来处理参数，使用 `step0.modify_output_artifact(art_name, artifact)` 来处理工件。

- [Reuse step example](examples/test_reuse.py)

#### 3.1.14. 执行器
对于“脚本步骤”（其模板是脚本操作模板的步骤），默认情况下，Shell脚本或Python脚本直接在容器中运行。或者，可以修改执行器来运行脚本。Dflow为“脚本步骤”提供了一个扩展点 `Step(..., executor=my_executor)`。在这里，`my_executor` 应该是从抽象类 `Executor` 派生的类的实例。 `Executor` 的实现类应该实现一个名为 `render` 的方法，将原始模板转换为新模板。


```python
class Executor(ABC):
    @abc.abstractmethod
    def render(self, template):
        pass
```

#### 3.1.15. 通过调度插件提交 HPC/Bohrium 作业

上下文（context）类似于执行器（executor），但分配给工作流 `Workflow(context=...)` 并影响每个步骤。

[DPDispatcher](https://github.com/deepmodeling/dpdispatcher) 是一个用于生成 HPC 调度系统（Slurm/PBS/LSF）或 [Bohrium](https://bohrium.dp.tech) 作业输入脚本并提交这些脚本并轮询直到完成的 Python 包。Dflow 提供了一个简单的界面，以调用调度程序作为执行器来完成脚本步骤。例如，

```python
from dflow.plugins.dispatcher import DispatcherExecutor
Step(
    ...,
    executor=DispatcherExecutor(host="1.2.3.4",
        username="myuser",
        queue_name="V100")
)
```

对于 SSH 认证，可以在本地指定私钥文件的路径，也可以将授权的私钥上传到每个节点（或等效地将每个节点添加到授权主机列表中）。要为调度程序配置额外的 [machine, resources or task parameters for dispatcher](https://docs.deepmodeling.com/projects/dpdispatcher/en/latest/)，请使用 `DispatcherExecutor(..., machine_dict=m, resources_dict=r, task_dict=t)`。

- [Dispatcher executor example](examples/test_dispatcher.py)

#### 3.1.16. 通过虚拟节点提交 Slurm 作业

请按照 [wlm-operator](https://github.com/dptech-corp/wlm-operator) 项目中的安装步骤，将 Slurm 分区添加为 Kubernetes 的虚拟节点（使用该项目中的清单 [configurator.yaml](manifests/configurator.yaml)、[operator-rbac.yaml](manifests/operator-rbac.yaml)、[operator.yaml](manifests/operator.yaml) 来修改一些 RBAC 配置）

```
$ kubectl get nodes
NAME                            STATUS   ROLES                  AGE    VERSION
minikube                        Ready    control-plane,master   49d    v1.22.3
slurm-minikube-cpu              Ready    agent                  131m   v1.13.1-vk-N/A
slurm-minikube-dplc-ai-v100x8   Ready    agent                  131m   v1.13.1-vk-N/A
slurm-minikube-v100             Ready    agent                  131m   v1.13.1-vk-N/A
```
然后您可以分配要在虚拟节点上执行的步骤（即向相应分区提交 Slurm 作业来完成该步骤）

```python
step = Step(
    ...
    executor=SlurmJobTemplate(
        header="#!/bin/sh\n#SBATCH --nodes=1",
        node_selector={"kubernetes.io/hostname": "slurm-minikube-v100"}
    )
)
```

#### 3.1.17. 在 Kubernetes 中使用资源

一个步骤也可以由 Kubernetes 资源（例如 Job 或自定义资源）完成。首先，一个清单被应用到 Kubernetes。然后监视资源的状态，直到满足成功条件或失败条件。

```python
class Resource(ABC):
    action = None
    success_condition = None
    failure_condition = None
    @abc.abstractmethod
        pass
```

- [Wlm example](examples/test_wlm.py)

#### 3.1.18. 重要提示：变量名称

Dflow对变量名称有以下限制:

| Variable name | Static/Dynamic | Restrictions | Example |
| :------------ | -------------- | ------------ | ------- |
| Workflow/OP template name | Static | Lowercase RFC 1123 subdomain (must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character | my-name |
| Step/Task name | Static | Must consist of alpha-numeric characters or '-', and must start with an alpha-numeric character | My-name1-2, 123-NAME |
| Parameter/Artifact name | Static | Must consist of alpha-numeric characters, '_' or '-' | my_param_1, MY-PARAM-1 |
| Key name | Dynamic | Lowercase RFC 1123 subdomain (must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character | my-name |

#### 3.1.19. 调试模式：dflow 独立于 Kubernetes

通过设置以下环境变量启用调试模式

```python
from dflow import config
config["mode"] = "debug"
```

在本地运行工作流之前，请确保工作流中所有操作符的依赖项在本地环境中配置正确，除非使用分派执行器将作业提交到某些远程环境。默认情况下，调试模式使用当前目录作为工作目录。每个工作流将在其中创建一个新目录，其结构如下
```
python-lsev6
├── status
└── step-penf5
    ├── inputs
    │   ├── artifacts
    │   │   ├── dflow_python_packages
    │   │   ├── foo
    │   │   └── idir
    │   └── parameters
    │       ├── msg
    │       └── num
    ├── log.txt
    ├── outputs
    │   ├── artifacts
    │   │   ├── bar
    │   │   └── odir
    │   └── parameters
    │       └── msg
    ├── phase
    ├── script
    ├── type
    └── workdir
        ├── ...
```
顶层包含工作流的状态和所有步骤。 每个步骤的目录名称将是其键（如果提供），否则从其名称生成。 步骤目录包含步骤的输入/输出参数/工件、类型和阶段。 对于"Pod"类型的步骤，其目录还包括该步骤的脚本、日志文件和工作目录。

- [Debug mode examples](examples/debug)

3.1.20. 存储插件

dflow 中的默认工件存储是 Kubernetes 集群中的 Minio 部署。虽然支持其他工件存储（例如，Aliyun OSS、Azure Blob 存储（ABS）、Google 云存储（GCS）），但 dflow 提供了一个扩展点，以在工件管理中使用自定义存储。存储客户端是一个实现了 5 个抽象方法的类，这些方法包括 `upload`、`download`、`list`、`copy` 和 `get_md5`（可选），它们分别提供了上传文件、下载文件、列出带有特定前缀的文件、在服务器端复制文件以及获取文件的 MD5 摘要的功能。通过配置 s3_config["storage_client"] 使用自定义存储客户端对象。

```python
class StorageClient(ABC):
    @abc.abstractmethod
    def upload(self, key: str, path: str) -> None:
        pass
    @abc.abstractmethod
    def download(self, key: str, path: str) -> None:
        pass
    @abc.abstractmethod
    def list(self, prefix: str, recursive: bool = False) -> List[str]:
        pass
    @abc.abstractmethod
    def copy(self, src: str, dst: str) -> None:
        pass
    @abc.abstractmethod
    def get_md5(self, key: str) -> str:
        pass
```

### 3.2. 接口层

#### 3.2.1. 切片
在与 [并行步骤](#Produceparallelstepsusingloop), 协同工作时，`Slices` 有助于用户将输入参数/工件（必须为列表）切片，以供并行步骤，并将它们的输出参数/工件堆叠到相同模式的列表中。Python OP 只需要处理一个切片。例如，

```python
step = Step(name="parallel-tasks",
    template=PythonOPTemplate(
        ...,
        slices=Slices("{{item}}",
            input_parameter=["msg"],
            input_artifact=["data"],
            output_artifact=["log"])
    ),
    parameters = {
        "msg": msg_list
    },
    artifacts={
        "data": data_list
    },
    with_param=argo_range(5)
)
```

在这个示例中，将`msg_list`中的每个项目作为输入参数`msg`传递给并行步骤，将`data_list`中的每个部分作为输入工件`data`传递给并行步骤。最后，收集所有并行步骤的输出工件`log`到一个工件中，命名为`step.outputs.artifacts["log"]`。这个示例类似于以下伪代码：
```python
log = [None] * 5
for item in range(5):
    log[item] = my_op(msg=msg_list[item], data=data_list[item])
```
其中，`with_param` 和 `slices` 对应于伪代码中的 `for` 循环和循环内的语句。

- [Slices example](examples/test_slices.py)

值得注意的是，默认情况下，该功能会将完整的输入工件传递给每个并行步骤，而这些步骤可能只使用这些工件的某些片段。相比之下，切片的子路径模式只会将输入工件的单个切片传递给每个并行步骤。要使用切片的子路径模式，

```python
step = Step(name="parallel-tasks",
    template=PythonOPTemplate(
        ...,
        slices=Slices(sub_path=True,
            input_parameter=["msg"],
            input_artifact=["data"],
            output_artifact=["log"])
    ),
    parameters = {
        "msg": msg_list
    },
    artifacts={
        "data": data_list
    })
```

在这里，`PythonOPTemplate`的切片模式(`{{item}}`)和`Step`的`with_param`参数不需要设置，因为在这种模式下它们是固定的。要切片的每个输入参数和工件必须具有相同的长度，且并行性等于这个长度。另一个需要注意的点是，为了使用工件的子路径，这些工件在生成时必须以无压缩的方式保存。例如，在Python OP的输出标志中声明`Artifact(..., archive=None)`，或在上传工件时指定`upload_artifact(..., archive=None)`。此外，可以使用`dflow.config["archive_mode"] = None`将默认的存档模式全局设置为无压缩。

- [Subpath mode of slices example](examples/test_subpath_slices.py)

####  3.2.2. 重试和错误处理
Dflow捕获从`OP`中抛出的`TransientError`和`FatalError`。用户可以通过`PythonOPTemplate(..., retry_on_transient_error=n)`来设置在`TransientError`上的最大重试次数。默认情况下，超时错误被视为致命错误。要将超时错误视为瞬态错误，请设置`PythonOPTemplate(..., timeout_as_transient_error=True)`。当引发致命错误或瞬态错误的重试次数达到最大重试次数时，步骤被视为失败。

- [Retry example](examples/test_error_handling.py)

####  3.2.3. 进度
`OP`可以在运行时更新进度，以便用户可以跟踪其实时进度。

```python
class Progress(OP):
    progress_total = 100
    ...
    def execute(op_in):
        for i in range(10):
            self.progress_current = 10 * (i + 1)
            ...
```

- [Progress example](examples/test_progress.py)

####  3.2.4. 上传Python开发包
为了避免在开发过程中频繁制作映像，dflow提供了一个接口，可以将本地包上传到容器中并将它们添加到`$PYTHONPATH`中，例如`PythonOPTemplate(..., python_packages=["/opt/anaconda3/lib/python3.9/site-packages/numpy"])`。还可以全局指定要上传的包，这将影响所有的`OP`:

```python
from dflow.python import upload_packages
upload_packages.append("/opt/anaconda3/lib/python3.9/site-packages/numpy")
```
