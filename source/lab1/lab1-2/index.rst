.. lab tutorial documentation master file, created by
   sphinx-quickstart on Mon Sep  4 18:05:09 2023.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

实验目的
========================================

实验1主要讲解的是中断处理机制。通过本章的学习，我们了解了 riscv 的中断处理机制、相关寄存器与指令。我们知道在中断前后需要恢复上下文环境，用 一个名为中断帧（TrapFrame）的结构体存储了要保存的各寄存器，并用了很大篇幅解释如何通过精巧的汇编代码实现上下文环境保存与恢复机制。最终，我们通过处理断点和时钟中断验证了我们正确实现了中断机制。

.. _本节内容:

本节内容
-------------------------------------

.. toctree::
   :maxdepth: 6

   lab1_2_1_exercise
   lab1_2_2_files
