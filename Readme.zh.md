# 目录
1. [概述](#概述)
2. [入门指南](#入门指南)
    1. [获取代码](#获取代码)
    2. [使用 Singularity 运行](#使用-singularity-运行)
    3. [从命令行运行任务](#从命令行运行任务)
    4. [理解控制台输出](#理解控制台输出)
    5. [结果可视化](#结果可视化)
3. [附加信息](#附加信息)
    1. [创建新领域](#创建新领域)
    2. [安装 Python 依赖](#安装-python-依赖)
    3. [构建 OCaml 二进制文件](#构建-ocaml-二进制文件)
    4. [构建 Rust 压缩器](#构建-rust-压缩器)
    5. [PyPy](#pypy)
4. [软件架构](#软件架构)
5. [`protonet-networks`](#protonet-networks)

# 概述

DreamCoder 是一种"清醒-睡眠"算法，它可以在特定领域内找到解决给定任务集的程序。

# 入门指南

本节将提供运行 DreamCoder 所需的基本信息。

## 用法

### 获取代码

克隆代码库及其子模块。

该代码库有几个 Git 子模块依赖项。

如果您已经克隆了仓库但没有克隆子模块，请运行：
```
git submodule update --recursive --init
```

### 使用 Singularity 运行

如果您不想在本地手动安装所有软件依赖项，可以使用 Singularity 容器。要构建容器，您可以使用仓库中的配方 `singularity`，并在仓库的根目录下运行以下命令（使用 Singularity 2.5 版本测试）：
```
sudo singularity build container.img singularity
```
然后运行以下命令在容器环境中打开一个 shell：
```
./container.img
```
或者，可以通过 `singularity` 命令在容器内运行任务：
```
singularity exec container.img python text.py <命令行参数>
```

### 从命令行运行任务

代码库在 `bin/` 目录下已经定义了一些用于运行特定领域任务的脚本。

通常的使用模式是从仓库的根目录运行脚本，如下所示：
```
python bin/text.py <命令行参数>
```

例如，请参阅以下模块：
 * `bin/text.py` - 在自动生成的文本编辑任务上训练系统。
 * `bin/list.py` - 在自动生成的列表处理任务上训练系统。
 * `bin/logo.py`
 * `bin/tower.py`

通常，如果您使用 `--help` 运行脚本，脚本会说明命令行选项：
```
python bin/list.py --help
```

一个示例训练任务命令可能是：
```
python text.py -t 20 -RS 5000
```
此命令使用 20 秒的枚举超时和识别超时（如果未提供 `-R`，识别超时默认为枚举超时）以及 5000 个识别步骤来运行。

使用 `--testingTimeout` 标志以确保运行测试任务。否则，它们将被跳过。

在 `docs/official_experiments` 文件中查看更多命令示例。

### 理解控制台输出

DreamCoder 脚本的第一个输出——在一些命令行调试语句之后——通常是启动任务的输出，将如下所示：
```
(python) Launching list(int) -> list(int) (1 tasks) w/ 1 CPUs. 15.000000 <= MDL < 16.500000. Timeout 10.201876.
(python) Launching list(int) -> list(int) (1 tasks) w/ 1 CPUs. 15.000000 <= MDL < 16.500000. Timeout 9.884262.
(python) Launching list(int) -> list(int) (1 tasks) w/ 1 CPUs. 15.000000 <= MDL < 16.500000. Timeout 2.449733.
	(ocaml: 1 CPUs. shatter: 1. |fringe| = 1. |finished| = 0.)
	(ocaml: 1 CPUs. shatter: 1. |fringe| = 1. |finished| = 0.)
(python) Launching list(int) -> list(int) (1 tasks) w/ 1 CPUs. 15.000000 <= MDL < 16.500000. Timeout 4.865186.
	(ocaml: 1 CPUs. shatter: 1. |fringe| = 1. |finished| = 0.)
```
MDL（Minimum Description Length，最小描述长度）对应于用于存储程序表达式的定义语言所使用的空间。可以通过更改 `-t` 超时选项来调整 MDL，这将改变算法运行的时间长度以及其解决任务的性能。

脚本的下一阶段将显示算法是否能够将程序与任务匹配。`HIT` 表示匹配，`MISS` 表示未能为任务找到合适的程序：
```
Generative model enumeration results:
HIT sum w/ (lambda (fold $0 0 (lambda (lambda (+ $0 $1))))) ; log prior = -5.545748 ; log likelihood = 0.000000
HIT take-k with k=2 w/ (lambda (cons (car $0) (cons (car (cdr $0)) empty))) ; log prior = -10.024556 ; log likelihood = 0.000000
MISS take-k with k=3
MISS remove eq 3
MISS keep gt 3
MISS remove gt 3
Hits 2/6 tasks
```

程序输出还包含一些关于算法尝试用来解决每个任务的程序的信息：
```
Showing the top 5 programs in each frontier being sent to the compressor:
add1
0.00    (lambda (incr $0))

add2
-0.29   (lambda (incr2 $0))
-1.39   (lambda (incr (incr $0)))

add3
-0.85   (lambda (incr (incr2 $0)))
-0.85   (lambda (incr2 (incr $0)))
-1.95   (lambda (incr (incr (incr $0))))
```

程序将循环执行多个"清醒"和"睡眠"阶段的迭代（由 `-i` 标志控制）。值得注意的是，每次迭代后，脚本将以 Python 的"pickle"数据格式导出检查点文件。
```
Exported checkpoint to experimentOutputs/demo/2019-06-06T18:00:38.264452_aic=1.0_arity=3_ET=2_it=2_MF=10_noConsolidation=False_pc=30.0_RW=False_solver=ocaml_STM=True_L=1.0_TRR=default_K=2_topkNotMAP=False_rec=False.pickle
```

这些 pickle 检查点文件是另一个脚本的输入，该脚本可以绘制算法结果的图表，这将在下一节中讨论。

### 结果可视化

`bin/graphs.py` 脚本可用于绘制程序输出结果的图表，这比尝试解释 pickle 文件和控制台输出要容易得多。

一个示例调用如下：
```
python bin/graphs.py --checkpoints <pickle_file> --export test.png
```
该脚本接受一个 pickle 文件的路径和一个用于保存图表的图像导出路径。

以下是输出示例：
![示例输出](./docs/example.png "示例结果")

该脚本还有许多其他选项，可以通过 `--help` 命令查看。

如果 `bin/graphs.py` 脚本抱怨缺少依赖项，请参阅下面的 [安装 Python 依赖](#安装-python-依赖) 部分。

此外，在某些情况下会发生以下错误：
```
feh ERROR: Can't open X display. It *is* running, yeah?
```
如果您看到该错误，可以在运行 `graphs.py` 之前在终端中运行以下命令来解决此问题：
```
export DISPLAY=:0
```

## 附加信息

本节包含附加信息，例如重建 OCaml 二进制文件的步骤，或扩展 DreamCoder 以解决新领域中的新问题。

### 创建新领域

要创建新的问题领域以供解决，必须完成一些事情。按照 [创建新领域](./docs/creating-new-domains.zh.md) 中的步骤开始。

请注意，创建新领域后，如果编辑了任何 OCaml 代码，则需要重新构建 OCaml 二进制文件。有关更多信息，请参阅 [构建 OCaml 二进制文件](#构建-ocaml-二进制文件)。

### 安装 Python 依赖

如果您想从本地计算机运行某些脚本（例如 `bin/graphs.py`），安装 Python 依赖项会很有用。

要安装 Python 依赖项，请激活虚拟环境（或不激活）并运行：
```
pip install -r requirements.txt
```

对于 macOS，安装 `requirements.txt` 中的这些库需要一些额外的系统依赖项。要在本地安装这些系统依赖项，您可以使用 homebrew 下载它们：
```
brew install swig
brew install libomp
brew cask install xquartz
brew install feh
brew install imagemagick
```

### 构建 OCaml 二进制文件

如果您为新的任务领域引入了新的基本类型，或出于任何原因修改了 OCaml 代码库（在 `solvers/` 中），则在重新运行 Python 脚本之前，需要重新构建 OCaml 二进制文件。

要重新构建 OCaml 二进制文件，请从仓库的根目录运行以下命令：
```
make clean
make
```

如果您没有在 Singularity 容器内运行，则需要先安装 OCaml 库依赖项。目前，为了在一个新的 opam switch 上构建求解器，需要以下包（来自 Arch x64 的经验数据，假设您有 `opam`）：
```bash
opam update                 # 说真的，执行这个
opam switch 4.06.1+flambda  # caml.inria.fr/pub/docs/manual-ocaml/flambda.html
eval `opam config env`      # *叹气*
opam install ppx_jane core re2 yojson vg cairo2 camlimages menhir ocaml-protoc zmq
```

现在尝试在根文件夹中运行 `make`，它应该会构建几个 OCaml 二进制文件。

### 构建 Rust 压缩器

获取 Rust（例如，根据 [https://www.rust-lang.org/](https://www.rust-lang.org/en-US/install.html) 运行 `curl https://sh.rustup.rs -sSf | sh`）

现在在 `rust_compressor` 文件夹中运行 `make` 应该会安装正确的包并构建二进制文件。

### PyPy

如果出于某种原因您想在 PyPy 中运行某些东西，请从以下位置安装：
```
https://github.com/squeaky-pl/portable-pypy#portable-pypy-distribution-for-linux
```
确保将 `pypy3` 添加到路径中。不过，您真的应该尝试使用 Rust 压缩器和 OCaml 求解器。即使您已经在 Python 端安装了并行库，您仍然需要（烦人地）在 PyPy 端安装它们：

```
pypy3 -m ensurepip
pypy3 -m pip install --user vmprof
pypy3 -m pip install --user dill
pypy3 -m pip install --user psutil
```

## 软件架构

要更好地了解 DreamCoder 的内部工作原理，请参阅 [软件架构](./docs/software-architecture.md) 文档。

## `protonet-networks`

### `protonet` 代码（大部分）的来源说明

`protonet-networks` 文件夹包含对来自[此仓库](https://github.com/jakesnell/prototypical-networks) 的大量代码的一些修改，以下是归属信息：

> NIPS 2017 论文 [Prototypical Networks for Few-shot Learning](http://papers.nips.cc/paper/6996-prototypical-networks-for-few-shot-learning.pdf) 的代码

如果您使用该部分代码，请引用他们的论文，并查看他们的工作：

```bibtex
@inproceedings{snell2017prototypical,
  title={Prototypical Networks for Few-shot Learning},
  author={Snell, Jake and Swersky, Kevin and Zemel, Richard},
  booktitle={Advances in Neural Information Processing Systems},
  year={2017}
}
```

### `protonets-networks` 文件夹的许可证

MIT 许可证

版权所有 (c) 2017 Jake Snell

特此免费授予任何获得本软件及相关文档文件（"软件"）副本的人以下权利，可以不受限制地处理本软件，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或销售软件副本的权利，并允许向其提供软件的人员这样做，但须遵守以下条件：

上述版权声明和本许可声明应包含在软件的所有副本或主要部分中。

本软件按"原样"提供，不作任何明示或暗示的保证，包括但不限于适销性、特定用途适用性和非侵权的保证。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任承担任何责任，无论是在合同诉讼、侵权行为还是其他方面，由软件或软件的使用或其他交易引起或与之相关。 