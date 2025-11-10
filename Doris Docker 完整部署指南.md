# Doris Docker 完整部署指南

## 一、创建 Dockerfile

### 1.1 先克隆 Doris 源码到本地

**重要：必须先下载源码，因为 Dockerfile 需要使用源码中的构建环境**

```bash
# 创建目录
mkdir -p ~/doris-docker/doris-src
cd ~/doris-docker/doris-src

# 克隆 Doris 源码
git clone https://github.com/apache/doris.git
# 或使用国内镜像（如果 GitHub 较慢）
# git clone https://gitee.com/mirrors/doris.git

# 等待克隆完成后，进入源码目录
cd doris
```

### 1.2 创建 Dockerfile 文件

在 Doris 源码的 docker 目录中已经有构建环境的 Dockerfile，我们需要在外部创建一个自定义的 Dockerfile：

```bash
# 创建构建目录
mkdir -p ~/doris-docker-build
cd ~/doris-docker-build
vim Dockerfile
```

将以下内容复制到 Dockerfile 中（使用本地源码中的 Dockerfile 作为基础）：

**方式：直接使用 Doris 官方构建镜像并自定义（更简单）**
参看 官网 ： [Doris Docker 快速搭建开发环境](https://doris.apache.org/zh-CN/community/developer-guide/docker-dev)

```dockerfile
FROM apache/doris:build-env-ldb-toolchain-latest

USER root
WORKDIR /root

# 设置 root 密码
RUN echo '123456' | passwd root --stdin

# 安装额外工具
RUN yum install -y vim net-tools man wget git mysql lsof bash-completion zsh sudo \
    && yum clean all

# 创建用户 ocsuser
RUN useradd -ms /bin/bash ocsuser \
    && echo '0)y^B5vD' | passwd ocsuser --stdin \
    && usermod -a -G wheel ocsuser \
    && echo 'ocsuser ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 切换到普通用户
USER ocsuser
WORKDIR /home/ocsuser

# 配置 Git
RUN git config --global color.ui true \
    && git config --global user.email "piepis@163.com" \
    && git config --global user.name "piepis"

# 配置 Maven（使用国内镜像加速）


# 安装并配置 zsh
USER root
RUN chsh -s /bin/zsh ocsuser

USER ocsuser
RUN wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O - | zsh || true \
    && git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions || true \
    && git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting || true \
    && sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting)/' ~/.zshrc || true

WORKDIR /home/ocsuser

CMD ["/bin/zsh"]
```

**推荐使用方式**，因为 Doris 官方的构建镜像已经包含了所有编译所需的依赖。

保存并退出（`:wq`）

## 二、准备工作

### 2.1 创建其他宿主机目录结构

源码目录已经在第一步创建了，现在创建其他需要的目录：

```bash
# 创建 Maven 缓存和数据目录（源码目录已经在第一步创建）
mkdir -p ~/doris-docker/.m2            # Maven 依赖缓存
mkdir -p ~/doris-docker/fe-data        # FE 数据目录
mkdir -p ~/doris-docker/be-data        # BE 数据目录
mkdir -p ~/doris-docker/logs           # 日志目录
```

### 2.2 验证源码是否下载完成

```bash
# 检查源码目录
ls -la ~/doris-docker/doris-src/doris/
# 应该能看到 fe、be、docker 等目录

# 查看 Doris 版本
cd ~/doris-docker/doris-src/doris
git branch
```

## 三、构建镜像

```bash
# 在 Dockerfile 所在目录执行
cd ~/doris-docker-build
docker build -t doris-dev:latest .
```

**注意事项：**
- 首次构建可能需要较长时间（下载基础镜像和安装依赖）
- **推荐使用方式的 Dockerfile**（基于官方构建镜像），因为已包含所有编译依赖
- 如果 GitHub 访问较慢，可以配置 Docker 代理或使用国内镜像源
- 构建完成后可以使用 `docker images` 查看镜像

**如果构建失败，可以尝试：**
```bash
# 使用代理构建（如果有代理）
docker build --build-arg HTTP_PROXY=http://your-proxy:port \
             --build-arg HTTPS_PROXY=http://your-proxy:port \
             -t doris-dev:latest .

# 或者跳过 oh-my-zsh 安装（手动安装）
# 修改 Dockerfile，在 zsh 安装命令后添加 || true
```

## 四、运行容器（推荐方式）

### 4.1 首次启动容器

```bash
docker run -dit \
  --name doris-dev \
  --restart=always \
  --network host \
  -v ~/doris-docker/doris-src/doris:/home/ocsuser/doris-src \
  -v ~/doris-docker/.m2:/home/ocsuser/.m2 \
  -v ~/doris-docker/fe-data:/home/ocsuser/fe-data \
  -v ~/doris-docker/be-data:/home/ocsuser/be-data \
  -v ~/doris-docker/logs:/home/ocsuser/logs \
  doris-dev:latest \
  /bin/zsh
```

**重要说明：**
- 注意源码挂载路径是 `~/doris-docker/doris-src/doris`（包含 doris 子目录）
- 这样容器内的 `/home/ocsuser/doris-src` 就是 Doris 源码根目录

**参数说明：**
- `-dit`: 后台运行并分配终端
- `--name doris-dev`: 容器名称
- `--restart=always`: 容器自动重启（包括 Docker 重启后）
- `--network host`: 使用宿主机网络（无需端口映射）
- `-v`: 挂载目录到容器

### 4.2 进入容器

```bash
docker exec -it doris-dev zsh
```

## 五、编译 Doris

### 5.1 首次编译（在容器内执行）

```bash
cd ~/doris-src

# 编译 FE 和 BE（首次编译会下载依赖，耗时较长）
sh build.sh --clean

# 或分别编译
# sh build.sh --fe --clean    # 只编译 FE
# sh build.sh --be --clean    # 只编译 BE
```

### 5.2 编译产物位置

```bash
# FE 编译产物
~/doris-src/output/fe/

# BE 编译产物
~/doris-src/output/be/
```

## 六、配置自动启动脚本

### 6.1 创建启动脚本

在容器内创建启动脚本：

```bash
# 进入容器
docker exec -it doris-dev zsh

# 创建 FE 启动脚本
cat > ~/start-fe.sh << 'EOF'
#!/bin/bash
FE_HOME=/home/ocsuser/doris-src/output/fe
if [ -d "$FE_HOME" ]; then
    echo "Starting Doris FE..."
    cd $FE_HOME
    ./bin/start_fe.sh --daemon
    echo "FE started"
else
    echo "FE not found at $FE_HOME, please compile first"
fi
EOF

# 创建 BE 启动脚本
cat > ~/start-be.sh << 'EOF'
#!/bin/bash
BE_HOME=/home/ocsuser/doris-src/output/be
if [ -d "$BE_HOME" ]; then
    echo "Starting Doris BE..."
    cd $BE_HOME
    ./bin/start_be.sh --daemon
    echo "BE started"
else
    echo "BE not found at $BE_HOME, please compile first"
fi
EOF

# 创建统一启动脚本
cat > ~/start-doris.sh << 'EOF'
#!/bin/bash
echo "=== Starting Doris Services ==="
bash ~/start-fe.sh
sleep 5
bash ~/start-be.sh
echo "=== Doris Services Started ==="
EOF

# 赋予执行权限
chmod +x ~/start-fe.sh ~/start-be.sh ~/start-doris.sh
```

### 6.2 配置自动启动（添加到 .zshrc）

```bash
# 在容器内执行
cat >> ~/.zshrc << 'EOF'

# Auto start Doris FE and BE
if [ -f ~/start-doris.sh ]; then
    ~/start-doris.sh
fi
EOF

# 使配置生效
source ~/.zshrc
```

## 七、配置 Doris

### 7.1 初次配置 FE

```bash
# 编辑 FE 配置文件
vim ~/doris-src/output/fe/conf/fe.conf

# 关键配置项（根据需要调整）：
# meta_dir = /home/ocsuser/fe-data/doris-meta
# sys_log_dir = /home/ocsuser/logs/fe
# priority_networks = 127.0.0.1/32  # 或你的实际网段
```

### 7.2 初次配置 BE

```bash
# 编辑 BE 配置文件
vim ~/doris-src/output/be/conf/be.conf

# 关键配置项（根据需要调整）：
# storage_root_path = /home/ocsuser/be-data/storage
# sys_log_dir = /home/ocsuser/logs/be
# priority_networks = 127.0.0.1/32  # 或你的实际网段
```

## 八、启动和管理

### 8.1 手动启动服务

```bash
# 方式1：使用启动脚本
~/start-doris.sh

# 方式2：分别启动
~/start-fe.sh
~/start-be.sh
```

### 8.2 停止服务

```bash
# 停止 FE
~/doris-src/output/fe/bin/stop_fe.sh

# 停止 BE
~/doris-src/output/be/bin/stop_be.sh
```

### 8.3 查看服务状态

```bash
# 查看进程
ps aux | grep doris

# 查看日志
tail -f ~/logs/fe/fe.log
tail -f ~/logs/be/be.INFO
```

### 8.4 连接 Doris

```bash
# 使用 MySQL 客户端连接（默认端口 9030）
mysql -h 127.0.0.1 -P 9030 -u root

# 添加 BE 节点（首次需要）
ALTER SYSTEM ADD BACKEND "127.0.0.1:9050";

# 查看 BE 状态
SHOW BACKENDS\G
```

## 九、常用命令

```bash
# 进入容器
docker exec -it doris-dev zsh

# 查看容器日志
docker logs doris-dev

# 重启容器（会自动启动 FE/BE）
docker restart doris-dev

# 停止容器
docker stop doris-dev

# 启动容器
docker start doris-dev

# 删除容器（保留数据）
docker rm -f doris-dev
```

## 十、重新编译

```bash
# 进入容器
docker exec -it doris-dev zsh

# 停止服务
~/doris-src/output/fe/bin/stop_fe.sh
~/doris-src/output/be/bin/stop_be.sh

# 清理并重新编译
cd ~/doris-src
sh build.sh --clean

# 启动服务
~/start-doris.sh
```

## 十一、故障排查

### 11.1 FE 启动失败

```bash
# 查看 FE 日志
tail -f ~/logs/fe/fe.log
tail -f ~/logs/fe/fe.out

# 检查端口占用
lsof -i:9030
lsof -i:8030
```

### 11.2 BE 启动失败

```bash
# 查看 BE 日志
tail -f ~/logs/be/be.INFO
tail -f ~/logs/be/be.out

# 检查端口占用
lsof -i:9050
lsof -i:8040
```

### 11.3 容器重启后服务未自动启动

```bash
# 检查 .zshrc 配置
cat ~/.zshrc | grep start-doris

# 手动启动
~/start-doris.sh
```

## 十二、端口说明

| 服务 | 端口 | 说明 |
|------|------|------|
| FE | 8030 | HTTP 端口（Web UI） |
| FE | 9030 | MySQL 协议端口 |
| FE | 9010 | RPC 端口 |
| BE | 8040 | HTTP 端口 |
| BE | 9050 | Heartbeat 端口 |
| BE | 9060 | RPC 端口 |
| BE | 8060 | Brpc 端口 |

由于使用 `--network host`，这些端口直接绑定在宿主机上。

## 十三、备份与恢复

```bash
# 备份（在宿主机执行）
tar -czf doris-backup-$(date +%Y%m%d).tar.gz ~/doris-docker/

# 恢复（在宿主机执行）
tar -xzf doris-backup-20240101.tar.gz -C ~/
```
