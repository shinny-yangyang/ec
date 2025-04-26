# DreamCoder - LOGO 领域

该目录包含 DreamCoder 的 LOGO 绘图领域实现。

## 目标

LOGO 领域的目标是让 DreamCoder **学习如何通过组合一系列基本的绘图指令来生成指定的二维图形**。这模仿了经典的 LOGO 编程语言，其中"海龟"根据指令在屏幕上移动并绘制线条。

具体来说，该领域涉及：

*   **程序合成 (Program Synthesis)**: 自动生成能够绘制目标图形的 LOGO 指令序列。
*   **视觉识别 (Visual Recognition)**: 使用卷积神经网络 (CNN) 从目标图形图像中提取特征，以指导程序合成过程。
*   **学习绘图概念**: 通过解决一系列任务，学习通用的绘图模式、结构和编程概念（如循环、子程序）。

## 基本指令 (Primitives)

LOGO 程序由 `logoPrimitives.py` 中定义的一组基本指令构成，主要包括：

*   **核心绘图**:
    *   `logo_FWRT length angle turtle -> turtle`: 向前移动指定 `length` 并右转指定 `angle`。
    *   `logo_PT (turtle -> turtle) (turtle -> turtle)`: 控制画笔状态（提起/放下），可能通过传入的函数定义行为。
    *   `logo_GETSET (turtle -> turtle) turtle -> turtle`: 获取、修改并设置海龟状态。
*   **控制流**:
    *   `logo_forLoop count (int -> turtle -> turtle) turtle -> turtle`: 执行循环。
*   **常量**:
    *   单位/零/极小长度和角度 (`logo_UA`, `logo_UL`, `logo_ZA`, `logo_ZL`, `logo_epsA`, `logo_epsL`)。
    *   整数常量 (0-9, `logo_IFTY`)。
*   **算术运算**:
    *   对角度 (`tangle`) 和长度 (`tlength`) 进行加、减、乘、除运算。

## 任务 (Tasks)

`makeLogoTasks.py` 文件定义了一系列具体的绘图任务，作为 DreamCoder 的训练和测试目标。这些任务覆盖了广泛的图形类型：

*   **基本形状**: 线段、直角、正多边形 (3-8边)。
*   **曲线与螺旋**: 圆、半圆、平滑螺旋线、阶梯螺旋线。
*   **重复与组合图案**:
    *   **花朵**: 通过旋转重复基本单元（叶子、圆弧）构成。
    *   **雪花**: 通过旋转重复基本"臂"状图形构成。
    *   **序列**: 将图形沿直线重复排列（如一排圆圈、一排虚线）。
    *   **组合**: 将不同图形并排放置（如线段挨着半圆）。
    *   **网格**: 绘制 2x2 网格等。
    *   **同心**: 绘制同心方块、同心圆等。
    *   **嵌入式**: 在绘制过程中调用其他绘图程序。
*   **带画笔控制的图形**: 虚线、不连续图案等。

这些任务旨在测试 DreamCoder 学习基本绘图能力以及更高级编程概念（循环、子程序、参数化、对称性）的能力。 