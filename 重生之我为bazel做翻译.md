# 重生之我为 bazel 做翻译
## 背景
最近项目需要在自己的架构上移植 kubevirt 系列镜像，golang 相关的镜像相对 c/c++ 移植起来问题少很多，但 kubevirt 这一项目是通过 bazel 工具编译的，在我了解了 bazel 编译项目的方法后发现，目前我们的架构对 bazel 生态的支持有限，暂时无法使用 bazel，只能暂时通过 BUILD.bazel 以及 WORKSPACE 的内容来写 dockerfile,曲线救国构建镜像。
## 快速体验
按照[官方教程](https://bazel.build/start/cpp)构建了一个简单的 c++ 项目，算是知道怎么使用了，我个人认为 BUILD.bazel 写起来还是比较麻烦的，但构建起来是真方便。
## bazel 构建 kubevirt
### 环境
x86_64  
docker 容器  
ubuntu 20.04
### 下载项目
```
git clone https://github.com/kubevirt/kubevirt.git &&\
cd kubevirt &&\
git checkout v0.50.0
```
### 安装bazel
kubevirt 根目录下的`.bazelversion`文件中写了 bazel 的版本，版本不匹配使用 bazel 构建时会有 log 信息。官方下载教程点击[这里](https://bazel.build/install)。
### 构建前的准备
bazel 安装完成后运行 kubevirt 的引导脚本`hack/bootstrap.sh`，运行完成后下载 python3：
```
/kubevirt/hack/bootstarp.sh &&\
apt-get install python3
```
### 构建镜像
以 virt-handler 举例，在构建这个镜像时遇到了一个问题，目前只是间接解决了，也算是记录一下。

在 kubevirt 根目录下，运行：
```
bazel build //cmd/virt-handler:virt-handler-image.tar
```
简单解释一下，`//`为 WORKSAPCE 所在的目录，也是整个工程的根目录；`cmd/virt-handler`为 BUILD.bazel 所在的目录，称为软件包，工作区中包含一个 BUILD（部分项目用 BUILD，与 BUILD.bazel 一个意思） 文件的目录就是一个软件包；最后的`virt-handler-image`为构建目标，加上`.tar`构建完成的 tar 包可以通过下方的命令导入到 docker 中。
```
docker load -i virt-handler-image.tar
```
构建过程中遇到了一个问题：
```
root@ea3aed099109:/usr/local/src/kubevirt# bazel build //cmd/virt-handler:virt-handler-image.tar
INFO: Analyzed target //cmd/virt-handler:virt-handler-image.tar (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
ERROR: /usr/local/src/kubevirt/cmd/container-disk-v2alpha/BUILD.bazel:3:10: Compiling cmd/container-disk-v2alpha/main.c failed: undeclared inclusion(s) in rule '//cmd/container-disk-v2alpha:container-disk':
this rule is missing dependency declarations for the following files included by 'cmd/container-disk-v2alpha/main.c':
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stdarg.h'
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stddef.h'
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stdbool.h'
Target //cmd/virt-handler:virt-handler-image.tar failed to build
Use --verbose_failures to see the command lines of failed build steps.
ERROR: /usr/local/src/kubevirt/cmd/virt-handler/BUILD.bazel:173:16 JoinLayers cmd/virt-handler/virt-handler-image.tar failed: undeclared inclusion(s) in rule '//cmd/container-disk-v2alpha:container-disk':
this rule is missing dependency declarations for the following files included by 'cmd/container-disk-v2alpha/main.c':
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stdarg.h'
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stddef.h'
  '/usr/lib/gcc/x86_64-linux-gnu/9/include/stdbool.h'
INFO: Elapsed time: 8.012s, Critical Path: 7.66s
INFO: 199 processes: 16 internal, 183 processwrapper-sandbox.
FAILED: Build did NOT complete successfully
```
bazel 无法直接成功构建`container-disk`目标，我目前的解决方法是查看构建`container-disk`目标的 BUILD.bazel 文件:
```
load("@io_bazel_rules_go//go:def.bzl", "go_test")

cc_binary(
    name = "container-disk",
    srcs = ["main.c"],
    linkopts = [
        "-static",
    ],
    visibility = ["//visibility:public"],
)

go_test(
    name = "go_default_test",
    srcs = [
        "container_disk_v2alpha_suite_test.go",
        "main_test.go",
    ],
    args = [
        "--container-disk-binary",
        "$(location //cmd/container-disk-v2alpha:container-disk)",
    ],
    data = ["//cmd/container-disk-v2alpha:container-disk"],
    deps = [
        "//staging/src/kubevirt.io/client-go/testutils:go_default_library",
        "//vendor/github.com/onsi/ginkgo:go_default_library",
        "//vendor/github.com/onsi/gomega:go_default_library",
    ],
)
```
手动构建：
```
gcc -static -o container-disk main.c
```
将生成的可执行文件移动到构建 virt-handler-image 目标的 BUILD.bazel 文件所在的位置，修改 BUILD.bazel：
```
diff --git a/cmd/virt-handler/BUILD.bazel b/cmd/virt-handler/BUILD.bazel
index fa4fa0c5c..0830bad78 100644
--- a/cmd/virt-handler/BUILD.bazel
+++ b/cmd/virt-handler/BUILD.bazel
@@ -181,7 +181,7 @@ container_image(
     entrypoint = ["/usr/bin/virt-handler"],
     files = [
         ":virt-handler",
-        "//cmd/container-disk-v2alpha:container-disk",
+        ":container-disk",
         "//cmd/virt-chroot",
     ],
     visibility = ["//visibility:public"],
```
重新构建，成功：
```
root@ea3aed099109:/usr/local/src/kubevirt# bazel build //cmd/virt-handler:virt-handler-image.tar
INFO: Analyzed target //cmd/virt-handler:virt-handler-image.tar (1 packages loaded, 20 targets configured).
INFO: Found 1 target...
INFO: From Converting handlerbase_x86_64 to tar:
Flag shorthand -s has been deprecated, use --symlinks instead
INFO: From ImageLayer cmd/virt-handler/version-container-layer.tar:
Duplicate file in archive: ./etc/nsswitch.conf, picking first occurrence
Duplicate file in archive: ./etc/ethertypes, picking first occurrence
Duplicate file in archive: ./etc/group, picking first occurrence
Duplicate file in archive: ./etc/passwd, picking first occurrence
Target //cmd/virt-handler:virt-handler-image.tar up-to-date:
  bazel-bin/cmd/virt-handler/virt-handler-image.tar
INFO: Elapsed time: 40.873s, Critical Path: 37.17s
INFO: 405 processes: 3 internal, 402 processwrapper-sandbox.
INFO: Build completed successfully, 405 total actions
```
## 从 bazel 到 dockerfile
要按照 BUILD.bazel 以及 WORKSPACE 写 dockerfile，关键是要理解其参数的含义，多数参数都是“见名知意”，具体可查看[官方仓库](https://github.com/bazelbuild/rules_docker)中的文档。
### 举例
kubevirt 这一个项目能打出的镜像可太多了，这里记录一个例子：

...to be continued
