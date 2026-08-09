# Unreal Build Tool (UBT)

Unreal Engine 使用 C++ 代码编写，C++ 源代码分为头文件和源文件两类。源代码编译后，可以生成二进制的可执行文件（*.exe）、动态链接库文件（*.dll）或静态链接库文件（*.lib）。

C++ 代码的编译通常分为三步：
1. 预处理
2. 编译
3. 链接

Unreal Engine 没有使用 MSBuild 或 CMake 等常用的构建工具编写构建规则，而是使用 C# 编写了一个全新的构建工具 —— UBT。

[Unreal Build Tool（UBT）](https://dev.epicgames.com/documentation/unreal-engine/unreal-build-tool-in-unreal-engine?application_version=5.5)是 Epic 官方提供的构建引擎和游戏工程的跨平台构建工具