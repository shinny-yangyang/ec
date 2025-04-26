创建新的领域
============

要创建新的问题领域以供解决，必须完成一些事情。按照以下步骤开始。

# 目录
1. [选择一个领域](#选择一个领域)
2. [创建领域脚本](#创建领域脚本)
    1. [添加 Python 导入](#添加-python-导入)
    2. [为脚本创建命令行](#为脚本创建命令行)
    3. [定义领域的基本类型](#定义领域的基本类型)
    4. [更新 OCaml 基本类型](#更新-ocaml-基本类型)
    5. [创建训练和测试任务](#创建训练和测试任务)
    5. [创建 `ecIterator`](#创建-eciterator)
    5. [最终脚本](#最终脚本)
3. [运行脚本](#运行脚本)

# 选择一个领域

选择一个合适的领域来训练 DreamCoder 算法以解决其中的任务。

在我们下面的例子中，领域将是正整数加法，任务涉及不同数字的加法。这是一个简单的玩具示例，用于演示你需要开始使用的代码库的不同组件。

# 创建领域脚本

首先，我们需要创建一个包含我们关心的基本类型以及我们希望程序解决的任务的脚本。

在 `bin/` 目录下创建一个 Python 文件作为我们的脚本。在这个例子中，我们将其命名为 `bin/incr.py`。

我们可以模仿 `bin/` 目录下的其他脚本来创建我们的脚本，例如 `bin/list.py` 或 `bin/text.py`。

### 添加 Python 导入

让我们首先在脚本中添加一些导入：
```python
import datetime
import os
import random

import binutil

from dreamcoder.ec import commandlineArguments, ecIterator
from dreamcoder.grammar import Grammar
from dreamcoder.program import Primitive
from dreamcoder.task import Task
from dreamcoder.type import arrow, tint
from dreamcoder.utilities import numberOfCPUs
```
`import binutil` 是针对此仓库目录结构的一个巧妙的解决方法。你可以暂时忽略它。

其他导入将为我们提供定义命令行、创建基本类型以及解决新领域内任务所需的其他基本组件。

### 为脚本创建命令行

要为我们的脚本创建一个 Python `argparse` 命令行，我们可以从 `ec.py` 模块导入 `commandlineArguments()` 函数，该函数包含了 DreamCoder 算法成功运行所需的大部分参数。

以下是一个设置了一些默认值的示例：
```python
args = commandlineArguments(
    enumerationTimeout=10, activation='tanh',
    iterations=10, recognitionTimeout=3600,
    a=3, maximumFrontier=10, topK=2, pseudoCounts=30.0,
    helmholtzRatio=0.5, structurePenalty=1.,
    CPUs=numberOfCPUs())
```

接下来，让我们开始定义我们的领域。

### 定义领域的基本类型

接下来，我们将为我们的玩具示例创建一个基本类型列表。

列表中的每个成员都必须是 `Primitive` 类的实例，其中每个基本类型都有一个唯一的名称，将其绑定到相应的 OCaml 代码（稍后讨论），一个从 `dreamcoder/type.py` 导入的类型，以及一个 lambda 函数：`Primitive(name, type, func)`。
```python
def _incr(x): return lambda x: x + 1
def _incr2(x): return lambda x: x + 2

primitives = [
    Primitive("incr", arrow(tint, tint), _incr),
    Primitive("incr2", arrow(tint, tint), _incr2),
]
```

然后根据基本类型创建一个语法：
```python
grammar = Grammar.uniform(primitives)
```

请注意，*目前无法在不修改 OCaml 代码的情况下创建新的基本类型*！请参阅下一节以继续。

### 更新 OCaml 基本类型

当前架构的一个限制是，基本类型也必须在 Python 前端和 OCaml 后端（在 `solvers/program.ml` 中）同时定义。

如果我们打开该文件，我们会发现没有 `incr2` 的基本类型，这将为我们的新领域引发运行时错误。因此，编辑 `solvers/program.ml` 文件以添加一个 "incr2" 基本类型：
```diff
 let primitive_increment = primitive "incr" (tint @> tint) (fun x -> 1+x);;
+let primitive_increment2 = primitive "incr2" (tint @> tint) (fun x -> 2+x);;
```

这也意味着我们需要重新构建 OCaml 二进制文件：
```
make clean
make
```
有关更多信息，请参阅仓库根目录下的 README（特别是"构建 OCaml 二进制文件"部分）。

### 创建训练和测试任务

现在我们已经定义了我们的基本类型和语法，我们可以在我们的领域中创建一些训练和测试任务。

首先，让我们定义一个辅助函数，它将某个数字 `N` 添加到一个伪随机数上：
```python
def addN(n):
    x = random.choice(range(500))
    return {"i": x, "o": x + n}
```
返回值是我们用来存储每个任务的输入和输出的字典格式。

每个任务将包含 3 个部分：
1. 一个名称
2. 从输入到输出类型的映射（例如 `arrow(tint, tint)`）
3. 输入-输出对列表

输入-输出对应该是一个元组列表 (input, output)，其中每个 input 本身就是一个元组。

让我们定义一个辅助函数来为我们创建任务：
```python
def get_tint_task(item):
    return Task(
        item["name"],
        arrow(tint, tint),
        [((ex["i"],), ex["o"]) for ex in item["examples"]],
    )
```

定义完辅助函数后，我们可以添加一些训练数据：
```python
# 训练数据
def add1(): return addN(1)
def add2(): return addN(2)
def add3(): return addN(3)
training_examples = [
    {"name": "add1", "examples": [add1() for _ in range(5000)]},
    {"name": "add2", "examples": [add2() for _ in range(5000)]},
    {"name": "add3", "examples": [add3() for _ in range(5000)]},
]
training = [get_tint_task(item) for item in training_examples]
```

接下来，让我们添加少量的测试数据：
```python
# 测试数据
def add4(): return addN(4)
testing_examples = [
    {"name": "add4", "examples": [add4() for _ in range(500)]},
]
testing = [get_tint_task(item) for item in testing_examples]
```

### 创建 `ecIterator`

最后，为了让 DreamCoder 算法能够处理我们的任务，我们需要在脚本中创建一个 `ecIterator`，如下所示：
```python
generator = ecIterator(grammar,
                       training,
                       testingTasks=testing,
                       **args)
for i, _ in enumerate(generator):
    print('ecIterator count {}'.format(i))
```

这将迭代处理我们任务的唤醒和睡眠周期。

### 最终脚本

我们领域脚本的最终结果可能看起来像这样：
```python
import datetime
import os
import random

import binutil  # 需要从 dreamcoder 模块导入

from dreamcoder.ec import commandlineArguments, ecIterator
from dreamcoder.grammar import Grammar
from dreamcoder.program import Primitive
from dreamcoder.task import Task
from dreamcoder.type import arrow, tint
from dreamcoder.utilities import numberOfCPUs

# 基本类型
def _incr(x): return lambda x: x + 1
def _incr2(x): return lambda x: x + 2


def addN(n):
    x = random.choice(range(500))
    return {"i": x, "o": x + n}


def get_tint_task(item):
    return Task(
        item["name"],
        arrow(tint, tint),
        [((ex["i"],), ex["o"]) for ex in item["examples"]],
    )


if __name__ == "__main__":

    # 参数或多或少是从 list.py 复制过来的

    args = commandlineArguments(
        enumerationTimeout=10, activation='tanh',
        iterations=10, recognitionTimeout=3600,
        a=3, maximumFrontier=10, topK=2, pseudoCounts=30.0,
        helmholtzRatio=0.5, structurePenalty=1.,
        CPUs=numberOfCPUs())

    timestamp = datetime.datetime.now().isoformat()
    outdir = 'experimentOutputs/demo/'
    os.makedirs(outdir, exist_ok=True)
    outprefix = outdir + timestamp
    args.update({"outputPrefix": outprefix})

    # 创建基本类型列表

    primitives = [
        Primitive("incr", arrow(tint, tint), _incr),
        Primitive("incr2", arrow(tint, tint), _incr2),
    ]

    # 创建语法

    grammar = Grammar.uniform(primitives)

    def add1(): return addN(1)
    def add2(): return addN(2)
    def add3(): return addN(3)

    # 训练数据

    training_examples = [
        {"name": "add1", "examples": [add1() for _ in range(5000)]},
        {"name": "add2", "examples": [add2() for _ in range(5000)]},
        {"name": "add3", "examples": [add3() for _ in range(5000)]},
    ]
    training = [get_tint_task(item) for item in training_examples]

    # 测试数据

    def add4(): return addN(4)

    testing_examples = [
        {"name": "add4", "examples": [add4() for _ in range(500)]},
    ]
    testing = [get_tint_task(item) for item in testing_examples]

    # EC 迭代

    generator = ecIterator(grammar,
                           training,
                           testingTasks=testing,
                           **args)
    for i, _ in enumerate(generator):
        print('ecIterator count {}'.format(i))
```

提醒：在进行下一步之前，不要忘记重新构建 OCaml 二进制文件！

# 运行脚本

要运行脚本，请执行以下命令：
```
python bin/incr.py -t 2 --testingTimeout 2
```

因此，在一个 Singularity 容器中进行 2 次迭代 (`-i 2`)：
```
singularity exec container.img python bin/incr.py -t 2 --testingTimeout 2 -i 2
```

我们的脚本是一个简单的示例，因此我们不应期望在迭代过程中看到太多改进。程序应该在第一次迭代期间解决。

训练任务很简单，因此我们应该期望看到类似控制台输出的内容，显示算法为每个任务找到了解决方案：
```
Generative model enumeration results:
HIT add1 w/ (lambda (incr $0)) ; log prior = -2.197225 ; log likelihood = 0.000000
HIT add2 w/ (lambda (incr2 $0)) ; log prior = -2.197225 ; log likelihood = 0.000000
HIT add3 w/ (lambda (incr (incr2 $0))) ; log prior = -3.295837 ; log likelihood = 0.000000
```

控制台输出应该显示算法在某个时刻也解决了测试任务：
```
HIT add4 w/ (lambda (incr2 (incr2 $0))) ; log prior = -3.295837 ; log likelihood = 0.000000
```

对于更复杂的示例，其中任务并非都在第一次迭代中立即解决，随着算法的改进，损失应在每次迭代中下降。

有关运行脚本的更多信息，请参阅 README.md 中的"从命令行运行任务"。在测试更复杂的领域时，还请阅读有关绘制结果图表的信息。 