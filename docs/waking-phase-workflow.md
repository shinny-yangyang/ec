# DreamCoder：Waking (唤醒) 阶段工作流程

本文档概述了 DreamCoder 系统中 "Waking"（也称为枚举或求解）阶段的程序执行流程。此阶段的主要目标是利用当前的程序库（Grammar）来枚举能够解决每个给定任务的程序，从而为每个任务形成一个解决方案的 "前沿"（Frontier）。

该工作流程涉及 Python 前端和 OCaml 后端之间的交互：

1.  **触发 (Python: `dreamcoder.py`)**
    *   DreamCoder 的主循环启动 Waking 阶段。

2.  **准备与调用 (Python: `enumeration.py`)**
    *   `enumeration.py` 模块负责协调此阶段。
    *   它获取当前的 `Grammar` 对象和需要解决的 `Task` 对象列表。
    *   它将 `Grammar`、`Task` 列表（包括任务类型、示例等）以及相关的配置参数（例如 `timeout`、`maximumFrontier`、`lowerBound`、`upperBound`、用于 CPU 核心数的 `nc`）序列化为 JSON 负载。此格式对应 OCaml `solver.ml` 的 `load_problems` 函数期望的输入格式。
    *   它使用 `subprocess.Popen` 将预编译的 OCaml 可执行文件 `solver`（由 `solvers/solver.ml` 编译而来）作为子进程启动。

3.  **数据传输 (Python -> OCaml)**
    *   序列化的 JSON 负载通过 OCaml `solver` 进程的标准输入 (stdin) 传递。

4.  **枚举执行 (OCaml: `solver.ml`)**
    *   `solver.ml` 进程启动。
    *   **输入解析 (`load_problems` 函数)：** 从 stdin 读取并解析 JSON 数据，将其转换为内部 OCaml 数据结构（语法、任务、参数）。
    *   **策略选择：** 根据输入 JSON 中是否提供了概率上下文无关文法 (PCFG) 来确定枚举策略。它在标准的基于语法的枚举 (`enumerate_programs`) 或可能更高效的基于 DP/PCFG 的枚举 (`dynamic_programming_enumeration`) 之间进行选择。
    *   **任务枚举 (`enumerate_for_tasks` 函数)：**
        *   协调任务列表 (`tf`) 的枚举过程。
        *   调用选定的后端枚举函数。
        *   后端根据 `Grammar` 和任务类型生成候选程序。
        *   根据任务的输入/输出示例评估生成的程序，以计算其对数似然 (`logLikelihood`)。
        *   同时计算程序在当前 `Grammar` 下的对数先验 (`logPrior`)。
        *   该过程利用多个 CPU 核心 (`nc` 参数) 进行并行计算。
        *   枚举受到 `timeout`、`lowerBound`、`upperBound` 和 `budgetIncrement` 等参数的限制。
        *   收集每个任务的成功程序，形成初始前沿。

5.  **结果返回 (OCaml -> Python)**
    *   **输出格式化 (`export_frontiers` 函数)：** 将为每个任务找到的解决方案（程序字符串、发现时间、似然、先验）以及枚举的总程序数 (`number_enumerated`) 打包成 JSON 对象。
    *   `solver.ml` 将此结果 JSON 字符串打印到其标准输出 (stdout)。

6.  **接收与处理 (Python: `enumeration.py`)**
    *   父 Python 进程 (`enumeration.py`) 读取 `solver` 进程的标准输出以捕获结果 JSON 字符串。
    *   它解析 JSON 字符串，将其转换回 Python 数据结构。主要是为每个任务创建一个 `Frontier` 对象，其中填充了表示解决程序的 `FrontierEntry` 对象及其元数据。

7.  **阶段完成**
    *   Waking 阶段结束。
    *   生成的 `Frontier` 对象列表被传递到 DreamCoder 循环的后续阶段，例如识别模型训练或压缩。

本质上，Waking 阶段将计算密集型的并行程序枚举和评估任务委托给 OCaml 后端，并由负责数据准备和结果处理的 Python 前端进行协调。 