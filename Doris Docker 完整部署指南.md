# Doris Docker 完整部署指南

## 一、准备工作

### 1.1 创建宿主机目录结构

```bash
# 创建 Doris 相关目录
mkdir -p ~/doris-docker/doris-src      # Doris 源码目录
mkdir -p ~/doris-docker/.m2            # Maven 依赖缓存
mkdir -p ~/doris-docker/fe-data        # FE 数据目录
mkdir -p ~/doris-docker/be-data        # BE 数据目录
mkdir -p ~/doris-docker/logs           # 日志目录
```

### 1.2 克隆 Doris 源码（如果还没有）

```bash
cd ~/doris-docker/doris-src
git clone https://github.com/apache/doris.git
# 或使用国内镜像
# git clone https://gitee.com/mirrors/doris.git
```

## 二、构建镜像

```bash
# 在 Dockerfile 所在目录执行
docker build -t doris-dev:latest .
```

## 三、运行容器（推荐方式）

### 3.1 首次启动容器

```bash
docker run -dit \
  --name doris-dev \
  --restart=always \
  --network host \
  -v ~/doris-docker/doris-src:/home/ocsuser/doris-src \
  -v ~/doris-docker/.m2:/home/ocsuser/.m2 \
  -v ~/doris-docker/fe-data:/home/ocsuser/fe-data \
  -v ~/doris-docker/be-data:/home/ocsuser/be-data \
  -v ~/doris-docker/logs:/home/ocsuser/logs \
  doris-dev:latest \
  /bin/zsh
```

**参数说明：**
- `-dit`: 后台运行并分配终端
- `--name doris-dev`: 容器名称
- `--restart=always`: 容器自动重启（包括 Docker 重启后）
- `--network host`: 使用宿主机网络（无需端口映射）
- `-v`: 挂载目录到容器

### 3.2 进入容器

```bash
docker exec -it doris-dev zsh
```

## 四、编译 Doris

### 4.1 首次编译（在容器内执行）

```bash
cd ~/doris-src

# 编译 FE 和 BE（首次编译会下载依赖，耗时较长）
sh build.sh --clean

# 或分别编译
# sh build.sh --fe --clean    # 只编译 FE
# sh build.sh --be --clean    # 只编译 BE
```

### 4.2 编译产物位置

```bash
# FE 编译产物
~/doris-src/output/fe/

# BE 编译产物
~/doris-src/output/be/
```

## 五、配置自动启动脚本

### 5.1 创建启动脚本

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

### 5.2 配置自动启动（添加到 .zshrc）

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

## 六、配置 Doris

### 6.1 初次配置 FE

```bash
# 编辑 FE 配置文件
vim ~/doris-src/output/fe/conf/fe.conf

# 关键配置项（根据需要调整）：
# meta_dir = /home/ocsuser/fe-data/doris-meta
# sys_log_dir = /home/ocsuser/logs/fe
# priority_networks = 127.0.0.1/32  # 或你的实际网段
```

### 6.2 初次配置 BE

```bash
# 编辑 BE 配置文件
vim ~/doris-src/output/be/conf/be.conf

# 关键配置项（根据需要调整）：
# storage_root_path = /home/ocsuser/be-data/storage
# sys_log_dir = /home/ocsuser/logs/be
# priority_networks = 127.0.0.1/32  # 或你的实际网段
```

## 七、启动和管理

### 7.1 手动启动服务

```bash
# 方式1：使用启动脚本
~/start-doris.sh

# 方式2：分别启动
~/start-fe.sh
~/start-be.sh
```

### 7.2 停止服务

```bash
# 停止 FE
~/doris-src/output/fe/bin/stop_fe.sh

# 停止 BE
~/doris-src/output/be/bin/stop_be.sh
```

### 7.3 查看服务状态

```bash
# 查看进程
ps aux | grep doris

# 查看日志
tail -f ~/logs/fe/fe.log
tail -f ~/logs/be/be.INFO
```

### 7.4 连接 Doris

```bash
# 使用 MySQL 客户端连接（默认端口 9030）
mysql -h 127.0.0.1 -P 9030 -u root

# 添加 BE 节点（首次需要）
ALTER SYSTEM ADD BACKEND "127.0.0.1:9050";

# 查看 BE 状态
SHOW BACKENDS\G
```

## 八、常用命令

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

## 九、重新编译

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

## 十、故障排查

### 10.1 FE 启动失败

```bash
# 查看 FE 日志
tail -f ~/logs/fe/fe.log
tail -f ~/logs/fe/fe.out

# 检查端口占用
lsof -i:9030
lsof -i:8030
```

### 10.2 BE 启动失败

```bash
# 查看 BE 日志
tail -f ~/logs/be/be.INFO
tail -f ~/logs/be/be.out

# 检查端口占用
lsof -i:9050
lsof -i:8040
```

### 10.3 容器重启后服务未自动启动

```bash
# 检查 .zshrc 配置
cat ~/.zshrc | grep start-doris

# 手动启动
~/start-doris.sh
```

## 十一、端口说明

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

## 十二、备份与恢复

```bash
# 备份（在宿主机执行）
tar -czf doris-backup-$(date +%Y%m%d).tar.gz ~/doris-docker/

# 恢复（在宿主机执行）
tar -xzf doris-backup-20240101.tar.gz -C ~/
```
