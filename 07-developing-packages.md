# 开发包

## 包开发流程

在先前的例子中，我们使用`conan create`命令创建了包，每次运行这条命令，Conan执行如下步骤：

- 将源码拷贝到一个新创建的干净的build目录下；
- 从源码构建完整的库；
- 当构建成功后，将库打包；
- 构建test_package样例并运行测试；

有的时候，我们构建的库很大，每次重复上述过程有些消耗和浪费。本节描述详细的包开发流程，深入的理解可以对该过程做优化。

### `conan source`

我们可以从`conan source`命令开始。这条命令会按照你的包配置下载源码文件，将其放到临时的子目录(`source-folder`参数指定)中。

```sh
$ cd example_conan_flow
$ conan source . --source-folder=tmp/source

PROJECT: Configuring sources in C:\Users\conan\example_conan_flow\tmp\source
Cloning into 'hello'...
```

### `conan install`

`conan install`用于激活所有和依赖相关的操作。 参数`install-folder`指定安装文件的生成目录。

```sh
$ conan install . --install-folder=tmp/build [--profile XXXX]

PROJECT: Installing C:\Users\conan\example_conan_flow\conanfile.py
Requirements
Packages
...
```

上面我们通过`conan install`在“tmp/build”子目录下生成了conaninfo.txt以及conanbuildinfo.cmake文件。

### `conan build`

build方法需要源码的路径以及install目录（获得依赖以及包配置文件），然后它就可以执行构建了。

```sh
$ conan build . --source-folder=tmp/source --build-folder=tmp/build

Project: Running build()
...
Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:03.34
```

这里如果我们忽略`--install-folder=tmp/build`，它将默认取`--build-folder`参数的值。

如果你只需要修改代码重新构建包，其实只用像这个例子这样调用`conan build`就可以了。

### `conan package`

`conan package`将会调用包配置文件中的`package()`函数。它需要所有的其它目录信息（源码中的头文件、install目录下的包依赖信息、构建的结果），以便能够执行打包。

```sh
$ conan package . --source-folder=tmp/source --build-folder=tmp/build --package-folder=tmp/package

PROJECT: Generating the package
PROJECT: Package folder C:\Users\conan\example_conan_flow\tmp\package
PROJECT: Calling package()
PROJECT package(): Copied 1 '.h' files: hello.h
PROJECT package(): Copied 2 '.lib' files: greet.lib, hello.lib
PROJECT: Package 'package' created
```

### `conan export-pkg`

当你检查你的包没有问题，这时你可以使用`conan export-pkg`将其导出到你的本地conan包缓存。这个命令需要的参数和`package()`一样，它会完整的重新执行一遍打包过程，以确认包的产生是可以重复的。

```sh
$ conan export-pkg . user/channel --source-folder=tmp/source --build-folder=tmp/build --profile=myprofile

Packaging to 6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Generating the package
Hello/1.1@user/channel: Package folder C:\Users\conan\.conan\data\Hello\1.1\user\channel\package\6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Calling package()
Hello/1.1@user/channel package(): Copied 2 '.lib' files: greet.lib, hello.lib
Hello/1.1@user/channel package(): Copied 2 '.lib' files: greet.lib, hello.lib
Hello/1.1@user/channel: Package '6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7' created
```

- 使用`source-folder`和`build-folder`将会调用`package()`函数从这些目录里抽取需要的文件在本地缓存里面创建包。这里不需要提前执行`conan package`。但是你在开发的时候可能需要`conan package`命令做中间的调试；

- 如果在上例中使用`package-folder`参数，那么将不会调用`package()`函数，它将假定包已经通过之前的`conan package`创建好了，并直接从提供的目录中进行拷贝生成包。

### `conan test`

最后一步就是测试包。

```sh
$ conan test test_package Hello/1.1@user/channel

Hello/1.1@user/channel (test package): Installing C:\Users\conan\repos\example_conan_flow\test_package\conanfile.py
Requirements
    Hello/1.1@user/channel from local
Packages
    Hello/1.1@user/channel:6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7

Hello/1.1@user/channel: Already installed!
Hello/1.1@user/channel (test package): Generator cmake created conanbuildinfo.cmake
Hello/1.1@user/channel (test package): Generator txt created conanbuildinfo.txt
Hello/1.1@user/channel (test package): Generated conaninfo.txt
Hello/1.1@user/channel (test package): Running build()
```

最后完整的过程如下：

```sh
$ git clone git@github.com:memsharded/example_conan_flow.git
$ cd example_conan_flow
$ conan source .
$ conan install . -pr=default
$ conan build .
$ conan package .
# So far, this is local. Now put the local binaries in cache
$ conan export-pkg . Hello/1.1@user/testing -pr=default
# And test it, to check it is working in the local cache
$ conan test test_package Hello/1.1@user/testing
...
Hello/1.1@user/testing (test package): Running test()
Hello World!
```

### `conan create`

现在我们知道了包配置的详细执行步骤。`conan create`是对上面命令的合并，除了`conan test`。

```sh
$ conan create . user/channel
```

即使使用这个命令，包创建者仍然可以在本地包缓存中迭代执行（当调试的时候），可以通过`--keep-source`和`--keep-build`参数。

如果在过程中看到`source()`方法执行成功了，但是包的构建执行失败了，则再次执行可以使用`--keep-source`。

```sh
$ conan create . user/channel --keep-source

Hello/1.1@user/channel: A new conanfile.py version was exported
Hello/1.1@user/channel: Folder: C:\Users\conan\.conan\data\Hello\1.1\user\channel\export
Hello/1.1@user/channel (test package): Installing C:\Users\conan\repos\example_conan_flow\test_package\conanfile.py
Requirements
    Hello/1.1@user/channel from local
Packages
    Hello/1.1@user/channel:6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7

Hello/1.1@user/channel: WARN: Forced build from source
Hello/1.1@user/channel: Building your package in C:\Users\conan\.conan\data\Hello\1.1\user\channel\build\6cc50b139b9c3d27b3e9042d5f5372d327b3a9f7
Hello/1.1@user/channel: Configuring sources in C:\Users\conan\.conan\data\Hello\1.1\user\channel\source
Cloning into 'hello'...
remote: Counting objects: 17, done.
remote: Total 17 (delta 0), reused 0 (delta 0), pack-reused 17
Unpacking objects: 100% (17/17), done.
Switched to a new branch 'static_shared'
Branch 'static_shared' set up to track remote branch 'static_shared' from 'origin'.
Hello/1.1@user/channel: Copying sources to build folder
Hello/1.1@user/channel: Generator cmake created conanbuildinfo.cmake
Hello/1.1@user/channel: Calling build()
```

如果你看到构建成功执行了，则可以使用`--keep-build`来跳过`build()`函数的执行。

```sh
$ conan create . user/channel --keep-build`
```

## 可编辑模式(editable mode)的包

注意：这是个实验特性，未来发布的版本可能会发生不兼容的修改。

当在有多个内在功能互相关联的大型项目中工作时，建议不要将工程组织成一个大的单体项目。推荐将项目分解成多个库，每个库专门完成一组内聚的任务，甚至分配给专门的开发团队维护。这种方式有助于隔离和重用代码，缩短编译时间，并减少include了错误头文件的可能性。

然而，在某些情况下，同时工作在多个库上，能够及时看到一个修改后对其它的影响，这点也是很有用的。我们前面介绍的工作流：`conan source`,`conan install`, `conan build`, `conan package`, 以及`conan export-pkg`，会将包构建后发布到本地缓存，为使用者做好准备。

但是，使用可编辑模式包，你能够告诉Conan从本地工作目录下查找包的头文件以及构建产物，而无需将包导出。

我们用一个例子来看看这个特性。假设开发者创建了一个`CoolApp`的应用，紧密依赖了一个`cpp/version@user/dev`的库。包`cpp/version@user/dev`已经可以工作了，开发者在本地目录下开发了代码，他们随时可以构建并执行`conan create . cool/version@user/dev`来创建这个包。同时，`CoolApp`里面有一个conanfile.txt(或者conanfile.py)，描述了它依赖于`cpp/version@user/dev`。当构建这个程序的时候，它将从Conan的本地缓存中查找依赖的`cool`包。

### 将包设置为可编辑模式

避免每次修改`cpp/version@user/dev`都需要在Conan缓存中创建包，我们可以将包设置为可修改模式：通过创建一个从Conan缓存到本地工作目录的引用连接。

```sh
$ conan editable add <path/to/local/dev/libcool> cool/version@user/dev
# you could do "cd <path/to/local/dev/libcool> && conan editable add . cool/version@user/dev"
```

执行上述命令后，本地的包或者项目每次使用`cpp/version@user/dev`，都将会重镜像到`<path/to/local/dev/libcool>`目录下，不会再从Conan缓存中去获取。

Conan的包配置文件通过`package_info()`函数定义了包的布局（layout）。如果不做修改，默认的布局如下述代码所示：

```python
def package_info(self):
    # default behavior, doesn't need to be explicitly defined in recipes
    self.cpp_info.includedirs = ["include"]
    self.cpp_info.libdirs = ["lib"]
    self.cpp_info.bindirs = ["bin"]
    self.cpp_info.resdirs = ["res"]
```

这意味着conan将会使用路径`path/to/local/dev/libcool/include`查找`cool`的头文件，通过`path/to/local/dev/libcool/lib`查找对应的库文件。

但是，实际情况是，大部分包在开发过程中经常做增量构建，构建目录的布局和包最终发布的布局不会完全一样。当然每次增量构建后可以执行`conan package`进行打包，使得布局满足包布局要求，但是这样不够优雅。Conan提供了几种为可编辑模式下的包自定义包布局的方式。

### 可编辑模式的包布局

可编辑模式的包的布局有几种不同的自定义方式：

- 通过包配置文件定义

通过包配置文件中的`package_info()`函数进行定义：

```python
from conans import ConanFile

class Pkg(ConanFile):
    settings = "build_type"
    def package_info(self):
        if not self.in_local_cache:
            d = "include_%s" % self.settings.build_type
            self.cpp_info.includedirs = [d.lower()]
```

上述代码通过构建类型来配置头文件的目录布局。如果`build_type=Debug`，则include目录是`path/to/local/dev/libcool/include_debug`，否则是`path/to/local/dev/libcool/include_release`。同样，其它的目录（libdirs，bindirs等）都可以如此自定义。

- 通过布局文件（layout files）配置

除了通过包配置文件自定义布局，还可以使用独立的布局文件。这对于有很多的库共享相同的布局的时候很有用。

布局文件是一些ini文件，但是Conan做了扩展，可以使用[Jinja2](https://palletsprojects.com/p/jinja/)模板引擎。在配置文件里面使用`settings`、`options`以及当前的`reference`对象，可以为布局配置文件增加逻辑：

```ini
[includedirs]
src/core/include
src/cmp_a/include

[libdirs]
build/{{settings.build_type}}/{{settings.arch}}

[bindirs]
{% if options.shared %}
build/{{settings.build_type}}/shared
{% else %}
build/{{settings.build_type}}/static
{% endif %}

[resdirs]
{% for item in ["cmp1", "cmp2", "cmp3"] %}
src/{{ item }}/resouces/{% if item != "cmp3" %}{{ settings.arch }}{% endif %}
{% endfor %}
```

Jinja的语法可以查看它的[官方文档](https://palletsprojects.com/p/jinja/)。

布局描述文件也可以对指定的包做配置，下面例子中设置`cool`包的include目录为`src/core/include`，而其它的则是`src/include`。

```ini
[includedirs]
src/include

[cool/version@user/dev:includedirs]
src/core/include
```

可编辑模式下的包的客赔目录有`includedirs`, `libdirs`, `bindirs`, `resdirs`, 以及`builddirs`。所有这些都在包描述文件的`cpp_info`的字典字段中，而该字典中其它值都是不能修改的，例如`cflags`、`defines`等。

默认所有的路径都是相对于conanfile.py的相对路径，也可以用绝对路径。

为包指定布局文件使用`conan editable add`命令，例如：

```sh
$ conan editable add . cool/version@user/dev --layout=win_layout
```

`win_layout`文件首先会在当前文件夹下查找，所以可将将其和源码放到一起，一起发布和被共享使用。如果在当前目录下没有找到，则会在本地缓存`~/.conan/layouts`目录下查找。可以定义布局文件共享给团队，然后通过`conan config install`进行安装。

如果`conan editable add`命令没有参数，将会默认使用`~/.conan/layouts/default`文件中描述的布局。

对于布局，Conan会按照以下优先级选择：

- 首先会执行包配置文件中的`package_info()`函数。这会定义各种标记参数（例如`cflags`）、definitions（例如`-D`的宏参数）以及布局目录: `includedirs`, `libdirs`等等；

- 如果布局文件存在，或者显示的应用了默认的`.conan/layouts/default`文件，Conan将会查找对应的匹配布局文件；

- 如果找到匹配的布局文件，则会使用布局文件中的定义替换(includedirs, libdirs, resdirs, builddirs, bindirs)；

- 布局文件的匹配，按照从特殊到一般的顺序 （布局文件可以为指定的包指定特殊的目录布局）；

- 如果没有匹配的，则仍旧使用原来定义在`package_info()`中的目录结构；

- 如果在`conan editable add`之后，手动增加了`.conan/layouts/default`文件，它将不会被使用；

### 使用可编辑模式的包

一旦对一个包建立了可编辑模式的引用，它就会对整个系统生效：所有本机（使用相同的conan缓存）的对其有依赖的工程或者包，都会重镜像到可编辑模式包的真实工作目录。

一个可编辑模式的包，对它的用户来说是透明的。使用可编辑模式的包的工作流如下：

- 通过`git/svn clone ... && cd folder`获得`cool/version@user/dev`的源码；

- 将包设置为可编辑模式：`conan editable add . cool/version@user/dev --layout=mylayout`;

- 现在可以正常编码和构建。注意你本地工作目录下的'cool'包的目录结构布局要和前面命令中的`mylayout`文件制定的一样；

- 对于消费`cool`的工程`CoolApp`，和以前一样，使用`conan install`并执行构建；

- 回到`cool/version@user/dev`的源码目录，修改代码并构建，这里不需要执行任何Conan命令以及打包；

- 回到`CoolApp`目录，重新构建它，测试`cool`的修改对其的影响；

可以看到，可以同时开发`cool`和`CoolApp`，不用反复打包发包。

注意：当一个包处于可编辑模式，大多数的工作也不会工作。可编辑模式的包不能使用`conan upload`, `conan export`或者`conan create`命令。

### 取消包的可编辑模式

取消包的可编辑模式，使用如下命令：

```sh
$ conan editable remove cool/version@user/dev
```

当执行上述命令后，所有对包的消费将会重新需要从本地包缓存中获取。

## 工作区（Workspaces）

### Conan workspace definition

### Single configuration build environments

### Multi configuration build environments

### Out of source builds

### Notes


