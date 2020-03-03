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
$ conan create . user/channel --keep-build
```

## Packages in editable mode

### Put a package in editable mode

### Editable packages layouts

### Using a package in editable mode

### Revert the editable mode


## Workspaces

### Conan workspace definition

### Single configuration build environments

### Multi configuration build environments

### Out of source builds

### Notes


