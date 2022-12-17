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
kubevirt 这一个项目能打出的镜像可太多了，这里还是以virt-handler作为例子：

考虑到篇幅，本文不会列举整个 BUILD.bazel 的内容，查看 virt-handler 完整的 BUILD.bazel 文件请[点击这里](https://github.com/kubevirt/kubevirt/blob/v0.50.0/cmd/virt-handler/BUILD.bazel)

`virt-handler-image`所依赖的基础镜像为`version-container`：
```
container_image(
    name = "version-container",
    directory = "/",
    files = [
        ":virt_launcher.cil",
        "//:get-version",
    ],
    tars = select({
        "@io_bazel_rules_go//go/platform:linux_arm64": [
            ":passwd-tar",
            ":nsswitch-tar",
            ":nftables-tar",
            "//rpm:handlerbase_aarch64",
        ],
        "//conditions:default": [
            ":passwd-tar",
            ":nsswitch-tar",
            ":nftables-tar",
            "//rpm:handlerbase_x86_64",
        ],
    }),
)
```
根据官方文档的描述，我的理解是：`directory`的内容是`files`和`tars`的内容放置或安装的位置，但是`tars`中的`xx-tar`会解压到`xx-tar`中指定的目录。所以`version-container`中的操作为拷贝`virt_launcher.cil`和项目根目录下`get-version`目标中的内容到`/`目录下，安装`passwd-tar`、`nsswitch-tar`、`nftables-tar`以及 rpm 目录下`handlerbase_某架构`目标的内容。

`passwd-tar`中的内容如下：
```
pkg_tar(
    name = "passwd-tar",
    srcs = [
        ":group",
        ":passwd",
    ],
    mode = "0644",
    package_dir = "etc",
    visibility = ["//visibility:public"],
)
```
我们可以进入容器中进行验证：
```
docker run -it --entrypoint /bin/bash bazel/cmd/virt-handler:virt-handler-image
```
`/`目录下：
```
.	    .version  dev   lib    mnt	 root  srv  usr
..	    bin       etc   lib64  opt	 run   sys  var
.dockerenv  boot      home  media  proc  sbin  tmp  virt_launcher.cil
```
‘/etc’目录下：
```
GREP_COLORS		gss	       nsswitch.conf   resolv.conf
X11			host.conf      opt	       rpc
adjtime			hostname       os-release      rpm
aliases			hosts	       pam.d	       security
alternatives		init.d	       passwd	       selinux
bash_completion.d	inputrc        pkcs11	       services
bashrc			iproute2       pki	       sestatus.conf
bindresvport.blacklist	issue	       pm	       shadow
centos-release		issue.net      popt.d	       shells
chkconfig.d		krb5.conf      printcap        skel
crypto-policies		krb5.conf.d    profile	       ssl
csh.cshrc		ld.so.conf     profile.d       subgid
csh.login		ld.so.conf.d   protocols       subuid
default			libaudit.conf  rc.d	       sysconfig
dnf			libibverbs.d   rc0.d	       system-release
environment		libnl	       rc1.d	       system-release-cpe
ethertypes		login.defs     rc2.d	       terminfo
exports			motd	       rc3.d	       virc
filesystems		mtab	       rc4.d	       xattr.conf
gcrypt			netconfig      rc5.d	       xdg
group			networks       rc6.d	       xinetd.d
gshadow			nftables       redhat-release  yum.repos.d
```
`group`以及`passwd`的内容如下：
```
bash-4.4# cat group 
qemu:x:107:
root:x:0:
bash-4.4# cat passwd 
qemu:x:107:107:user::/bin/bash
root:x:0:0:root:/root:/bin/bash
```
`passwd-tar`中创建了两个组`qemu`、`root`以及两个用户`qemu-user`、`root-user`，分别保存在`group`和`passwd`文件中，文件属性为`644`，解压的位置为 `/etc`下。其余的内容大致也是如此，rpm目录下`handlerbase`目标是安装必要的软件。至此，我们也能够完成`version-container`这部分的 dockerfile：
```
FROM base_image

COPY .version /

COPY virt_launcher.cil /

COPY nsswitch.conf /etc

RUN groupadd qemu -g 107 &&\
        useradd qemu -u 107 -g 107 &&\
        usermod -s /bin/bash qemu &&\
        mkdir -p /etc/nftables

COPY ipv4-nat.nft /etc/nftables
COPY ipv6-nat.nft /etc/nftables

RUN cd /etc &&\
        dnf update -y &&\
        dnf install -y bzip2 \
                diffutils \
                iptables \
                jansson \
                libaio \
                libbpf \
                libburn \
                libisoburn \
                libisofs \
                libselinux-utils \
                nftables \
                policycoreutils \
                qemu-img \
                rpm-plugin-selinux \
                selinux-policy \
                selinux-policy-targeted \
                xorriso \
        &&\
        cp /usr/sbin/iptables /usr/sbin/iptables-legacy &&\
        chmod 755 nftables &&\
        cd /
```
`virt-handler-image`除了基础镜像外都是“见名知意”的操作：
```
container_image(
    name = "virt-handler-image",
    architecture = select({
        "@io_bazel_rules_go//go/platform:linux_arm64": "arm64",
        "//conditions:default": "amd64",
    }),
    base = ":version-container",
    directory = "/usr/bin/",
    entrypoint = ["/usr/bin/virt-handler"],
    files = [
        ":virt-handler",
        ":container-disk",
        "//cmd/virt-chroot",
    ],
    visibility = ["//visibility:public"],
)
```
将`files`中的内容放在`directory`的位置，包括可执行文件`virt-handler`、`container-disk`以及 cmd 下`virt-chroot目标`构建的结果。`entrypoint`,`user`,`cmd`这类操作与 dockerfile 一致。
```
COPY virt-handler /usr/bin/
COPY virt-chroot /usr/bin/
COPY container-disk /usr/bin/

ENTRYPOINT ["/usr/bin/virt-handler"]
```
