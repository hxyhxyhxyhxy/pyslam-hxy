# docker安装方法
- linux系统下的安装方法

```
# 1. 更新索引并安装依赖
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y

# 2. 添加 Docker 官方 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 3. 设置稳定版仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. 安装 Docker Engine
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

# 5. 免 sudo 使用 Docker (非常建议)
sudo usermod -aG docker $USER
# 注销并重新登录后生效
```

- 若要是使用宿主机的GPU则需要安装如下依赖
```
# 官方安装脚本
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
# ... (根据 NVIDIA 官网文档配置仓库并安装)
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

# docker使用方法
1. docker pull ---->   <镜像名>:<标签>从仓库拉取镜像
2. docker images---->  查看本地所有镜像
3. docker run -it --name my_container <镜像ID> 
   1. docker run :运行容器
   2. -i：使用具体哪一个镜像启动容器
   3. -t：使用什么交互终端 eg：bash
   4. --name：给容器起一个名字,名字是my_container
4. docker ps -a ---->     查看包括已停止的所有容器
5. docker stop/start <容器ID> ---->  管理容器状态启动或停止
6. docker exec -it <容器ID> /bin/bash ----> 进入正在运行的容器后台
7. docker rmi <镜像> / docker rm <容器> ----> 删除镜像或者容器
8. docker ps  ---->  查看所有正在运行的容器
9. docker ps -a ----> 查看所有容器（包括已经停止的）
10. docker inspect <容器名或ID> ---->  查看容器详细信息
11. docker status ----> 查看容器资源使用情况（cpu、内存）
12. docker logs <容器名或ID> ----> 查看容器日志
13. docker top <容器名或ID> ---->  查看容器进程
14. docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"----> 查看运行中容器的详细信息（格式化)
15. docker system df---->查看 Docker 占用的磁盘空间
16. docker builder prune -f ----> 清理构建缓存（回收 42.96GB）
17. docker container prune -f ----> 清理已停止的容器
18. docker system prune -a --volumes -f ---->  一键清理所有未使用资源
19. docker system prune -a --volumes----> 清理所有未使用的镜像、容器、网络和构建缓存
20. docker image prune ----> 清理悬空镜像
21. docker container prune ----> 清理未使用的容器


# 或者更安全地只和未使用的容器

docker container prune




# docker中环境配置
- **注意** 一下操作都是进入docker中的操作或者构建容器时的操作

## pip源更换
### 方法一：进入容器中运行一下命令
```
mkdir -p ~/.pip
cat > ~/.pip/pip.conf << 'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
```
### 方法二：临时使用镜像源
```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple <package-name>
```
### 方法三：构建镜像时添加镜像源（修改Dockerfile）
```
RUN mkdir -p ${home}/.pip && \
    echo "[global]" > ${home}/.pip/pip.conf && \
    echo "index-url = https://pypi.tuna.tsinghua.edu.cn/simple" >> ${home}/.pip/pip.conf && \
    echo "trusted-host = pypi.tuna.tsinghua.edu.cn" >> ${home}/.pip/pip.conf && \
    chown -R ${user}:${user} ${home}/.pip
```
笔者自己使用的是方法一

## apt换源
### 方法一：使用脚本一键换源（推荐）
```
# 备份原源文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 使用清华源（Ubuntu 20.04 focal）
sudo tee /etc/apt/sources.list > /dev/null << 'EOF'
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse
EOF

# 更新软件包列表
sudo apt update
```

### 方法二：使用阿里源
```
sudo tee /etc/apt/sources.list > /dev/null << 'EOF'
deb https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
EOF

sudo apt update
```
### 方法三：修改Dockerfile（永久方案）
```
# 在第 20 行之前添加（在RUN apt-get -y update之前添加）
RUN sed -i 's|archive.ubuntu.com|mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list && \
    sed -i 's|security.ubuntu.com|mirrors.tuna.tsinghua.edu.cn|g' /etc/apt/sources.list

RUN apt-get -y update
```

## 配置代理
### 方法一：在容器终端中配置网络代理（仅仅一次有效）
```
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```
**注意**：这里的端口根据自己的实际情况更改代理端口，clash的默认端口是7890.

### 方法二：使用--build-arg 构建参数（推荐）

**dockerfile中增添项**
```
# 在 Arguments 部分添加
ARG http_proxy
ARG https_proxy
ARG no_proxy
# 为当前 shell 会话设置环境变量（供后续命令使用）
ENV http_proxy=$http_proxy \
    https_proxy=$https_proxy \
    no_proxy=$no_proxy
```
**build** 
```
docker build \
    --build-arg http_proxy=http://127.0.0.1:7890 \
    --build-arg https_proxy=http://127.0.0.1:7890 \
    --build-arg no_proxy=localhost,127.0.0.1 \
    -t pyslam:cuda \
    -f Dockerfile_pyslam_cuda .

```

### 方法三：环境变量（推荐）
**启动容器时传递代理端口**
```
docker run -it --rm \
  --name pyslam-dev \
  --gpus all \
  --network host \
  -e http_proxy=http://127.0.0.1:7890 \
  -e https_proxy=http://127.0.0.1:7890 \
  -e no_proxy=localhost,127.0.0.1 \
  -e HTTP_PROXY=http://127.0.0.1:7890 \
  -e HTTPS_PROXY=http://127.0.0.1:7890 \
  -e NO_PROXY=localhost,127.0.0.1 \
  -v /home/hxy/doctor/slam/pyslam/pyslam:/home/hxy/pyslam \
  pyslam:cuda bash
```
**容器内验证**
```
# 检查代理环境变量
echo $http_proxy
echo $https_proxy

# 测试网络
curl -I https://www.google.com
```

### 方法四：使用宿主机网关ip(更可靠)
​
如果 127.0.0.1 不工作，使用宿主机网关 IP：

# 获取宿主机 IP（在容器内或外都可以）
```
ip addr show docker0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1
# 通常输出：172.17.0.1
```
启动命令改为：
```
docker run -it --rm \
  --name pyslam-dev \
  --gpus all \
  --network host \
  -e http_proxy=http://172.17.0.1:7890 \
  -e https_proxy=http://172.17.0.1:7890 \
  -e no_proxy=localhost,127.0.0.1 \
  -v /home/hxy/doctor/slam/pyslam/pyslam:/home/hxy/pyslam \
  pyslam:cuda bash
```
### 方法五：配置apt和git代理网络
**配置原因** ： 编译项目源码时，可能需要下载apt服务或者从github中clone再进行编译，此时不配置代理拉取代码可能会很慢

如果需要为特定工具配置网络（在容器内执行）：
```
# apt 代理（容器内执行）
sudo mkdir -p /etc/apt
echo 'Acquire::http::Proxy "http://172.17.0.1:7890";' | sudo tee /etc/apt/apt.conf.d/proxy.conf

# git 代理
git config --global http.proxy http://172.17.0.1:7890
git config --global https.proxy http://172.17.0.1:7890
```
# dockerfile实例分析
## dockerfile构建
```
# from https://hub.docker.com/r/nvidia/cuda/tags?page=1&name=ubuntu20
FROM nvidia/cuda:12.1.1-devel-ubuntu20.04 AS base
# Arguments
ARG user
ARG uid
ARG home
ARG workspace
ARG shell
ARG nvidia_driver_version
ARG container_name
ARG timezone=Etc/UTC

LABEL maintainer="luigifreda@gmail.com"

ENV TZ=${timezone}

################################

RUN apt-get -y update 
# && apt-get -y dist-upgrade 
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata apt-utils keyboard-configuration
RUN apt-get install -y git curl wget ca-certificates --no-install-recommends

# Basic Utilities 
RUN apt-get install -y screen tree sudo ssh synaptic aptitude gedit geany --no-install-recommends

# Latest X11 / mesa GL
RUN apt-get install -y mesa-utils --no-install-recommends

# Dependencies required to build rviz
RUN apt-get install -y libqt5core5a libqt5dbus5 libqt5gui5 --no-install-recommends

# Additional development tools
RUN apt-get install -y cmake build-essential git --no-install-recommends

# Python 
RUN apt-get install -y python3-pip
RUN apt-get install -y python-is-python3  

# ROS deps
RUN apt-get install -y doxygen libssh2-1-dev libudev-dev --no-install-recommends 

# pyslam stuff 
RUN apt-get install -y rsync python3-sdl2 python3-tk \
    libprotobuf-dev libeigen3-dev libopencv-dev libsuitesparse-dev libglew-dev --no-install-recommends
RUN apt-get install -y libhdf5-dev --no-install-recommends   # needed when building h5py wheel from src is required (arm64)

################################

# from  http://wiki.ros.org/docker/Tutorials/Hardware%20Acceleration  (with nvidia-docker2)
# nvidia-container-runtime
ENV NVIDIA_VISIBLE_DEVICES \
    ${NVIDIA_VISIBLE_DEVICES:-all}
ENV NVIDIA_DRIVER_CAPABILITIES \
    ${NVIDIA_DRIVER_CAPABILITIES:+$NVIDIA_DRIVER_CAPABILITIES,}graphics

# Make SSH available
EXPOSE 22

# Mount the user's home directory
VOLUME "${home}"

RUN useradd ${user} -m -s ${shell} -u ${uid} -G sudo,dialout

# Clone user into docker image and set up X11 sharing 
RUN \
    echo "${user}:x:${uid}:${uid}:${user},,,:${home}:${shell}" >> /etc/passwd && \
    echo "${user}:x:${uid}:" >> /etc/group && \
    echo "${user} ALL=(ALL) NOPASSWD: ALL" > "/etc/sudoers.d/${user}" && \
    chmod 0440 "/etc/sudoers.d/${user}"

#RUN echo "${user} ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers 
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

RUN echo "${user}:docker" | chpasswd 
# this allows to switch to user with $ su -
RUN echo "root:docker" | chpasswd  

# Switch to user
USER "${user}"

# This is required for sharing Xauthority
ENV QT_X11_NO_MITSHM=1

ENV CONTAINER_NAME="${container_name}"

# Switch to the workspace
WORKDIR ${workspace}

RUN sudo chown -R ${user}:${user} ${home} && sudo chmod -R u+rwx ${home}
COPY ./bashrc_add ${home}/bashrc_add
RUN sudo touch ${home}/.bashrc && sudo chown ${user}:${user} ${home}/.bashrc && sudo chmod 644 ${home}/.bashrc
RUN cat ${home}/bashrc_add >> "${home}/.bashrc"

################################
FROM base AS final

# Install and configure git
RUN sudo mkdir -p /home/${user}/.ssh && sudo chmod 0777 -R /home/${user}
#RUN touch ~/.gitconfig && touch ~/.ssh/known_hosts
RUN sudo touch ${home}/.gitconfig && sudo chown ${user}:${user} ${home}/.gitconfig && sudo chmod 644 ${home}/.gitconfig
RUN sudo touch ${home}/.ssh/known_hosts && sudo chown ${user}:${user} ${home}/.ssh/known_hosts && sudo chmod 644 ${home}/.ssh/known_hosts
#RUN ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts
RUN git config --global user.name "${user}"
RUN git config --global user.email "${user}@${user}.com"
RUN git config --global http.proxy ""
RUN sudo git config --system --add safe.directory '*' # to avoid "detected dubious ownership" error

# Install pyslam - Code will be mounted as volume at runtime
WORKDIR ${home}
# the following to inform we are inside docker at build time
RUN sudo touch /.dockerenv
ENV USER=${user}
# Note: Code is NOT copied here. Mount local directory with:
# docker run -v /path/to/pyslam:/home/docker/pyslam ...

# Uncomment below lines for production build with code embedded:
# RUN echo "Copying pyslam repository to ${home} (with CUDA support)"
# COPY --chown=${user}:${user} . ${home}/pyslam
# WORKDIR ${home}/pyslam
# RUN ./install_all.sh
# RUN /bin/bash -c "echo 'source pyslam/pyenv-activate.sh' >> ~/.bashrc"


# Create an entrypoint that dynamically sets XDG_RUNTIME_DIR
RUN sudo touch /usr/local/bin/entrypoint.sh && sudo chmod 0777 /usr/local/bin/entrypoint.sh
RUN printf '#!/bin/bash\n\
    export XDG_RUNTIME_DIR=/tmp/runtime-$(id -u)\n\
    mkdir -p "$XDG_RUNTIME_DIR"\n\
    chmod 700 "$XDG_RUNTIME_DIR"\n\
    exec "$@"\n' > /usr/local/bin/entrypoint.sh && \
    sudo chmod +x /usr/local/bin/entrypoint.sh

# Set the entrypoint
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["bash"]
```
### dockerfile详细讲解
#### 整体结构
```
FROM nvidia/cuda:12.1.1-devel-ubuntu20.04 AS base    ← 第一阶段：base
...
FROM base AS final                                   ← 第二阶段：final

```
这是一个**多阶段构建**，分为两个阶段：

- base: 基础环境配置
- final: 最终镜像，安装应用
​
*** 
#### 第一阶段：base基础环境
##### 1.基础镜像和参数
```
FROM nvidia/cuda:12.1.1-devel-ubuntu20.04 AS base

ARG user          # 用户名（运行时传入）
ARG uid           # 用户 UID（如 1000）
ARG home          # 家目录（如 /home/hxy）
ARG workspace     # 工作目录（如 /home/hxy/pyslam）
ARG shell         # Shell 类型
ARG nvidia_driver_version  # NVIDIA 驱动版本
ARG container_name         # 容器名
ARG timezone=Etc/UTC       # 时区

```

##### 2.安装系统依赖
```
# 更新包管理器
RUN apt-get -y update

# 安装时区工具
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata apt-utils keyboard-configuration

# 基础工具
RUN apt-get install -y git curl wget ca-certificates
RUN apt-get install -y screen tree sudo ssh synaptic aptitude gedit geany

# 图形相关（X11, OpenGL, Qt5）
RUN apt-get install -y mesa-utils
RUN apt-get install -y libqt5core5a libqt5dbus5 libqt5gui5

# 开发工具
RUN apt-get install -y cmake build-essential git

# Python
RUN apt-get install -y python3-pip
RUN apt-get install -y python-is-python3   # python → python3

# ROS 和 SLAM 依赖
RUN apt-get install -y doxygen libssh2-1-dev libudev-dev
RUN apt-get install -y rsync python3-sdl2 python3-tk \
    libprotobuf-dev libeigen3-dev libopencv-dev \
    libsuitesparse-dev libglew-dev libhdf5-dev

```

##### 3.NVIDIA GPU配置
```
ENV NVIDIA_VISIBLE_DEVICES ${NVIDIA_VISIBLE_DEVICES:-all}
ENV NVIDIA_DRIVER_CAPABILITIES ${NVIDIA_DRIVER_CAPABILITIES:+$NVIDIA_DRIVER_CAPABILITIES,}graphics
```
- all: 使用所有 GPU
- graphics: 启用图形渲染（用于 OpenGL GUI）


##### 4.创建用户
```
RUN useradd ${user} -m -s ${shell} -u ${uid} -G sudo,dialout
```
- 创建用户并加入sudo组（免密执行sudo）
##### 5.配置SSH和密码
```
EXPOSE 22                                  # 开放 SSH 端口
RUN echo "${user}:docker" | chpasswd       # 设置密码为 docker
RUN echo "root:docker" | chpasswd

```
##### 6.切换到普通用户
```
USER "${user}"
ENV QT_X11_NO_MITSHM=1                    # Qt X11 共享设置
WORKDIR ${workspace}                       # 切换工作目录
```

##### 7.配置bashrc
```
COPY ./bashrc_add ${home}/bashrc_add       # 复制本地文件到容器
RUN cat ${home}/bashrc_add >> "${home}/.bashrc"  # 追加到 .bashrc
```
#### 第二阶段：final安装应用
##### 1.配置git
```
RUN git config --global user.name "${user}"
RUN git config --global user.email "${user}@${user}.com"
RUN sudo git config --system --add safe.directory '*'
```
##### 2.安装pyslam
```
WORKDIR ${home}
RUN sudo touch /.dockerenv                 # 标记在 Docker 中运行
RUN git clone --recursive --progress https://github.com/luigifreda/pyslam.git
RUN cd pyslam && ./install_all.sh         # 执行安装脚本（耗时最长）
RUN echo 'source pyslam/pyenv-activate.sh' >> ~/.bashrc
```
##### 3.入口脚本
```
RUN printf '#!/bin/bash\n\
export XDG_RUNTIME_DIR=/tmp/runtime-$(id -u)\n\
mkdir -p "$XDG_RUNTIME_DIR"\n\
chmod 700 "$XDG_RUNTIME_DIR"\n\
exec "$@"\n' > /usr/local/bin/entrypoint.sh
```
这个脚本设置运行时目录(用于GUI应用)

##### 4.启动配置
```
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]   # 容器启动时执行
CMD ["bash"]                                  # 默认运行 bash
```
##### 构建流程图
```
nvidia/cuda:12.1.1-devel-ubuntu20.04
         ↓
    安装系统依赖
         ↓
    创建用户、配置环境
         ↓
   【第一阶段完成: base】
         ↓
    克隆 pyslam 仓库
         ↓
    执行 install_all.sh
         ↓
  【第二阶段完成: final】
         ↓
    生成镜像 pyslam:cuda
```
*** 
##### 关键特性总结
| 特性        | 作用           |
| ----------- | -------------- |
| CUDA 12.1.1 | GPU支持        |
| 多阶段构建  | 优化镜像大小   |
| 非root用户  | 安全性         |
| SSH免密登陆 | 方便远程连接   |
| X11/GUI支持 | 可运行图形程序 |
| 完整pyslam  | 开箱即用       |


## 容器构建命令：
```
docker build \
  -f Dockerfile_pyslam_cuda \
  --build-arg user=hxy \
  --build-arg uid=1000 \
  --build-arg home=/home/hxy \
  --build-arg workspace=/home/hxy/pyslam \
  --build-arg shell=/bin/bash \
  --build-arg container_name=pyslam_cuda \
  --build-arg timezone=Asia/Shanghai \
  -t pyslam:cuda \
  .
```
### 命令分析
- user=hxy - 容器内用户名
- uid=1000 - 与你本机 UID 一致（保证文件权限正确）
- home=/home/hxy - 容器内用户家目录
- workspace=/home/hxy/pyslam - 工作目录
- shell=/bin/bash - shell 类型
- container_name=pyslam_cuda - 容器名
- timezone=Asia/Shanghai - 时区设置

**如果需要使用网络**，添加这些参数：

```
--build-arg http_proxy=http://your-proxy:port \
--build-arg https_proxy=http://your-proxy:port \
--build-arg no_proxy=localhost,127.0.0.1
```
换成自己的代理端口，一般为127.0.0.1

## 启动容器命令
```
docker run -it --rm \
  --name pyslam-dev \
  --gpus all \
  --network host \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /home/hxy/doctor/slam/pyslam/pyslam:/home/hxy/pyslam \
  pyslam:cuda bash
```
### 参数说明

- --gpus all - 启用 NVIDIA GPU 支持
- --network host - 使用宿主机网络
- -e DISPLAY=$DISPLAY - X11 转发（支持 GUI）
- -v /tmp/.X11-unix:/tmp/.X11-unix - X11 socket 转发
- -v /home/hxy/doctor/slam/pyslam/pyslam:/home/hxy/pyslam - 挂载本地代码目录

### 基础参数
| 参数              | 含义                                                          |
| ----------------- | ------------------------------------------------------------- |
| -it               | -i (交互式) + -t (伪终端)，让你能像操作普通终端一样与容器交互 |
| --rm              | 容器退出后**自动删除**，适合临时开发环境（不会留下僵尸容器）  |
| --name pyslam-dev | 给容器命名，方便后续引用（如 docker exec -it pyslam-dev ...） |
| pyslam:cuda       | 要运行的**镜像名称**                                          |
| bash              | 容器启动后执行的**命令**（进入 bash shell）                   |


### GPU和代理
| 参数           | 含义                                                                |
| -------------- | ------------------------------------------------------------------- |
| --gpus all     | 将所有 NVIDIA GPU 传递给容器（需要 nvidia-docker 支持）             |
| --network host | 容器使用宿主机网络栈，容器与宿主机共享网络 IP（方便访问宿主机服务） |

**注意：这里有一个端口映射问题需要理解**

***容器中的端口和宿主机的端口直接就是一个端口吗？***

#### 模式1：--network host
**端口是同一个！**
```
docker run --network host ...
```
在这种模式下：
- ✅ 容器和宿主机共享网络栈
- ✅ 容器端口 7890 = 宿主机端口 7890
- ✅ 不需要端口映射
- ✅ 性能最好（无 NAT 转换开销）

示例：
```
# 宿主机运行代理（监听 7890）
clash -d ~/.config/clash

# 容器内直接访问宿主机端口
curl -x http://127.0.0.1:7890 https://www.google.com
```

#### 模式2：默认bridge模式（独立网络）
**端口是独立的，需要映射** 
```
docker run -p 7890:7890 ...  # 映射容器端口到宿主机
```
在这种模式下：
- ❌ 容器有独立的 IP 和网络命名空间
- ❌ 容器端口 7890 ≠ 宿主机端口 7890（除非映射）
- ⚠️ 需要 -p 参数才能从宿主机访问

示例：
```
# 将容器的 8080 端口映射到宿主机的 8888
docker run -p 8888:8080 ...

# 宿主机访问 localhost:8888 = 容器的 8080
```

### 环境变量和挂载
| 参数                                     | 含义                                                                |
| ---------------------------------------- | ------------------------------------------------------------------- |
| -e DISPLAY=$DISPLAY                      | 设置 DISPLAY 环境变量，让容器内的 GUI 程序显示在你的屏幕上          |
| -v /tmp/.X11-unix:/tmp/.X11-unix         | 挂载 X11 socket，实现图形界面转发（支持 rviz、OpenGL 等可视化工具） |
| -v /home/hxy/.../pyslam:/home/hxy/pyslam | **挂载本地代码目录**到容器，实现代码实时同步                        |
注意：这里可以添加很多个挂载，将宿主机目录映射到容器中

#docker使用建议
1. 先进行容器的构建build
2. 再将本地目录映射到容器中
3. 配置各种apt、pip源以及网络配置
4. 编译运行代码


# vscode配置
安装`Dev Containers `插件 
否则需要再终端中执行如下命令进入容器：
1. 最常用的方式：`docker exec (推荐)`
这是最标准的方法。它会在运行中的容器里开启一个新的终端进程。即使你执行`exit`退出，容器本身也会保持运行。
- -it 结合使用可以让你获得一个交互式的终端
- /bin/bash 是容器内常用的 shell，有些精简镜像（如 Alpine）需换成 /bin/sh


`docker exec -it <容器名称或ID> /bin/bash`

```
docker run -it \
  --name pyslam_x11 \
  --net=host \
  --gpus all \
  --ipc=host \
  --shm-size=16g \
  --privileged \
  -u $(id -u):$(id -g) \
  -e SHELL=/bin/bash \
  -e USER=hxy \
   -e http_proxy=http://127.0.0.1:7890 \
  -e https_proxy=http://127.0.0.1:7890 \
  -e no_proxy=localhost,127.0.0.1,.local \
  -e CONTAINER_NAME=pyslam_cuda \
  -e DISPLAY="$DISPLAY" \
  -e DOCKER=1 \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v /tmp/.docker.xauth:/tmp/.docker.xauth:rw \
  -v /home/hxy/doctor/pyslam:/home/hxy/pyslam \
  -v /media:/media \
  -v /mnt:/mnt \
  -v /dev:/dev \
  -v /home/hxy/.ssh:/home/hxy/.ssh:ro \
  pyslam:cuda \
  bash
```
