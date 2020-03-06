# 打包应用与开发工具

使用Conan可以打包和部署应用程序，同样支持打包和部署开发工具，如编译器（例如MinGW）或者构建系统（如CMake）。

本节描述如何打包以及运行可执行程序以及开发工具，以及如何依据`build_requires`的描述从源码构建各种开发工具或者库（如测试框架等）。

## 运行以及部署包

使用Conan也可以对包含动态库的可执行程序进行分发和部署。相比较使用其他部署工具，使用Conan具有如下优点：

- Conan可以作为统一的跨系统和平台的开发和分发工具；
- 以统一的方式（包管理的方式）管理大量不同的部署配置；
- 可以使用Conan服务端存储各种系统、平台以及目标的应用程序和运行时；

具体Conan可以使用以下几种不同的方式，对应用程序进行分发和部署。

### 使用虚拟环境（virtual environments）

我们可以创建包含可执行程序的包。我们以默认的`conan new`产生的包模板举例：`conan new Hello/0.1`。

这个源码会产生一个叫做`greet`的可执行程序，但是可执行程序默认并不会被打包。我们可以修改包配置的`package()`函数将可执行程序也打包起来：

```python
def package(self):
    self.copy("*greet*", src="bin", dst="bin", keep_path=False)
```

现在我们可以像以往一样创建包，但是当我们想要运行可执行程序的时候会发现找不到：

```sh
$ conan create . user/testing
...
Hello/0.1@user/testing package(): Copied 1 '.h' files: hello.h
Hello/0.1@user/testing package(): Copied 1 '.exe' files: greet.exe
Hello/0.1@user/testing package(): Copied 1 '.lib' files: hello.lib

$ greet
> ... not found...
```

默认情况下，Conan并不会修改环境，它仅将包创建在本地缓存，而对应的路径不会加入到系统PATH里面，所以greet的可执行程序系统是找不到的。

使用`virtualrunenv`生成器可以产生对应的文件，能够将包的默认二进制路径加到以下需要的路径中：

- 将依赖的lib子目录加到`DYLD_LIBRARY_PATH`环境变量中（为OSX系统中的共享库）；
- 将依赖的lib子目录加入到`LD_LIBRARY_PATH`环境中（为Linux系统中的共享库）；
- 将依赖的bin子目录加入到系统的PATH环境变量中（为可执行程序）；

我们在安装包的时候，指定`virtualrunenv`：

```sh
$ conan install Hello/0.1@user/testing -g virtualrunenv
```

这样就会产生一些文件，可以激活或者去激活需要的环境变量：

```sh
$ activate_run.sh # $ source activate_run.sh in Unix/Linux
$ greet
> Hello World!
$ deactivate_run.sh # $ source deactivate_run.sh in Unix/Linux
```

### Imports

同样可以自定义conanfile(txt或者py的都可以)，在里面使用`imports`段，这样就会把本地缓存中需要的文件拷贝出来。`imports`的具体细节会在后面给出示例。

### 可部署的包

使用`deploy`函数可以定义将包对应的文件或者构建产物拷贝到系统其它地方的用户空间中。我们给前面的例子加上`deploy()`方法：

```python
def deploy(self):
    self.copy("*", dst="bin", src="bin")
```

这时运行`conan create . user/testing`。可以看到Conan将包的可执行程序拷贝到了当前的bin目录下：

```sh
$ conan install Hello/0.1@user/testing
...
> Hello/0.1@user/testing deploy(): Copied 1 '.exe' files: greet.exe
$ bin\greet.exe
> Hello World!
```

在部署的过程中，Conan创建了一个deploy_manifest.txt文件，里面记录了所有部署的文件及其内容的哈希值。

有的时候部署如果不关心构建时的编译器的话，可以为此调整包的package ID：

```python
def package_id(self):
    del self.info.settings.compiler
```

进一步了解更多关于`deploy`函数的用法，参见[deploy文档](https://docs.conan.io/en/latest/reference/conanfile/methods.html#method-deploy)。


### 使用部署生成器（deploy generator）

部署生成器负责生成文件，用来记录所有被拷贝部署的文件和其哈希值。这使得部署的过程变得可重复。下面的命令会将所有的依赖项移除conan缓冲区，将其汇集到一个独立的空间：`conan install . -g deploy`

### 使用json生成器（json generator）

一个更好的方式是使用json生成器：这个生成器在部署时不会将文件拷贝到一个目录中，而是产生一个JSON文件(conanbuildinfo.json)记录所有的依赖信息，包含每个文件在Conan缓冲区中的位置。

```sh
$ conan install . -g json
```

conanbuildinfo.json文件是一个为机器生成的文件，可以使用脚本处理它。下面的代码演示了如何从该文件中读取库以及库的目录：

```python
import os
import json

data = json.load(open("conanbuildinfo.json"))

dep_lib_dirs = dict()
dep_bin_dirs = dict()

for dep in data["dependencies"]:
    root = dep["rootpath"]
    lib_paths = dep["lib_paths"]
    bin_paths = dep["bin_paths"]

    for lib_path in lib_paths:
        if os.listdir(lib_path):
            lib_dir = os.path.relpath(lib_path, root)
            dep_lib_dirs[lib_path] = lib_dir
    for bin_path in bin_paths:
        if os.listdir(bin_path):
            bin_dir = os.path.relpath(bin_path, root)
            dep_bin_dirs[bin_path] = bin_dir
```

Json生成器的好处在于它只记录部署依赖的文件，但是并不会自行拷贝，将选择权交给用户，用户可以根据需要进行选择并将文件拷贝成想要的目录布局。上面的脚本很容易修改得去执行各种过滤，并完成目标任务。

另外，你也可以自己写一些简单的启动脚本，为你的应用程序进行各种信息配置：

```python
executable = "MyApp"  # just an example
varname = "$APPDIR"

def _format_dirs(dirs):
    return ":".join(["%s/%s" % (varname, d) for d in dirs])

path = _format_dirs(set(dep_bin_dirs.values()))
ld_library_path = _format_dirs(set(dep_lib_dirs.values()))
exe = varname + "/" + executable

content = """#!/usr/bin/env bash
set -ex
export PATH=$PATH:{path}
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:{ld_library_path}
pushd $(dirname {exe})
$(basename {exe})
popd
""".format(path=path,
       ld_library_path=ld_library_path,
       exe=exe)
Note
```

### 从包开始运行

如果想要在conanfile中运行某个依赖包中的可执行程序，可以在包配置文件中使用`run_environment=True`参数。它会间接调用`RunEnvironment()`帮助函数。例如我们想要在构建Consumer包的时候执行greet程序：

```python
from conans import ConanFile, tools, RunEnvironment

class ConsumerConan(ConanFile):
    name = "Consumer"
    version = "0.1"
    settings = "os", "compiler", "build_type", "arch"
    requires = "Hello/0.1@user/testing"

    def build(self):
        self.run("greet", run_environment=True)
```

现在为Consumer执行`conan install`和`conan build`，可以看到greet被执行了。

```sh
$ conan install . && conan build .
...
Project: Running build()
Hello World!
```

当然也可以显示的通过对应依赖的路径访问可执行程序，如下例。但是这种方法对于有动态库存在的情况下是不行的。

```python
def build(self):
    path = os.path.join(self.deps_cpp_info["Hello"].rootpath, "bin")
    self.run("%s/greet" % path)
```

因此，使用`run_environment=True`是一个更完整的解决方案。

最后，还有另一种做法，那就是可以将包的可执行程序的bin目录直接添加到系统的PATH中。例如在Hello包的配置中如下修改：

```python
def package_info(self):
    self.cpp_info.libs = ["hello"]
    self.env_info.PATH = os.path.join(self.package_folder, "bin")
```

使用这种方法，如果可执行程序需要的话，我们同样可以定义`DYLD_LIBRARY_PATH`和`LD_LIBRARY_PATH`。

这样消费方的包就会比较简单，可以直接调用可执行程序：

```python
def build(self):
    self.run("greet")
```

### Runtime packages and re-packaging

可以创建只包含可以运行的二进制的包，而将供编译时依赖的头文件和库文件等文件都去掉。比如对于Hello包，通过如下方式就可以做到：

```python
from conans import ConanFile

class HellorunConan(ConanFile):
    name = "HelloRun"
    version = "0.1"
    build_requires = "Hello/0.1@user/testing"
    keep_imports = True

    def imports(self):
        self.copy("greet*", src="bin", dst="bin")

    def package(self):
        self.copy("*")
```

这样的包配置文件具有如下特点：

- 它将`Hello/0.1@user/testing`设置为`build_requires`。这意味着Hello包仅仅被用于构建HelloRun包，一旦构建结束就不再需要Hello包了；

- 它使用`imports()`将所有依赖的可执行文件拷贝出来；

- 它通过设置`keep_imports=True`定义将构建阶段（`build()`函数没有定义，所以用默认的）的产物在构建结束后保留下来；

- `package()`函数将build目录下import出来的文件进行打包；

以下是创建并上传包：

```python
$ conan create . user/testing
$ conan upload HelloRun* --all -r=my-remote
```

安装及运行这个包，可以使用我们前面介绍过的任一方式，例如

```sh
$ conan install HelloRun/0.1@user/testing -g virtualrunenv
# You can specify the remote with -r=my-remote
# It will not install Hello/0.1@...
$ activate_run.sh # $ source activate_run.sh in Unix/Linux
$ greet
> Hello World!
$ deactivate_run.sh # $ source deactivate_run.sh in Unix/Linux
```

### 部署挑战

当部署一个C/C++应用程序的时候，有一些特殊的挑战需要应对。这里是一些常见的挑战以及建议的解决方式。

#### C标准库

一个常见的挑战是，应用程序（无论是C还是C++写的）都可能依赖C的标准库，最常见的就是GNU的C库：glibc。

Glibc其实不仅是C标准库，它包含以下内容：

- C函数，如`malloc()`、`sin()`等，以及语言标准，如C99；
- POSIX函数，如`pthread`库；
- BSD函数，如BSD套接字（socket）；
- 对操作系统特定API的封装，如Linux的系统调用；

及时你的应用程序没有直接使用这些函数，但是有大量的库在使用这些函数，因此很可能你的库间接依赖到了glibc。

这里的问题是glibc在不同的Linux发行版本中是不兼容的！

例如我们在新的Ubuntu系统上构建的hello world程序在Centos 6上就会出错：

```sh
$ /hello
/hello: /lib64/libc.so.6: version `GLIBC_2.14' not found (required by /hello)
```

可以看到，两个Linux系统上glibc的版本是不同的。

还有其它一些C标准库的实现，例如面向嵌入式开发的[newlib](https://sourceware.org/newlib/)和[musl](https://www.musl-libc.org/)，也会有一样的挑战。

有几种针对该问题的解决方案：
- [LibcWrapGenerator](https://github.com/AppImage/AppImageKit/tree/stable/v1.0/LibcWrapGenerator)
- [glibc_version_header](https://github.com/wheybags/glibc_version_header)
- [bingcc](https://github.com/sulix/bingcc)

一些人建议使用glibc的静态库，但是强烈建议不要这样做。一个原因是新的glibc可能使用早期版本不存在的系统调用，如果你的应用程序依赖了新的glibc，可能在某些系统上出现随机的运行时失败，这种问题非常难以调试和定位。

可以通过在Conan包配置里面把不同glibc版本的Linux发行名称作为Conan的子settings（通过在settings.yml文件中定义），这和我们前面讲的通过`package_id()`和`build_id()`为包定义兼容性是一样的。这样不同glibc版本的linux将会获取不同的包或者触发自行从源码构建。具体为Conan增加子settings的方式见[文档](https://docs.conan.io/en/latest/extending/custom_settings.html#add-new-settings)。

#### C++标准库

一般默认的C++标准库是`libstdc++`，但是`libc++`和`stlport`也是常用的实现版本。

和标准C库glibc类似，应用程序和老系统上的libstdc++链接，也可能会发生错误：

```sh
$ /hello
/hello: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found (required by /hello)
/hello: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.26' not found (required by /hello)
```

幸运的是，对于C++我们可以简单的为编译增加`-static-libstdc++`的编译标记即可，这是因为C++标准库不会直接使用系统调用，往往是通过libc库帮助其封装。

#### 编译器运行时

除了C和C++运行时库，应用程序可能还会用编译器的运行时库。这些编译器运行时库为应用程序提供了一些低层次的函数，例如支持处理异常的编译器指令。编译器运行时库中的函数往往不会被代码直接调用，大多数时候是由编译器隐式的插入到代码中的。例如下面的可执行程序就依赖了libgcc_s.so。

```sh
$ ldd ./a.out
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f6626aee000)
```

可以使用`-static-libgcc`的编译期标记来对其静态链接而避免各种应用程序分发问题。另外，可以通过[GCC手册](https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html)尝试找到其它更具体的解决方案。

#### 系统调用（syscall）

Linux内核经常会提供新的系统调用，如果应用程序或者第三方库想要使用这些新的能力，有的时候可能会直接调用这些系统API（而非使用glibc的封装）。

上述行为的结果是，应用程序需要在对应的内核上编译，当其发布到其它老的内核系统上执行可能会运行失败。

所以建议尽量使用glibc，或者根据linux内核的版本为定义Conan定义新的settings和子settings，然后用其保证二进制兼容性。

## Creating conan packages to install dev tools
### Using the tool packages in other recipes

### Using the tool packages in your system

## Build requirements
### Declaring build requirements

### Properties of build requirements

### Testing libraries

### Common python code


