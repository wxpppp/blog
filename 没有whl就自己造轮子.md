# 没有whl就自己造轮子
## 背景
在做 python 相关的镜像时常常需要通过 pip 装一些 whl 包，一些架构相关的包可能无法直接安装成功，这时候就需要找到源码，在本地进行适配后打成 whl 包。
## 从源码到whl包
适配的源码各不相同，但打包的步骤几乎是一致的，本文通过打包`thefuck`记录打包过程。

首先安装 python3 python3-pip:
```
yum install -y python3 python3-pip
```
pip3 安装 wheel 包:
```
pip3 install wheel
```
下载`thefuck`源码，切换到一个稳定的版本：
```
git clone https://github.com/nvbn/thefuck.git &&\
cd thefuck &&\
git checkout 3.32
```
如果该项目无架构相关代码，直接运行：
```
python3 setup.py bdist_wheel
```
即可在`dist/`目录下得到生成的`.whl`文件
```
[root@41c1d480239f dist]# ls
thefuck-3.32-py2.py3-none-any.whl
```
如果有架构相关的代码且该代码无法屏蔽，则需要先对源码进行适配。

得到的`.whl`文件可直接安装：
```
pip3 install thefuck-3.32-py2.py3-none-any.whl
```
安装中如果出现的错误，需要根据报错信息具体分析。
