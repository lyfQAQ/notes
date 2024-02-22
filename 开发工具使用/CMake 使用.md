# Basic

**`cmake -B <build tree> -S <source tree>`**

**`cmake --build <build tree>`**

- **build tree** is the path to target/output directory,
- **source tree** is the path at which your source code is located.

CMake本身无法构建任何东西——它依赖于特定系统中的其他工具来执行实际的编译、链接和其他任务。你可以将其视为你构建过程的策划者：它知道需要执行哪些步骤，最终目标是什么，以及如何找到正确的工人和材料来完成工作。

这个过程有三个阶段：

- 配置
- 生成
- 构建

## 配置阶段

这个阶段涉及读取源目录的细节信息，并为生成阶段准备一个输出目录（build tree）。

首先创建空的 build tree，并收集 OS 环境的所有信息（编译器、架构等），还检查能否编译一个简单的测试程序。

接着，CMake 解析并执行 CMakeLists.txt。该文件包含项目结构、其目标和依赖关系（库和其他CMake包）的信息。在此过程中，CMake将收集到的信息存储在 build tree 中，例如系统详细信息、项目配置、日志和临时文件。

## 生成阶段

为特定 OS 环境**生成构建系统，就是针对其他构建工具，例如GNU Make的Makefile或Ninja以及Visual Studio的IDE项目文件的定制配置文件。**

## 构建阶段

运行特定的构建工具来生成目标

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/92882367-3ea8-472a-880f-875f6ab3b8d4/5fce4fcb-27f0-44db-90ec-69847e82915a/Untitled.png)

```bash
cmake -B buildtree      // 在 buildtree 目录生成构建系统
cmake --build buildtree // 开始构建

docker pull swidzinski/cmake:examples // 拉取目标镜像
docker run -it swidzinski/cmake:examples // 运行
```

# CMake

推荐使用`cmake -B ./build -S ./project` ，该命令根据`./project` 中的源码文件生成一个构建系统，放置到`./build` 目录中

## Building a project

```bash
cmake --build <dir> [<options>] [-- <build-tool-options>]
```

此处 `dir` 就是`-B` 指定的构建系统所在目录如`build` ，`<build-tool-options>`是特定平台构建工具的参数。

####多进程并行构建

```bash
cmake --build <dir> --parallel [<number-of-jobs>]
cmake --build <dir> -j [<number-of-jobs>]
```

### 目标选项

每个项目由多个目标组成，然而可以只生成特定目标：

```bash
cmake --build <dir> --target <target1> -t <target2> -t <target3> ...
```

未构建的目标称为`clean` ，可以删除其在构建目录 build 中的相关内容：

```bash
cmake --build <dir> -t clean
```

### 多配置生成器

选择`Debug`、`Release`、`MinSizeRel`或`RelWithDebInfo`，并按以下方式指定：

```bash
cmake --build <dir> --config <cfg>
```

CMake 默认是 `Debug`

### 详细日志

```bash
cmake --build <dir> --verbose
cmake --build <dir> -v
```

## Installing a project

```bash
cmake --install <dir>
cmake --install <dir> [<options>]
```

### 安装组件

```bash
cmake --install <dir> --component <comp>
```

### 安装目录

在项目配置中指定的安装路径之前添加一个自定义的前缀。例如，如果项目配置指定的安装路径是 **`/usr/local`**，而您选择的前缀是 **`/home/user`**，那么安装后的路径将变为**`/home/user/usr/local`**

```bash
cmake --install <dir> --prefix <prefix>
```

# CTest

# 项目文件结构

## 项目根目录

- 根目录必须提供 CMakeLists.txt 配置文件
- 使用cmake命令的-S参数指定该目录的路径

## 构建目录

如 build

- 创建二进制文件，例如可执行文件和库文件，以及用于最终链接的目标文件和归档文件
- 不要将此目录添加到您的版本控制系统（VCS）中，它是特定于您的系统的。如果您决定将其放在源树内部，请确保将其添加到VCS忽略文件中。
- 建议生成的构建系统产物存放在与所有源文件分离的目录中
- 建议项目包含一个安装阶段，允许将最终的构建产物放置在系统中的正确位置，以便删除所有用于构建的临时文件

## Listfiles

包含 CMake 语言的文件，具有`.cmake`扩展名，它们可以通过include()和find_package()、add_subdirectory()来包含

## CMakeLists.txt

处于项目根目录中的该文件是配置阶段首先执行的文件，它应该至少包含两个命令： • `cmake_minimum_required(VERSION <x.xx>)`：设置所需的CMake版本 • `project(<name> <OPTIONS>)`：用于为项目命名（提供的名称将存储在`PROJECT_NAME`变量中）

可以讲项目划分成更小的单元，设置顶级 CMakeLists.txt 的内容：

```bash
cmake_minimum_required(VERSION 3.20)
project(app)
message("Top level CMakeLists.txt")
**add_subdirectory(api) #包含 api 目录的 CMakeLists.txt**
```

下面是某项目结构：

```bash
CMakeLists.txt
**api/CMakeLists.txt
api/api.h
api/api.cpp**
```

## CMakeCache.txt