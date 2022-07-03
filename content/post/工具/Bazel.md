---
title: Bazel
date: 2022-07-01T18:54:13+08:00
categories:
    - 工具
tags:
    - C++
    - Bazel
---

# 什么是 Bazel？

## 简介

Bazel 是一个开源构建和测试工具，类似于 Make、Maven 和 Gradle。它使用人类可读的简要构建语言。Bazel 支持多种语言的项目，并针对多个平台构建输出。Bazel 支持跨多个代码库和大量用户的大型代码库。



## 特点

- **高级构建语言**：使用人类可读的抽象语言在高度语义上描述项目的构建属性。与其他工具不同，Bazel 会根据库、二进制文件、脚本和数据集的概念进行操作，避免了编写对编译器和连接器等工具进行单独调用的复杂性。
- **快速而又可靠**：借助高级的本地和分布式缓存来记录所有已经完成的工作，并跟踪对文件内容和构建命令所做的更改。这样，Bazel 就能知道何时需要重新构建某些内容，并且仅重建必要的内容。如需进一步加快构建速度，您可以将项目设置为以高度并行且增量的方式进行构建。
- **跨平台**：可以在 Linux、macOS 和 Windows 上运行。Bazel 可以在同一个项目中为多个平台（包括桌面端、服务器和移动端）构建二进制文件和可部署软件包。
- **可伸缩**：在处理具有 10 万多个源文件的 build 时，Bazel 也能够保持敏捷性。Bazel 还可以帮助你扩展你的组织、代码库和持续集成系统，它可与数以万计的代码库和用户群进行协作。
- **可拓展**：支持如 C++、Java、Android、iOS、Go和其他各种语言平台，并且可以使用 Bazel 熟悉的扩展语言轻松添加对新语言和平台的支持，分享和重用由不断增长的 Bazel 社区编写的语言规则。



## 使用流程

如果需要使用 Bazel 构建或测试项目，通常需要执行以下操作：

1. **设置 Bazel**。下载并安装 Bazel。
2. **设置项目的 WORKSPACE**。这是 Bazel 用于查找构建输入、BUILD 文件的目录，以及存储构建输出的目录。
3. **编写 BUILD 文件**。告诉 Bazel 应该构建什么以及如何去构建。
4. **通过命令行运行 Bazel**。运行后 Bazel 会将结果输出至 WORKSPACE 中。



## 构建流程

当运行构建或测试时，Bazel 会执行以下操作：

1. 加载与目标相关的 BUILD  文件。
2. 分析输入及其依赖项，应用指定的构建规则，并生成操作图。
3. 对输入执行构建操作，直到生成最终构建输出。



## 基本概念

### WORKSPACE（工作区）

工作区是一个目录，它包含了构建目标所需要的源码文件。每个工作区中都有一个名为 WORKSPACE 的文本文件，该文件可能为空，也可能包含构建输出所需的外部依赖的引用。包含名为 WORKSPACE 的文件的目录被视为工作区的根。

代码以代码库形式组织。包含 WORKSPACE 文件的目录是主代码库的根目录，也称为 @。其他（外部）代码库是使用工作区规则在 WORKSPACE 文件中定义的。



### PACKAGE（包）

包是工作区中主要的代码组织单元，其中包含一系列相关的文以及描述这些文件之间关系的 BUILD 文件。

包是工作区的子目录，它的根目录必须包含文件 BUILD（或 BUILD.bazel）。除了那些具有 BUILD 文件的子包以外，其它子目录属于包的一部分。



### TARGET（目标）

包是目标的容器，所以目标在包的 BUILD 文件中定义。 目标主要分为以下几种类型：

- **File（文件）**：

  - **源文件（Source File）**：由相关人员编写的，并签入到代码库中。
  - **派生文件（Derived files）**：又叫输出文件，是构建工具依据规则自动生成的文件。

- **Rule（规则）**：每个规则实例均指定一组输入文件和输出文件之间的关系。 规则的输入可以是源文件，但也可以是其他规则的输出。

  - 在BUILD中声明规则的语法时：

    ```shell
    规则类型(
        name = "...",  #必须
        其它属性 = ...	#可选
    )
    ```

  - 规则的类型一般以编程语言为前缀，例如 cc，java，后缀通常有：

    - `*_binary`：用于构建目标语言的可执行文件

    - `*_test`：用于自动化测试，其目标是可执行文件，如果测试通过应该返回零。

    - `*_library`：用于构建目标语言的库 

- **PACKAGE GROUP（包组）**：即一组包，主要用于限制某些特定规则的可见性。由 `package_group` 函数定义。它们有三个属性：它们包含的包列表、名称以及它们包含的其他包组。引用它们的唯一方法来自规则的 `visibility` 属性或 `package` 函数的 `default_visibility` 属性，且它们不会生成或使用文件。

规则生成的文件与规则本身始终属于同一个包，即无法将文件生成到其他包中。但是规则的输入可以来自其他包。



### LABEL（标签）

目标的名称又被称为标签。每个标签都唯一标识一个目标。规范格式的典型标签如下所示：

```shell
@myrepo//my/app/main:app_binary
```

以上面的标签为例，它由三部分组成

- **代码库名**：标签的第一部分是代码库名称，例如上面的 `@myrepo//`。如果标签引用了其使用的同一个代码库，其代码库标识符可以缩写为 //，例如：

  ```shell
  //my/app/main:app_binary
  ```

- **包名**：标签的第二部分是包名称 `my/app/main`，这是相对于代码库根目录的包路径。如果标签引用了其使用的同一个包，则可以省略包名称（还可以选择是否使用英文冒号进行分隔），例如：

  ```shell
  #两者等价
  app_binary
  :app_binary
  ```

- **目标名**：标签后面的冒号部分 app_binary 是目标名称。当与软件包路径的最后一个组成部分匹配时，可以省略冒号和冒号，例如：

  ```shell
  #两者等价
  //my/app/lib
  //my/app/lib:lib
  ```

  在 BUILD 文件中，引用当前包中定义的规则时，冒号不能省略。而引用当前包中文件时，冒号可以省略。 但是，从其它包引用时、从命令行引用时，都必须使用完整的标签。

  

### BUILD（构建文件）

每个包中都包含一个 BUILD 文件，而 BUILD 文件中的构建规则的每个实例又称为目标，并指向一组特定的源文件和依赖项。 目标还可以指向其他目标。

BUILD 文件包含几种不同类型的 Bazel 指令。最重要的类型是构建规则，告知 Bazel 如何构建所需的输出，例如可执行二进制文件或库。BUILD 文件中的语句会被从上而下的逐条解释，某些语句的顺序很重要， 例如变量必须先定义后使用，但是规则声明的顺序无所谓。

为了促进代码和数据之间的完全分离，BUILD 文件不能包含函数定义、for 语句或 if 语句，但可以在 .bzl 文件中声明函数、控制结构，并在BUILD文件中用 `load` 语句加载。例如：

```shell
load("//foo/bar:file.bzl", "some_library")
```

上面的语句加载 `foo/bar/file.bzl` 并添加其中定义的符号 `some_libraray` 到当前环境中，`load `语句可以用来加载规则、函数、常量（字符串、列表等）。（==参数必须是字符串字面量（无变量），并且 load 语句必须出现在顶层，它们不能位于函数正文中。==）



### DEPENDENCY（依赖项）

如果在构建或执行时 A 需要目标 B，目标 A 会依赖于目标 B。依存关系会使目标产生有向无环图（DAG），这就是所谓的依赖图。

**直接依赖项**是指依赖项图表中长度为 1 的路径可以访问的其他目标。 **传递依赖项**是指它通过图中任意长度的路径来依赖的那些目标。

大多数构建规则都具有三个用于指定不同类型通用依赖项的属性：srcs、deps 和 data。

- **srcs**：直接输出源文件的规则所使用的文件。
- **deps**：指向独立编译的模块的规则，这些模块提供头文件、符号、库、数据等。
- **data**：构建目标可能需要一些数据文件才能正常运行。这些数据文件不是源代码：它们不会影响目标的构建方式。



# 实战

## 环境搭建

### 安装 Bazel

这里以 CentOS 8 为例：

1. 首先从 Fedora COPR 下载相应的 .repo 文件并将其复制到 /etc/yum.repos.d/ 中：

   ```shell
   wget --no-check-certificate -P /etc/yum.repos.d/ https://copr.fedorainfracloud.org/coprs/vbatts/bazel/repo/epel-7/vbatts-bazel-epel-7.repo
   ```

2. 使用 yum 安装 Bazel：

   ```shell
   yum -y install bazel4
   ```

3. 查看 Bazel 版本号，看看是否安装成功：

   ```shell
   bazel --version
   ```

   

### 安装示例项目

从 Bazel 的 GitHub 中下载官方提供的示例项目： 

```shell
git clone https://github.com/bazelbuild/examples
```

示例项目位于 `examples/cpp-tutorial` 目录，结构如下：

```shell
examples
└── cpp-tutorial
    ├──stage1
    │  ├── main
    │  │   ├── BUILD
    │  │   └── hello-world.cc
    │  └── WORKSPACE
    ├──stage2
    │  ├── main
    │  │   ├── BUILD
    │  │   ├── hello-world.cc
    │  │   ├── hello-greet.cc
    │  │   └── hello-greet.h
    │  └── WORKSPACE
    └──stage3
       ├── main
       │   ├── BUILD
       │   ├── hello-world.cc
       │   ├── hello-greet.cc
       │   └── hello-greet.h
       ├── lib
       │   ├── BUILD
       │   ├── hello-time.cc
       │   └── hello-time.h
       └── WORKSPACE
```

示例主要分为三个阶段：

- **stage1**：构建位于单个软件包中的单个目标。
- **stage2**：将项目拆分为多个目标，但将其保留在单个软件包中。
- **stage3**：将项目拆分为多个软件包，并使用多个目标进行构建。



## C++ 项目构建示例

### 基础知识

#### 构建项目

想要构建项目，只需要在命令行输入 `bazel build` 命令，并在其后面指定标签名，例如：

```shell
bazel build 标签名
```

当构建执行成功时，就会在终端输出提示信息，其中包括每个目标构建输出的位置、构建时间、关键路径等，具体的输出信息可以参考下面的几个示例。

当构建完成后，可以使用 `bazel clean` 命令清除构建结果。



#### 查看依赖项图表

成功的构建会将其所有依赖项明确列在 BUILD 文件中。Bazel 会使用这些语句创建项目的依赖项图，以实现准确的增量构建。

如需直观呈现示例项目的依赖项，您可以通过在工作区根目录下运行以下命令，生成依赖项图的文本表示形式：

```shell
bazel query --notool_deps --noimplicit_deps "依赖项属性(目标)" \
  --output graph
```

上述命令指示 Bazel 查找目标的所有依赖项（不包括主机和隐式依赖项），并将输出格式化为图表。接着将文本粘贴到 [WebGraphviz](http://www.webgraphviz.com/) 中即可查看。



### 示例一：编译单个目标

首先我们需要查看示例一的 BUILD 文件，规则可参考该链接 [cc\_library 规则](https://bazel.build/reference/be/c-cpp#cc_library)。文件内容如下：

```shell
load("@rules_cc//cc:defs.bzl", "cc_binary")

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
)
```

可以看到，依赖关系非常简单，只需要编译一个 TARGET，并且这个 TARGET 也只依赖于 hello-world.cc。直接运行构建命令：

```shell
bazel build //main:hello-world
```

此时 Bazel 生成以下输出信息：

```shell
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (15 packages loaded, 57 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 14.070s, Critical Path: 1.19s
INFO: 6 processes: 4 internal, 2 processwrapper-sandbox.
INFO: Build completed successfully, 6 total actions
```

执行生成的二进制文件，看看是否能够正常运行：

```shell
bazel-bin/main/hello-world

#输出
Hello world
Sun Jun  5 18:14:07 2022
```

接着生成示例项目的依赖项图表：

```shell
bazel query --notool_deps --noimplicit_deps "deps(//main:hello-world)" \
  --output graph
```

此时输出依赖项文本如下：

```shell
digraph mygraph {
  node [shape=box];
  "//main:hello-world"
  "//main:hello-world" -> "//main:hello-world.cc"
  "//main:hello-world.cc"
}
```

将其复制到 [WebGraphviz](http://www.webgraphviz.com/) 中，得到的依赖项图表如下：

![](http://img.orekilee.top//imgbed/bazel/bazel1.png)

<center>hello-world 的依赖关系图会显示带有单个源文件的单个目标。</center>

如上图，示例一仅具有一个目标，该目标会构建没有其他依赖项的单个源文件。



### 示例二：编译多个目标

接着我们查看示例二的 BUILD 文件：

```shell
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
    ],
)
```

在这个 BUILD 文件中存在两个 TARGET，分别是库文件 hello-greet 与 可执行文件 hello-world。由于 hello-world 目标中的 deps 属性说明该目标的构建依赖于 hello-greet，因此在构建时会先构建 hello-greet 库，再根据这个库构建 hello-world。

执行下面的命令开始构建：

```shell
bazel build //main:hello-world
```

输出结果如下：

```shell
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (15 packages loaded, 60 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 11.865s, Critical Path: 0.25s
INFO: 7 processes: 4 internal, 3 processwrapper-sandbox.
INFO: Build completed successfully, 7 total actions
```

执行生成的二进制文件，看看是否能够正常运行：

```shell
bazel-bin/main/hello-world

#输出
Hello world
Sun Jun  5 20:01:37 2022
```

接着生成示例项目的依赖项图表：

![](http://img.orekilee.top//imgbed/bazel/bazel2.png)

<center>hello-world 的依赖关系图显示了对文件进行修改后的结构变化。</center>

如上图，示例二构建包含两个目标的项目。hello-world 目标会构建一个源文件，并依赖于另一个目标 hello-greet，后者会构建两个额外的源文件。



### 示例三：编译多个包

由于示例三中需要对多个包进行编译，所以我们首先使用 `tree` 命令来看看它们的目录结构：

```shell
.
├── lib
│   ├── BUILD
│   ├── hello-time.cc
│   └── hello-time.h
├── main
│   ├── BUILD
│   ├── hello-greet.cc
│   ├── hello-greet.h
│   └── hello-world.cc
├── README.md
└── WORKSPACE
```

可以看到，在工作区中包含了两个子目录 lib 和 main，并且这两个子目录中都有 BUILD 文件，因此它们都是包。

接着查看这两个包中的 BUILD 文件：

```shell
#lib/BUILD
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "hello-time",
    srcs = ["hello-time.cc"],
    hdrs = ["hello-time.h"],
    visibility = ["//main:__pkg__"],
)

#main/BUILD
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
        "//lib:hello-time",
    ],
)
```

我们发现，这两个包中存在跨包依赖。main 包中的 hello-world 不仅依赖于本包的 hello-greet，还依赖于 lib 包中的 hello-time。因此我们需要确保 lib 包中的 hello-time 对 main 包中的 hello-world 可见（默认情况下仅对同一 BUILD 文件中的其他目标可见，即可见性为 `//visibility:private`），这时就需要在 hello-time 目标中加上 visibility 属性，并将可见性设置为 `//main:__pkg__`，即对 main 包可见。

执行下面的命令开始构建：

```shell
bazel build //main:hello-world
```

输出结果如下：

```shell
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (16 packages loaded, 63 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 8.273s, Critical Path: 0.26s
INFO: 8 processes: 4 internal, 4 processwrapper-sandbox.
INFO: Build completed successfully, 8 total actions
```

执行生成的二进制文件，看看是否能够正常运行：

```shell
bazel-bin/main/hello-world

#输出
Hello world
Sun Jun  5 20:26:44 2022
```

接着生成示例项目的依赖项图表：

![](http://img.orekilee.top//imgbed/bazel/bazel3.png)

<center>hello-world 的依赖关系图显示了 main 包中的目标如何依赖于 lib 软件包中的目标。</center>



## C++ 常见构建方法

###  在目标中包含多个文件

> Glob 是一个辅助函数，可用于查找符合特定路径模式的所有文件，并返回一个包含可变排序的新路径列表。Glob 仅在自己的包中搜索文件，并且仅查找源文件（而不是生成的文件或其他目标）。

如果想在单个目标中包含多个文件，可以使用 `glob` 函数来进行文件匹配，例如：

```shell
cc_library(
    name = "build-all-the-files",
    srcs = glob(["*.cc"]),
    hdrs = glob(["*.h"]),
)
```

使用此目标时，Bazel 将在其找到的包含此目标的 BUILD 文件所在的目录（不包括子目录）中构建找到的所有 .cc 和 .h 文件。



### 使用传递性包含

如果文件包含一个头文件，那么任何以该文件为源的规则（即在 srcs、hdrs 或 textual_hdrs 属性中有该文件）都应该依赖于所包含的头文件的库规则。反过来说，只有直接依赖项才需要指定为依赖项。

假设 sandwich.h 包含 bread.h，bread.h 包含 flour.h。sandwich.h 不包括 flour.h，因此 BUILD 文件将如下所示：

```shell
cc_library(
    name = "sandwich",
    srcs = ["sandwich.cc"],
    hdrs = ["sandwich.h"],
    deps = [":bread"],
)

cc_library(
    name = "bread",
    srcs = ["bread.cc"],
    hdrs = ["bread.h"],
    deps = [":flour"],
)

cc_library(
    name = "flour",
    srcs = ["flour.cc"],
    hdrs = ["flour.h"],
)
```

此时 sandwich 库依赖于 bread 库，又 bread 依赖于 flour 库。



### 添加包含路径

如果不想在工作区的根目录中设置包含（include）路径。而现有的库可能已经有了一个 include 目录，但是它却与工作区中的路径不一致，例如下面目录结构：

```shell
└── my-project
    ├── legacy
    │   └── some_lib
    │       ├── BUILD
    │       ├── include
    │       │   └── some_lib.h
    │       └── some_lib.cc
    └── WORKSPACE
```

Bazel 应该包含 some_lib.h 作为 legacy/some_lib/include/some_lib.h，但假设 some_lib.cc 包含 some_lib.h。要使该包含路径有效，legacy/some_lib/BUILD 需要在 copts 属性中指定 some_lib/include 目录是包含目录：

````shell
cc_library(
    name = "some_lib",
    srcs = ["some_lib.cc"],
    hdrs = ["include/some_lib.h"],
    copts = ["-Ilegacy/some_lib/include"],
)
````

这对于外部依赖项尤其有用，因为其头文件必须包含 / 前缀。



### 包括外部库

> gtest（Google Test）是一个跨平台的（Liunx、Mac OS X、Windows等）C++单元测试框架，由 Google发布。gtest 是为在不同平台上为编写 C++ 测试而生成的。它提供了丰富的断言、致命和非致命判断、参数化、死亡测试等等。

当我们需要引入一个外部库时，可以使用 WORKSPACE 文件中的某个代码库函数下载，并代码库中提供它，这里以 gtest 为例：

```shell
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "gtest",
    url = "https://github.com/google/googletest/archive/release-1.10.0.zip",
    sha256 = "94c634d499558a76fa649edb13721dce6e98fb1e7018dfaeba3cd7a083945e91",
    build_file = "@//:gtest.BUILD",
)
```

接着创建 gtest 的 BUILD 文件，由于 gtest 具有几项特殊要求，使得其 cc_library 规则更加复杂：

- `googletest-release-1.10.0/src/gtest-all.cc` `#include` 对 `googletest-release-1.10.0/src/` 中的所有其他文件执行排除操作：将其从编译中排除，以防止出现重复符号的链接错误。
- 它使用相对于 `googletest-release-1.10.0/include/` 目录 (gtest/gtest.h) 的头文件，因此必须将该目录添加到包含路径。
- 它需要关联 pthread，因此需要添加为 linkopt。

规则将如下所示：

```shell
cc_library(
    name = "main",
    srcs = glob(
        ["googletest-release-1.10.0/src/*.cc"],
        exclude = ["googletest-release-1.10.0/src/gtest-all.cc"]
    ),
    hdrs = glob([
        "googletest-release-1.10.0/include/**/*.h",
        "googletest-release-1.10.0/src/*.h"
    ]),
    copts = [
        "-Iexternal/gtest/googletest-release-1.10.0/include",
        "-Iexternal/gtest/googletest-release-1.10.0"
    ],
    linkopts = ["-pthread"],
    visibility = ["//visibility:public"],
)
```

由于所有内容都以 `googletest-release-1.10.0` 为前缀，是归档结构的副产物。可以通过添加 strip_prefix 属性让 http_archive 去除此前缀：

```shell
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "gtest",
    url = "https://github.com/google/googletest/archive/release-1.10.0.zip",
    sha256 = "94c634d499558a76fa649edb13721dce6e98fb1e7018dfaeba3cd7a083945e91",
    build_file = "@//:gtest.BUILD",
    strip_prefix = "googletest-release-1.10.0",
)
```

此时 gtest.BUILD 可以简化为：

```shell
cc_library(
    name = "main",
    srcs = glob(
        ["src/*.cc"],
        exclude = ["src/gtest-all.cc"]
    ),
    hdrs = glob([
        "include/**/*.h",
        "src/*.h"
    ]),
    copts = ["-Iexternal/gtest/include"],
    linkopts = ["-pthread"],
    visibility = ["//visibility:public"],
)
```

编写一个使用 gtest 的测试代码，看看是否能够成功构建：

```cpp
#include "gtest/gtest.h"
#include "main/hello-greet.h"

TEST(HelloTest, GetGreet) {
  EXPECT_EQ(get_greet("Bazel"), "Hello Bazel");
}
```

创建测试文件的 BUILD：

```
cc_test(
    name = "hello-test",
    srcs = ["hello-test.cc"],
    copts = ["-Iexternal/gtest/include"],
    deps = [
        "@gtest//:main",
        "//main:hello-greet",
    ],
)
```

使用 `bazel test` 进行测试：

```shell
bazel test test:hello-test

#输出
INFO: Found 1 test target...
Target //test:hello-test up-to-date:
  bazel-bin/test/hello-test
INFO: Elapsed time: 4.497s, Critical Path: 2.53s
//test:hello-test PASSED in 0.3s

Executed 1 out of 1 tests: 1 test passes.
```



### 添加对预编译库的依赖关系

如果想使用只包含已编译版本的库（例如头文件和 .so 文件），就需要将其封装在 cc_library 规则中，例如：

```shell
cc_library(
    name = "mylib",
    srcs = ["mylib.so"],
    hdrs = ["mylib.h"],
)
```