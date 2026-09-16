本手册包含 1 份源文件：E:\GithubProject\docs\general\Docker部署教程.md

# Docker 部署教程

## 1. 镜像 image

- 一个“打包好的模板”

- 一个 Python 运行环境模板
- 一个 MySQL 模板
- 一个 Nginx 模板

- 模板
- 毛坯
- 安装包

你先把镜像理解成：

比如：

镜像本身不是真正正在运行的程序，它更像：

---

## 2. 容器 container

- 由镜像真正启动出来的“运行中的实例”

- 镜像 = 模具
- 容器 = 用模具做出来的成品

你先把容器理解成：

如果类比：

你最终运行的是容器，不是镜像。

---

## 3. Dockerfile

- “告诉 Docker 应该怎么打包这个项目”的说明书

- 从哪个基础镜像开始
- 工作目录是什么
- 复制哪些文件进去
- 安装哪些依赖
- 最后启动什么命令

这是一个文件名。

你可以把它理解成：

里面会写：

---

## 4. `docker build`

- 按照 `Dockerfile` 把镜像做出来

你可以把它理解成：

---

## 5. `docker run`

- 把已经做好的镜像启动成容器

你可以把它理解成：

---

## 6. `docker compose`

为什么需要它？

- “一次同时管理多个容器的工具”

- 后端容器
- 前端容器
- MySQL 容器
- Redis 容器
- RabbitMQ 容器

- `docker-compose.yml`

```powershell
docker compose up
```

这个非常重要。

你先把它理解成：

因为像 `toutiaoA` 这种项目，不是一个容器就够了。它至少会有：

如果你每个都手写一大串 `docker run`，会非常乱。所以要用：

这个文件统一写清楚，然后用：

一次全部启动。

## 5. 第三步：把 `ZBot` 根目录的 `Dockerfile` 改成下面这样

- `E:\LLMsApplicationDevelopment\ZBot\Dockerfile`

```dockerfile
FROM python:3.13-slim

请打开：

替换成下面内容：

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV UV_LINK_MODE=copy

WORKDIR /app

RUN pip install --no-cache-dir uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen

COPY . .

CMD ["uv", "run", "python", "-m", "ZBot", "--help"]
```

---

## 6. 这一份 `Dockerfile` 每一行都是什么意思

### `FROM python:3.13-slim`

- 这次打包从一个已经带 Python 3.13 的基础镜像开始

- “先拿一个干净的 Python 3.13 小型 Linux 系统作为起点”

- 精简版

- 体积更小
- 下载更快一点

意思是：

你可以把它理解成：

#### `slim`

意思是：

也就是：

### `ENV PYTHONDONTWRITEBYTECODE=1`

- 这是为了让容器里更干净

- 不要生成多余的 `.pyc` 缓存字节码文件

意思是：

你现在不用深究原理，只要知道：

### `ENV PYTHONUNBUFFERED=1`

- Python 输出日志时尽量立刻显示

意思是：

这样你看容器日志更方便。

### `ENV UV_LINK_MODE=copy`

- 让 `uv` 在容器里安装依赖时使用复制方式

意思是：

这是为了让容器构建更稳定。

### `WORKDIR /app`

- 容器里后续操作都以 `/app` 作为工作目录

- “容器里的项目根目录以后就叫 `/app`”

意思是：

你可以把它理解成：

### `RUN pip install --no-cache-dir uv`

- 在镜像里先安装 `uv`

- 在构建镜像的时候执行一条命令

- 安装完以后不要额外保留 pip 下载缓存

意思是：

#### `RUN`

意思是：

#### `--no-cache-dir`

意思是：

这样镜像更干净。

### `COPY pyproject.toml uv.lock ./`

为什么只先复制这两个？

- 先把项目的依赖说明文件复制进镜像

- 依赖文件变化没那么频繁
- Docker 可以更好地利用缓存

意思是：

因为：

### `RUN uv sync --frozen`

- 按 `uv.lock` 里已经锁定好的版本安装依赖

- 同步依赖

- 严格按照当前锁文件来
- 不要擅自改锁文件

意思是：

#### `sync`

意思是：

#### `--frozen`

意思是：

### `COPY . .`

- 再把整个项目代码复制进镜像

- 当前电脑里的项目目录

- 容器当前工作目录 `/app`

意思是：

第一个点表示：

第二个点表示：

### `CMD ["uv", "run", "python", "-m", "ZBot", "--help"]`

为什么先用 `--help`？

- 容器默认启动时，先显示 `ZBot` 的帮助信息

- 它最安全
- 它最容易验证镜像有没有构建成功

意思是：

因为：

---

## 8. 第五步：把 `ZBot` 根目录的 `docker-compose.yml` 改成下面这样

- `E:\LLMsApplicationDevelopment\ZBot\docker-compose.yml`

```yaml
services:
  zbot:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: zbot
    stdin_open: true
    tty: true
    volumes:
      - ${USERPROFILE}/.ZBot:/root/.ZBot
      - .:/app
    working_dir: /app
    command: ["uv", "run", "python", "-m", "ZBot", "--help"]
    restart: "no"
```

请打开：

替换成下面内容：

---

## 9. 这份 `docker-compose.yml` 每一项是什么意思

### `services`

- 下面开始定义有哪些服务容器

意思是：

### `zbot`

- “这个容器组里的 ZBot 服务”

这是这个服务的名字。你可以把它理解成：

### `build`

- 不直接下载现成镜像
- 而是用当前目录里的 `Dockerfile` 自己构建镜像

意思是：

### `context: .`

- 构建镜像时，以当前目录为上下文

- Docker 构建时可以看到的项目文件范围

意思是：

你现在先把“上下文”理解成：

### `dockerfile: Dockerfile`

- 构建时使用当前目录里的 `Dockerfile`

意思是：

### `container_name: zbot`

- 给启动出来的容器起一个固定名字：`zbot`

意思是：

### `stdin_open: true`

为什么要它？

- 保持标准输入打开

意思是：

因为 `ZBot` 是命令行交互工具。
如果不保留输入，你没法和它对话。

### `tty: true`

- 分配一个终端

- “让容器像一个正常终端程序一样工作”

意思是：

你可以把它理解成：

### `volumes`

- 做目录挂载

- 把你 Windows 主机里的 `C:\Users\你的用户名\.ZBot`
- 挂载到容器里的 `/root/.ZBot`

- 你的配置文件不会随着容器删除而丢失

- 把当前项目目录挂进容器里的 `/app`

意思是：

这很重要。

#### `${USERPROFILE}/.ZBot:/root/.ZBot`

意思是：

这样做的好处是：

#### `.:/app`

意思是：

这样容器可以直接访问你的项目代码。

### `working_dir: /app`

- 容器运行命令时，以 `/app` 为当前目录

意思是：

### `command`

- 容器启动后默认执行什么命令

- 显示 `ZBot` 帮助信息

意思是：

这次先设成：

### `restart: "no"`

为什么？

  为什么用 Volume 而不是直接存宿主机目录？
    1. 容器里的 Linux 文件系统和 Windows 文件系统不一样
    2. 直接映射 Windows 路径可能有权限、性能问题
    3. Volume 由 Docker 管理，更稳定
```

```
Milvus 的数据不是存在你的项目目录里，而是存在 Docker 管理的"卷"里。

```
Docker Compose 会自动给卷名加项目前缀：

```
查看卷的命令：

```
启动 Milvus（推荐方式）：

```
查看容器是否在运行：

```
查看所有容器（包括已退出的）：

```
查看容器日志：

```
查看容器端口映射：

- 这个容器退出后，不要自动重启

意思是：

因为 `ZBot` 是 CLI 工具，不是那种必须长期后台运行的服务。

---

#### 4）Docker Volume：Milvus 数据存在哪里

  Docker Volume（卷）是什么？
    Docker 为容器分配的一块持久化存储空间。
    容器删了，卷里的数据还在（除非你主动删卷）。
    类比：U 盘。容器是电脑，U 盘是卷，电脑坏了 U 盘数据还在。

  docker-compose.yml 里写的卷名：
    volumes:
      milvus_data:

  实际创建出来的卷名：
    项目目录名_卷名

  例：项目目录叫 toutiaoa
    实际卷名 = toutiaoa_milvus_data

  所以你 docker volume ls 看到的卷名会多一个前缀，这是正常的。
```

  docker volume ls
    volume        → Docker 卷管理子命令
    ls            → list（列出所有卷）

  docker volume inspect toutiaoa_milvus_data
    inspect       → 查看卷的详细信息（创建时间、挂载路径、大小等）
```

#### 2）Docker 命令详解

  docker compose up -d

    docker compose  → Docker 的多容器编排工具，读取 docker-compose.yml 文件
    up              → 创建并启动 yml 里定义的所有容器
    -d              → detached 模式，后台运行（不占用终端）

  执行后，Docker 会根据 docker-compose.yml 创建 etcd、MinIO、Milvus 三个容器。
```

  docker ps

    docker  → Docker 命令行工具
    ps      → process status（进程状态），列出正在运行的容器

  输出示例：
  CONTAINER ID  IMAGE              STATUS         PORTS
  a1b2c3d4e5f6  milvusdb/milvus    Up 2 minutes   0.0.0.0:19530->19530/tcp
  f6e5d4c3b2a1  quay.io/coreos/etcd Up 2 minutes  2379-2380/tcp
  1a2b3c4d5e6f  minio/minio        Up 2 minutes   9000/tcp

  STATUS 是 Up → 容器在运行
  没有出现 → 容器没启动或已退出
```

  docker ps -a

    -a  → all（全部），包括正在运行的和已停止的

  如果 docker ps 看不到 Milvus，用 docker ps -a 看它是否退出了。
  退出的容器 STATUS 会显示 Exited (退出码)。
```

  docker logs milvus --tail 100

    docker logs  → 查看容器的输出日志（标准输出和错误输出）
    milvus       → 容器名称
    --tail 100   → 只显示最后 100 行（不加会显示全部日志，可能很长）

  日志里能看到 Milvus 启动是否成功、报了什么错。
  常见错误：连不上 etcd、端口被占用、内存不足。
```

  docker port milvus

    port  → 显示容器的端口映射关系

  输出示例：
  19530/tcp -> 0.0.0.0:19530
  9091/tcp -> 0.0.0.0:9091

  19530 → Milvus gRPC/HTTP 客户端访问端口（Python SDK 连这个端口）
  9091  → Milvus metrics/健康检查端口（监控用）
```

## 10. 第六步：先构建 ZBot 镜像

```powershell
docker compose build
```

在 `E:\LLMsApplicationDevelopment\ZBot` 目录里执行：

### 这条命令是什么意思

- 使用 Docker Compose 方式操作

- 根据 `docker-compose.yml` 里写的构建规则，去构建镜像

#### `docker compose`

表示：

#### `build`

意思是：

### 这一步做完后发生什么

Docker 会：

1. 读取 `docker-compose.yml`
2. 找到 `zbot` 服务
3. 读取它指定的 `Dockerfile`
4. 开始构建镜像

---

## 11. 第七步：先测试 ZBot 镜像能不能启动

```powershell
docker compose run --rm zbot
```

### 这条命令每一部分是什么意思

为什么这个参数好？

- 临时启动一个容器来执行这个服务

- 命令执行完以后，自动删除这个临时容器

- 不会留下很多没用的临时容器垃圾

- 启动 `docker-compose.yml` 里那个叫 `zbot` 的服务

#### `run`

意思是：

#### `--rm`

意思是：

因为：

#### `zbot`

表示：

### 你期望看到什么

- 镜像构建成功了
- 依赖也基本装好了

你应该看到 `ZBot` 的帮助信息。只要能看到帮助信息，就说明：

---

## 12. 第八步：第一次初始化 ZBot 配置

```powershell
docker compose run --rm zbot uv run python -m ZBot onboard
```

如果你是第一次在 Docker 里运行 ZBot，可以执行：

### 这条命令在做什么

- 临时启动 `zbot` 容器
- 然后在容器里执行 `ZBot` 的初始化命令

它的意思是：

### 为什么这里还要写 `uv run`

因为我们希望容器里也严格走项目自己的依赖环境。

---

## 13. 第九步：正式运行 ZBot

```powershell
docker compose run --rm zbot uv run python -m ZBot agent
```

### 这条命令是什么意思

- 启动一个临时的 `ZBot` 容器
- 在里面运行 `agent` 模式

它表示：

### 部署成功的标志是什么

- 容器能正常启动
- 你能在容器里看到 `ZBot` 交互

不是浏览器打开页面。而是：

---

## 第四部分：再部署 toutiaoA，因为它是完整 Web 项目

- Python 后端
- MySQL
- Redis
- RabbitMQ
- React 前端

`toutiaoA` 比 `ZBot` 难很多。
因为它不是一个单容器项目。

它至少要用到：

所以你要接受一个事实：

**部署 `toutiaoA`，本质上是在一次启动 5 个容器。**

---

## 1. 先理解：这次我们部署的是 `toutiaoA` 的完整最小可运行版本

我们这次的目标是：

1. 后端 API 容器能启动
2. MySQL 容器能启动
3. Redis 容器能启动
4. RabbitMQ 容器能启动
5. 前端页面容器能启动

### 最终你会访问哪些地址

部署成功后，你大概率会访问：

- 前端页面：`http://localhost:8080`
- 后端接口文档：`http://localhost:8000/docs`
- RabbitMQ 管理后台：`http://localhost:15673`

---

## 2. 为什么 `toutiaoA` 不能继续用原来的 `.env`

- `E:\LLMsApplicationDevelopment\toutiaoA\.env`

- `localhost`

你现在项目根目录里的：

我已经看过了，里面很多主机地址还是：

这在你本机直跑时没问题。
但是一旦进 Docker 容器，就会出问题。

### 为什么

- `localhost` 指的是容器自己

- 你的 Windows 主机
- 也不是别的容器

因为在容器里面：

不是：

### 举个例子

```text
MYSQL_HOST=localhost
```

- “后端容器自己内部有没有 MySQL”

如果后端容器里写：

那它实际上是在找：

但真正的 MySQL 在另一个容器里。
所以会连不上。

### 正确做法是什么

- `mysql`
- `redis`
- `rabbitmq`

在 Docker Compose 里，容器之间应该用服务名互相访问，比如：

---

## 3. 第一步：进入 `toutiaoA` 根目录

```powershell
cd E:\LLMsApplicationDevelopment\toutiaoA
```

---

## 4. 第二步：创建 `toutiaoA` 专用的 `.dockerignore`

- `E:\LLMsApplicationDevelopment\toutiaoA`

- `.dockerignore`

```text
.git
.gitignore
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
.mypy_cache
.ruff_cache
node_modules
client/node_modules
client/dist
logs
root-dev.err.log
root-dev.out.log
server-dev.err.log
server-dev.out.log
.env
.env.docker
```

请在：

根目录新建：

写入下面内容：

### 为什么这里把 `.env` 和 `.env.docker` 也忽略

- 环境变量文件里经常有密码和密钥
- 不应该直接打包进镜像

- `env_file`

因为：

我们后面会用：

让容器运行时再读取，而不是在构建镜像时写死进去。

---

## 5. 第三步：创建 `toutiaoA` 后端的 `Dockerfile`

- `E:\LLMsApplicationDevelopment\toutiaoA\Dockerfile`

```dockerfile
FROM python:3.13-slim

请在根目录新建文件：

写入下面内容：

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV UV_LINK_MODE=copy

WORKDIR /app

RUN pip install --no-cache-dir uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen

COPY . .

EXPOSE 8000

CMD ["uv", "run", "python", "main.py"]
```

### 这份后端 `Dockerfile` 的思路是什么

- 最后启动命令变成了 `python main.py`
- 并且声明了 `8000` 端口

和前面的 `ZBot` 很像，只不过：

### `EXPOSE 8000` 是什么意思

- 告诉 Docker：这个容器主要会对外使用 8000 端口

- 它不是自动开放端口
- 真正映射端口还要在 `docker-compose.yml` 里写

注意：

意思是：

---

## 6. 第四步：创建 `toutiaoA` 前端的 Dockerfile

- `E:\LLMsApplicationDevelopment\toutiaoA\client`

- `Dockerfile`

```dockerfile
FROM node:20-alpine AS build

请在：

目录里新建文件：

写入下面内容：

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

ARG VITE_API_BASE_URL=http://localhost:8000
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}

RUN npm run build

FROM nginx:1.27-alpine

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## 7. 这份前端 Dockerfile 在做什么

- 第一段：负责把 React/Vite 前端打包
- 第二段：负责用 Nginx 把打包好的静态页面提供出来

它分成两段，你先只要理解成：

### `FROM node:20-alpine AS build`

- 第一阶段用 Node 20 精简镜像来构建前端

- 给这一阶段起个名字叫 `build`

意思是：

#### `AS build`

意思是：

后面第二阶段会从这里拿打包结果。

### `RUN npm ci`

- 按锁文件精确安装前端依赖

意思是：

### `ARG VITE_API_BASE_URL=http://localhost:8000`

- 构建前端时，允许传一个后端接口地址

意思是：

### 为什么这里是 `http://localhost:8000`

- 你主机映射出来的 8000 端口

因为最终浏览器运行在你电脑上。浏览器访问后端接口时，访问的是：

### 第二阶段为什么用 `nginx`

- 用 Nginx 来提供这些静态页面

因为前端打包完以后，本质上是一堆静态文件。最常见的做法就是：

---

## 8. 第五步：创建 `toutiaoA` 的 Docker 专用环境变量文件

- `E:\LLMsApplicationDevelopment\toutiaoA\.env.docker`

```text
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=123456
MYSQL_DB_NAME=news_app
MYSQL_DB_POOL_SIZE=20
MYSQL_DB_OVERFLOW=40

请在根目录新建：

写入下面内容：

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest
RABBITMQ_VHOST=/

OPENAI_API_KEY=请替换成你自己的真实密钥
OPENAI_API_BASE=https://api.openai.com/v1
LLM_ANALYZE_MODEL=Pro/MiniMaxAI/MiniMax-M2.5
LLM_GENERATE_MODEL=gpt-4o-mini

RAG_PRELOAD_ON_STARTUP=false
```

---

## 9. 这份 `.env.docker` 里最关键的 3 个变化

### 变化 1：`MYSQL_HOST=mysql`

- 后端容器要去找名字叫 `mysql` 的容器

意思是：

### 变化 2：`REDIS_HOST=redis`

- 后端容器要去找名字叫 `redis` 的容器

意思是：

### 变化 3：`RABBITMQ_HOST=rabbitmq`

- 后端容器要去找名字叫 `rabbitmq` 的容器

意思是：

这 3 个名字，后面你会在 `docker-compose.yml` 里看到。
它们不是随便写的。

### 为什么我这里把 `RAG_PRELOAD_ON_STARTUP` 改成了 `false`

- `false`

因为你这个项目有大模型和向量库预热。
Docker 第一次启动时，如果一上来就预热，可能会更慢，也更容易让你误判“是不是起不来”。

所以新手第一次先改成：

更稳妥。

---

## 10. 第六步：创建 `toutiaoA` 的 `docker-compose.yml`

- `E:\LLMsApplicationDevelopment\toutiaoA\docker-compose.yml`

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: toutiao-mysql
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: news_app
    ports:
      - "3307:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p123456"]
      interval: 10s
      timeout: 5s
      retries: 10

请在根目录新建：

写入下面内容：

  redis:
    image: redis:7-alpine
    container_name: toutiao-redis
    ports:
      - "6380:6379"
    volumes:
      - redis-data:/data

  rabbitmq:
    image: rabbitmq:3-management
    container_name: toutiao-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5673:5672"
      - "15673:15672"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

  backend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: toutiao-backend
    env_file:
      - .env.docker
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
      rabbitmq:
        condition: service_started
    ports:
      - "8000:8000"
    volumes:
      - ./data:/app/data

  client:
    build:
      context: ./client
      dockerfile: Dockerfile
      args:
        VITE_API_BASE_URL: http://localhost:8000
    container_name: toutiao-client
    depends_on:
      - backend
    ports:
      - "8080:80"

volumes:
  mysql-data:
  redis-data:
  rabbitmq-data:
```

---

## 11. 这份 `docker-compose.yml` 要怎么理解

你不要一次全背。
我按服务给你拆开讲。

---

## 11.1 `mysql` 服务

### `image: mysql:8.0`

- 直接使用官方 MySQL 8.0 镜像

意思是：

### `container_name: toutiao-mysql`

- 给这个 MySQL 容器取名字

意思是：

### `environment`

- 给 MySQL 容器传初始化环境变量

- 设置 root 密码

- 容器第一次启动时自动创建一个叫 `news_app` 的数据库

意思是：

#### `MYSQL_ROOT_PASSWORD: 123456`

表示：

#### `MYSQL_DATABASE: news_app`

表示：

### `ports`

```yaml
- "3307:3306"
```

- 你电脑的 3307 端口，对应容器里的 3306 端口

意思是：

### 为什么不用 `3306:3306`

- 主机 3307
- 容器 3306

因为很多人本机已经装了 MySQL，3306 很可能已经被占用了。所以我们故意改成：

### `volumes`

- 把 MySQL 数据存进命名卷里

意思是：

这样即使你把容器删掉，数据库数据也不一定丢。

### `healthcheck`

为什么要它？

- 启动后周期性检查 MySQL 有没有真的准备好

意思是：

因为“容器启动了”不等于“数据库已经能连了”。

---

## 11.2 `redis` 服务

### `image: redis:7-alpine`

- 使用 Redis 7 的轻量镜像

表示：

### `ports`

为什么用 6380？

```yaml
- "6380:6379"
```

- 你电脑访问 6380
- 实际对应容器里的 6379

表示：

因为很多人本机自己的 Redis 已经占用了 6379。

---

## 11.3 `rabbitmq` 服务

### `image: rabbitmq:3-management`

- 使用带管理后台的 RabbitMQ 镜像

意思是：

这个比纯 RabbitMQ 镜像好理解，因为你还能打开网页看它的后台。

### `ports`

```yaml
- "5673:5672"
- "15673:15672"
```

- 主机 5673 对应容器消息队列端口 5672

- 主机 15673 对应容器管理后台端口 15672

第一条表示：

第二条表示：

### 管理后台怎么打开

- `http://localhost:15673`

- `guest`
- `guest`

后面成功后，你可以在浏览器访问：

账号密码就是：

---

## 11.4 `backend` 服务

这就是 `toutiaoA` 的 Python 后端。

### `build`

- 后端不是下载现成镜像
- 而是用当前目录里的 `Dockerfile` 自己构建

表示：

### `env_file`

- 运行后端容器时，读取 `.env.docker` 里的环境变量

表示：

### `depends_on`

- 后端依赖其他服务

- MySQL 至少要健康
- Redis 至少要启动
- RabbitMQ 至少要启动

表示：

在这里表示：

### `ports`

```yaml
- "8000:8000"
```

- 你电脑访问 8000
- 对应容器里的后端 8000

- `http://localhost:8000/docs`

表示：

所以后面你访问接口文档时会打开：

### `volumes`

```yaml
- ./data:/app/data
```

- 把项目根目录下的 `data` 文件夹，映射到容器里的 `/app/data`

- 向量库、模型缓存等数据不容易因为容器删除而彻底消失

表示：

这样做的好处是：

---

## 11.5 `client` 服务

这就是前端页面服务。

### `build.context: ./client`

- 前端镜像构建时，以 `client` 目录为构建上下文

表示：

### `args`

- 给前端 Dockerfile 的构建参数赋值

```yaml
VITE_API_BASE_URL: http://localhost:8000
```

- 前端页面最终请求后端接口时，去访问你电脑上的 8000 端口

表示：

这里写：

意思是：

### `ports`

```yaml
- "8080:80"
```

- 你电脑的 8080 端口
- 对应前端容器里的 80 端口

- `http://localhost:8080`

表示：

所以最后你打开页面时访问：

---

## 12. 第七步：构建并启动 toutiaoA 全套容器

```powershell
docker compose up -d --build
```

在 `E:\LLMsApplicationDevelopment\toutiaoA` 根目录执行：

---

## 13. 这条命令每一部分是什么意思

为什么建议第一次一定加它？

- 把 `docker-compose.yml` 里写的服务启动起来

- 后台运行

- 启动前先重新构建镜像

#### `up`

意思是：

#### `-d`

意思是：

如果不加 `-d`，终端会一直被占住，日志会不断刷屏。

#### `--build`

意思是：

因为你刚刚新写了 Dockerfile。
必须让 Docker 先按最新文件构建一次。

---

## 14. 第八步：检查 toutiaoA 的容器是不是都起来了

```powershell
docker compose ps
```

执行：

### 这条命令是什么意思

- 查看这个 compose 项目里的服务容器状态

#### `ps`

意思是：

### 你希望看到什么

- `toutiao-mysql`
- `toutiao-redis`
- `toutiao-rabbitmq`
- `toutiao-backend`
- `toutiao-client`

你应该看到这些服务大体都处于运行状态：

---

## 15. 第九步：看日志，判断是不是哪里没启动好

```powershell
docker compose logs -f
```

如果有服务没起来，执行：

### 这条命令是什么意思

- 看日志

- 持续跟着看，就像实时刷日志一样

#### `logs`

意思是：

#### `-f`

意思是：

### 如果你只想看后端日志

```powershell
docker compose logs -f backend
```

### 如果你只想看前端日志

```powershell
docker compose logs -f client
```

### 如果你只想看 MySQL 日志

```powershell
docker compose logs -f mysql
```

---

## 第五部分：部署后最常用的 8 条命令

这一部分你以后会经常用到。
我把最常见的命令都用最白话的方式解释一遍。

---

## 1. 查看正在运行的容器

```powershell
docker ps
```

- 查看当前所有“正在运行”的容器

意思：

---

## 2. 查看所有容器，包括已经停止的

```powershell
docker ps -a
```

- 查看所有容器，不管它们现在在不在运行

意思：

---

## 3. 查看 compose 项目里的服务状态

```powershell
docker compose ps
```

- 只看当前 compose 项目里的服务情况

意思：

---

## 4. 启动 compose 项目

```powershell
docker compose up -d
```

- 后台启动当前 compose 项目的所有服务

意思：

---

## 5. 停止 compose 项目

```powershell
docker compose down
```

- 把当前 compose 项目的容器停掉并删除

- 这不会自动删除命名卷
- 所以数据库数据通常不会立刻消失

注意：

意思：

---

## 6. 重建并启动

```powershell
docker compose up -d --build
```

- 先重建镜像
- 再后台启动

- 你刚改了 Dockerfile
- 你刚改了前后端代码并想重新构建镜像

意思：

适合什么时候用？

---

## 7. 看日志

```powershell
docker compose logs -f
```

- 持续看整个项目所有服务的日志

意思：

---

## 8. 进入容器内部

```powershell
docker compose exec backend sh
```

### 这条命令是什么意思

- 进入一个已经在运行的容器里执行命令

- 进入名叫 `backend` 的服务容器

- 进入一个最基础的 Linux shell

#### `exec`

意思是：

#### `backend`

表示：

#### `sh`

表示：

### 这一招什么时候有用

- 容器里有没有某个文件
- 容器里环境变量对不对
- 容器里依赖是不是装好了

比如你想检查：

---

## 2. 环境隔离：一台电脑同时跑多个项目互不干扰

### 先讲一个你可能以后会遇到的场景

- 需要 Python 3.11
- 需要 MySQL 5.7
- 需要 Redis 6.0

- 需要 Python 3.13
- 需要 MySQL 8.0
- 需要 Redis 7.0

- 你电脑上只能装一个 Python 版本
- 你电脑上只能装一个 MySQL 版本
- 你电脑上只能装一个 Redis 版本

假设你现在有两个项目：

**项目 A：**

**项目 B：**

如果你直接在本机安装，就会遇到问题：

你怎么同时跑这两个项目？

### 为什么会冲突？深入分析

- 配置更复杂
- 你的程序也要改连接端口

- 配置复杂
- 容易出错
- 不同软件之间可能还有依赖冲突

要理解这个问题，我们先要理解：**为什么一台电脑不能同时装两个版本的 Python？**

#### 软件安装的本质

当你在电脑上安装软件时，实际上发生了什么？

**Windows 上安装 Python：**

1. 把 Python 解释器文件复制到 `C:\Python313\`
2. 把 Python 添加到系统 PATH 环境变量
3. 在注册表里写入一些信息

**问题来了：**

如果你再安装 Python 3.11，它会：

1. 把 Python 解释器文件复制到 `C:\Python311\`
2. 修改系统 PATH 环境变量
3. 修改注册表

当你输入 `python` 命令时，系统会根据 PATH 环境变量来决定运行哪个版本。

**所以：** 不是不能装两个版本，而是系统不知道你想用哪个版本。

你可以用一些技巧（比如 pyenv、conda）来管理多个版本，但是：

#### MySQL 的问题更严重

MySQL 的问题比 Python 更严重，因为：

1. MySQL 需要监听特定的端口（默认 3306）
2. 一个端口只能被一个程序占用
3. 如果你装了两个 MySQL，它们会抢同一个端口

你可以让第二个 MySQL 用不同的端口（比如 3307），但是：

### Docker 怎么解决这个问题？原理是什么？

```
Docker容器依赖Linux内核的三个特性：

```
执行 docker run nginx 时：

```
容器内的视图 vs 宿主机的视图：

实现原理：
struct pid_namespace {
    struct pid *pid;        // 容器内的PID
    struct pid *real_pid;   // 宿主机的PID
};

```
限制容器内存为512MB：

Docker 使用了一种叫做 **Namespace（命名空间）** 的技术来实现隔离。

**容器的核心技术**

1. Namespace（命名空间）：隔离视图
   - PID namespace：进程ID隔离
   - Network namespace：网络隔离
   - Mount namespace：文件系统隔离
   - UTS namespace：主机名隔离
   - User namespace：用户隔离

2. Cgroups（控制组）：资源限制
   - 限制CPU使用
   - 限制内存使用
   - 限制磁盘IO
   - 限制网络带宽

3. UnionFS（联合文件系统）：分层存储
   - 镜像由多层组成
   - 容器层是可写的
   - 镜像层是只读的
```

**创建容器时发生了什么**

1. Docker客户端发送请求到Docker守护进程

2. Docker守护进程检查镜像是否存在：
   - 不存在：从仓库拉取
   - 存在：直接使用

3. 创建容器：
   int create_container() {
       // 1. 创建namespace
       clone(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | ...);
   
       // 2. 设置cgroup限制
       mkdir("/sys/fs/cgroup/memory/docker/容器ID");
       echo "512M" > memory.limit_in_bytes;
   
       // 3. 挂载文件系统
       mount("overlay", "/var/lib/docker/...", "overlayfs", ...);
   
       // 4. 执行容器进程
       execve("/docker-entrypoint.sh", ...);
   }

4. 容器进程启动：
   - 在新的PID namespace中，PID从1开始
   - 在新的Mount namespace中，有自己的文件系统
   - 在新的Network namespace中，有自己的网络栈
```

**Namespace隔离效果**

容器内：
# ps aux
PID   USER     COMMAND
1     root     nginx  ← PID 1
2     root     worker
3     root     worker

宿主机：
# ps aux | grep nginx
12345 root     nginx  ← 实际PID是12345
12346 root     worker
12347 root     worker

同一个进程，在容器内看到PID是1，在宿主机看到PID是12345

进程调度时：
- 内核用real_pid调度
- 用户空间看到的是namespace内的pid
```

**Cgroup资源限制**

# 创建cgroup
mkdir /sys/fs/cgroup/memory/docker/abc123

# 设置内存限制
echo 536870912 > /sys/fs/cgroup/memory/docker/abc123/memory.limit_in_bytes

# 把进程加入cgroup
echo 12345 > /sys/fs/cgroup/memory/docker/abc123/cgroup.procs

#### 什么是 Namespace？

#### 为什么多个容器都能用 3306 端口？

```
Docker镜像由多层组成：

```
容器有自己的网络栈：

```
虚拟机：
┌─────────────────────────────────────┐
│ 应用程序                              │
├─────────────────────────────────────┤
│ Guest操作系统（完整内核）              │
├─────────────────────────────────────┤
│ Hypervisor                           │
├─────────────────────────────────────┤
│ Host操作系统                          │
├─────────────────────────────────────┤
│ 硬件                                  │
└─────────────────────────────────────┘

```
虚拟机：
  ┌─────────────────────────────┐
  │ 虚拟机                        │
  │  ┌─────────────────────┐    │
  │  │ 完整的操作系统         │    │
  │  │ （几百MB到几GB）       │    │
  │  └─────────────────────┘    │
  │  应用程序                    │
  └─────────────────────────────┘
  每个虚拟机需要完整的操作系统，启动慢，占用多

```
┌─────────────────────────────────────────────┐
│              整个系统                        │
│  ┌─────────────────────────────────────┐   │
│  │  进程 1、进程 2、进程 3...            │   │
│  │  所有进程都在同一个"空间"里           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────┐
│              整个系统                        │
│  ┌───────────┐  ┌───────────┐  ┌──────────┐│
│  │ 容器 A    │  │ 容器 B    │  │ 容器 C   ││
│  │ 的空间    │  │ 的空间    │  │ 的空间   ││
│  │           │  │           │  │          ││
│  │ 进程 1,2  │  │ 进程 3,4  │  │ 进程 5,6 ││
│  └───────────┘  └───────────┘  └──────────┘│
└─────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                        宿主机                                │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │    容器 A       │    │    容器 B       │                │
│  │                 │    │                 │                │
│  │  MySQL 5.7      │    │  MySQL 8.0      │                │
│  │  监听 3306 端口  │    │  监听 3306 端口  │                │
│  │                 │    │                 │                │
│  │  容器内部 IP:    │    │  容器内部 IP:    │                │
│  │  172.17.0.2     │    │  172.17.0.3     │                │
│  └─────────────────┘    └─────────────────┘                │
│           │                     │                          │
│           ▼                     ▼                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Docker 网桥（docker0）                  │   │
│  │              负责容器之间的网络转发                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              宿主机网络                              │   │
│  │                                                     │   │
│  │  端口映射：                                          │   │
│  │  宿主机 3307 → 容器 A 的 3306                        │   │
│  │  宿主机 3308 → 容器 B 的 3306                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

| Namespace 类型    | 隔离的内容     | 通俗解释                                                            |
| ----------------- | -------------- | ------------------------------------------------------------------- |
| PID Namespace     | 进程 ID        | 每个容器有自己的进程编号，容器里的进程 1 和容器外的进程 1 是不同的  |
| Network Namespace | 网络设备、端口 | 每个容器有自己的网卡、IP 地址、端口，所以多个容器都可以用 3306 端口 |
| Mount Namespace   | 文件系统挂载点 | 每个容器有自己的文件系统，互不影响                                  |
| UTS Namespace     | 主机名         | 每个容器可以有自己的主机名                                          |
| IPC Namespace     | 进程间通信     | 每个容器有自己的进程间通信资源                                      |
| User Namespace    | 用户和用户组   | 每个容器可以有自己的用户体系                                        |

内核检查：
每次进程申请内存时：
if (current_memory + request > limit) {
    // 内存超限
    if (can_swap) {
        // 尝试交换到磁盘
    } else {
        // 触发OOM，杀死进程
        out_of_memory(current);
    }
}
```

**镜像分层存储**

┌─────────────────────────────────────┐
│ 容器层（可写）                        │  ← 容器修改写在这里
├─────────────────────────────────────┤
│ 镜像层3：应用代码                     │  ← 只读
├─────────────────────────────────────┤
│ 镜像层2：Python运行时                 │  ← 只读
├─────────────────────────────────────┤
│ 镜像层1：基础系统（Ubuntu）            │  ← 只读
└─────────────────────────────────────┘

读取文件：
1. 从容器层开始查找
2. 找不到则逐层向下查找

写入文件：
1. 从镜像层复制文件到容器层（Copy-on-Write）
2. 修改容器层的副本

好处：
- 多个容器可以共享相同的镜像层
- 节省磁盘空间
- 加快镜像构建速度
```

**容器网络**

1. 创建veth pair（虚拟网卡对）：
   ip link add veth0 type veth peer name veth1

2. 一端放入容器：
   ip link set veth1 netns <容器PID>

3. 另一端连接到docker0网桥：
   ip link set veth0 master docker0

4. 容器内配置IP：
   ip addr add 172.17.0.2/16 dev eth0

网络流量路径：
容器进程 → eth0 → veth1 → veth0 → docker0 → 宿主机网卡 → 外部网络

每个容器有自己的IP地址，可以互相通信
```

**容器 vs 虚拟机**

容器：
┌─────────────────────────────────────┐
│ 应用程序                              │
├─────────────────────────────────────┤
│ Docker引擎（namespace + cgroup）      │
├─────────────────────────────────────┤
│ Host操作系统（共享内核）               │
├─────────────────────────────────────┤
│ 硬件                                  │
└─────────────────────────────────────┘

关键区别：
- 虚拟机：每个虚拟机运行完整操作系统
- 容器：所有容器共享宿主机内核
- 容器更轻量，启动更快
```

**容器 vs 虚拟机**：

容器：
  ┌─────────────────────────────┐
  │ 容器                         │
  │  应用程序 + 依赖              │
  │  （几十MB）                  │
  └─────────────────────────────┘
  共享宿主机操作系统内核，启动快，占用少
```

Namespace 是 Linux 内核提供的一种功能，它可以把系统的全局资源"分割"成多个独立的区域。

**举个例子：**

没有 Namespace 时：

有 Namespace 后：

每个容器都以为自己独占整个系统，但实际上它们只是在使用系统的一部分资源。

#### Docker 具体隔离了什么？

Docker 使用了多种 Namespace 来隔离不同的资源：

这是新手最容易困惑的问题。

**关键理解：** 每个容器都有自己独立的 Network Namespace。

**解释：**

1. 容器 A 里面，MySQL 监听的是容器 A 的 3306 端口
2. 容器 B 里面，MySQL 监听的是容器 B 的 3306 端口
3. 这两个 3306 端口是**完全独立的**，因为它们在不同的 Network Namespace 里
4. 从宿主机访问时，通过端口映射：宿主机的 3307 映射到容器 A 的 3306，宿主机的 3308 映射到容器 B 的 3306

### 你可以把它理解成

- 传统方式：你只有一个厨房，做川菜和做粤菜都在这个厨房，调料容易混
- Docker 方式：你有多个独立的厨房，每个厨房只做一种菜系，互不干扰

---

## 3. 可移植性：一次构建，到处运行

### 这句话是什么意思

- Linux 服务器上跑
- 云服务器上跑
- 别人的电脑上跑
- 甚至 Mac 电脑上跑

"一次构建，到处运行"是 Docker 最核心的卖点之一。

它的意思是：

你在你 Windows 电脑上构建好的 Docker 镜像，可以直接拿到：

而且运行效果完全一样。

### 为什么能做到这一点？深入分析原理

```python
# Windows 上的路径
file_path = "C:\\Users\\YourName\\project\\data.txt"

要理解这个问题，我们需要先理解：**为什么传统方式做不到"一次构建，到处运行"？**

#### 传统方式的困境

假设你在 Windows 上开发了一个 Python 项目，现在要部署到 Linux 服务器上。

你会遇到这些问题：

**1. 路径问题**

# Linux 上的路径
file_path = "/home/yourname/project/data.txt"
```

- Windows 用 `\r\n` 作为换行符
- Linux 用 `\n` 作为换行符

- `psycopg2` 依赖 PostgreSQL 的客户端库
- `Pillow` 依赖图像处理库

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker 镜像                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  你的应用代码                                        │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Python 包（pip install 安装的）                     │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Python 解释器                                       │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  系统库（apt-get install 安装的）                    │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  精简的 Linux 系统（Debian/Ubuntu/Alpine）           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────┐
│              Linux 宿主机                    │
│  ┌───────────┐  ┌───────────┐  ┌──────────┐│
│  │ 容器 A    │  │ 容器 B    │  │ 容器 C   ││
│  │ (Linux)   │  │ (Linux)   │  │ (Linux)  ││
│  └───────────┘  └───────────┘  └──────────┘│
│                    │                        │
│                    ▼                        │
│            Linux 内核（共享）                │
└─────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                     Windows 宿主机                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              WSL2 / Hyper-V 虚拟机                   │   │
│  │  ┌───────────┐  ┌───────────┐  ┌──────────┐        │   │
│  │  │ 容器 A    │  │ 容器 B    │  │ 容器 C   │        │   │
│  │  │ (Linux)   │  │ (Linux)   │  │ (Linux)  │        │   │
│  │  └───────────┘  └───────────┘  └──────────┘        │   │
│  │                      │                              │   │
│  │                      ▼                              │   │
│  │              Linux 内核（在虚拟机里）                │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│                   Windows 内核                              │
└─────────────────────────────────────────────────────────────┘
```

- 镜像里包含的是 Linux 环境
- 不管你在什么系统上构建，构建出来的都是同样的 Linux 镜像
- 运行时，容器都是运行在 Linux 内核上的

路径格式完全不同，你的代码可能要改。

**2. 换行符问题**

这可能导致一些奇怪的问题，比如脚本在 Linux 上执行失败。

**3. 文件权限问题**

Linux 有严格的文件权限控制，Windows 没有。你的程序可能在 Linux 上因为权限问题无法读写文件。

**4. 系统库问题**

某些 Python 包依赖系统库，比如：

这些系统库在 Windows 和 Linux 上的安装方式、版本都可能不同。

#### Docker 为什么能解决这些问题？

**关键理解：** Docker 镜像里包含了完整的运行环境，包括操作系统。

**这意味着：**

1. 你的代码运行在镜像里的 Linux 系统上，不是运行在宿主机上
2. 不管宿主机是 Windows、Linux 还是 macOS，你的代码看到的都是同一个 Linux 环境
3. 所有的路径、换行符、权限、系统库都是一致的

#### Docker 在不同操作系统上是怎么工作的？

**在 Linux 上：**

Docker 直接使用 Linux 内核的功能（Namespace、Cgroups），容器进程就是普通的 Linux 进程。

**在 Windows 上：**

Windows 本身不支持 Linux 容器，所以 Docker Desktop 做了一些"魔法"：

1. 安装了一个轻量级的 Linux 虚拟机（通过 WSL2 或 Hyper-V）
2. 在这个 Linux 虚拟机里运行 Docker
3. Windows 和这个 Linux 虚拟机之间通过特殊的文件共享和网络桥接来通信

**在 macOS 上：**

和 Windows 类似，macOS 也不原生支持 Linux 容器，Docker Desktop 也是通过一个轻量级 Linux 虚拟机来运行的。

**这就是为什么：**

你在 Windows 上构建的镜像，可以直接在 Linux 服务器上运行，因为：

### 你可以把它理解成

总结：Docker 管运行环境和进程隔离，Nginx 管网络入口和请求转发。`docker run nginx` 是用 Docker 启动一个运行 Nginx 的容器；它本身不是负载均衡，只有容器里的 Nginx 进程按配置转发请求时，才承担反向代理或负载均衡角色。

- 传统方式：你搬家的时候，要把家具一件件拆开，搬到新家再一件件装回去，可能还装不回去
- Docker 方式：你搬家的时候，直接把整个房子搬过去，房子里的东西都不用动

```bash
docker run -d -p 8080:80 nginx
```

```text
Docker CLI 把请求发给 Docker daemon
  ↓
daemon 检查本地是否有 nginx 镜像
  ↓
没有则从镜像仓库拉取
  ↓
基于镜像创建容器可写层
  ↓
创建 namespace，隔离进程、网络、文件系统等视图
  ↓
设置 cgroup，限制 CPU、内存等资源
  ↓
创建容器网络，并把宿主机 8080 端口映射到容器 80 端口
  ↓
启动容器内的 nginx 主进程
```

```text
浏览器访问宿主机:8080
  ↓
宿主机 Docker 网络规则转发
  ↓
容器网络命名空间里的 80 端口
  ↓
Nginx 进程接收请求
```

```nginx
server {
    listen 80;

```text
客户端
  ↓ HTTP
Nginx
  ↓ 转发 HTTP
后端服务 FastAPI / Java / Node.js
```

```nginx
upstream backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

| 工具   | 解决的问题                                                   |
| ------ | ------------------------------------------------------------ |
| Docker | 把应用、运行时、依赖和启动命令打包成可重复运行的容器         |
| Nginx  | 接收网络请求，做静态文件服务、反向代理、负载均衡、TLS 终止等 |

#### Docker 和 Nginx 分别解决什么问题？`docker run nginx` 到底做了什么？

**问题**：Docker 是什么？Nginx 是什么？Docker 是负载均衡器吗？Nginx 如何做反向代理和负载均衡？

**回答**：

Docker 和 Nginx 解决的是不同层面的问题。

Docker 不是负载均衡器。它负责“把进程放到隔离环境里运行”。Nginx 才常用于“把外部请求转发给后端服务”。

执行：

大致发生了这些事：

端口映射的含义是：

Nginx 做反向代理时，客户端只知道 Nginx 地址，不直接知道后端服务地址：

    location /api/ {
        proxy_pass http://127.0.0.1:8000/;
    }
}
```

链路是：

Nginx 做负载均衡时，会在多个后端实例之间选择一个：

server {
    listen 80;

    location /api/ {
        proxy_pass http://backend;
    }
}
```

这时 Nginx 可以按轮询、权重、最少连接等策略把请求分散到多个后端进程。

### 9.5 云服务器部署

总结：部署不是简单“把代码放到服务器”。它包括运行环境、进程生命周期、网络入口、安全边界、域名解析和 HTTPS。代码只是其中一层，真正的线上服务是这一整条链路稳定协作的结果。

为什么通常不直接让后端服务暴露公网端口？

部署的本质是：**让你的程序在一台公网可访问的服务器上长期运行，并让用户能通过稳定域名安全访问它**。

```text
本地代码
  ↓
构建或上传到云服务器
  ↓
安装运行时和依赖
  ↓
配置环境变量
  ↓
启动后端进程
  ↓
用进程守护保证异常退出后可恢复
  ↓
Nginx 监听 80/443 并反向代理到后端端口
  ↓
域名 DNS 指向服务器公网 IP
  ↓
HTTPS 证书保证加密访问
```

```text
后端服务通常只监听 127.0.0.1:8000
Nginx 监听公网 80/443
Nginx 校验请求、处理 TLS、转发到后端
```

```nginx
server {
    listen 80;
    server_name api.example.com;

```text
域名是否解析到正确 IP
  ↓
安全组是否开放 80/443
  ↓
Nginx 是否在监听端口
  ↓
Nginx 是否能连到后端 127.0.0.1:8000
  ↓
后端进程是否存活
  ↓
环境变量和数据库连接是否正确
  ↓
应用日志是否有异常
```

| 好处     | 说明                                     |
| -------- | ---------------------------------------- |
| 统一入口 | 多个服务可以挂在同一个域名下             |
| TLS 终止 | HTTPS 证书由 Nginx 管理，后端只处理 HTTP |
| 反向代理 | 隐藏后端真实端口和进程结构               |
| 静态资源 | Nginx 可以直接返回静态文件               |
| 负载均衡 | 多个后端实例可以由 Nginx 分发请求        |

#### 把项目部署到云服务器是什么意思？

**问题**：从本地代码到线上服务，服务器、端口、安全组、环境变量、进程守护、Nginx、域名、HTTPS 分别在链路里做什么？

**回答**：

完整链路可以拆成：

这样做有几个好处：

一个典型 Nginx 配置如下：

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

线上服务出问题时，应按链路排查：

## 4. 版本控制和回滚：出问题可以立刻恢复

### Docker 镜像可以打标签

- `myapp:v1.0`
- `myapp:v1.1`
- `myapp:v1.2`
- `myapp:latest`

每次构建镜像的时候，你可以给它打一个标签，比如：

### 这有什么用

```powershell
docker compose down
docker tag myapp:v1.1 myapp:latest
docker compose up -d
```

假设你现在的生产环境跑的是 `myapp:v1.2`。

突然你发现 v1.2 有一个严重的 bug。

你可以立刻回滚到上一个版本：

几秒钟就回滚完成。

### 为什么能做到这么快？原理是什么？

#### 为什么回滚这么快？

```
┌─────────────────────────────────────┐
│  第 4 层：应用代码 v1.2              │  ← 只读
├─────────────────────────────────────┤
│  第 3 层：应用代码 v1.1              │  ← 只读
├─────────────────────────────────────┤
│  第 2 层：Python 包                  │  ← 只读
├─────────────────────────────────────┤
│  第 1 层：Python 解释器 + Linux      │  ← 只读
└─────────────────────────────────────┘
```

- 每一层都是独立的、只读的
- 多个镜像可以共享相同的层
- 标签只是指向某个镜像的"指针"

```
┌─────────────────────────────────────┐
│  第 3 层：应用代码 v1.1              │
├─────────────────────────────────────┤
│  第 2 层：Python 包                  │
├─────────────────────────────────────┤
│  第 1 层：Python 解释器 + Linux      │
└─────────────────────────────────────┘
```

```
┌─────────────────────────────────────┐
│  第 4 层：应用代码 v1.2              │  ← 只有这一层是新的
├─────────────────────────────────────┤
│  第 3 层：应用代码 v1.1              │  ← 和 v1.1 共享
├─────────────────────────────────────┤
│  第 2 层：Python 包                  │  ← 和 v1.1 共享
├─────────────────────────────────────┤
│  第 1 层：Python 解释器 + Linux      │  ← 和 v1.1 共享
└─────────────────────────────────────┘
```

```
回滚前：
latest → v1.2 → [第4层 + 第3层 + 第2层 + 第1层]

- 不需要重新下载镜像
- 不需要重新构建
- 只需要改变标签的指向

要理解这个问题，我们需要理解 **Docker 镜像的存储原理**。

#### 镜像是怎么存储的？

Docker 镜像采用 **分层存储（Layered Storage）** 的方式。

每一层都是只读的，所有层叠加在一起，就构成了完整的文件系统。

**关键理解：**

假设你有两个版本的镜像：

**v1.1 镜像：**

**v1.2 镜像：**

**当你回滚时：**

你只需要改变标签指向的镜像，不需要重新下载或构建任何东西。

回滚后：
latest → v1.1 → [第3层 + 第2层 + 第1层]
```

**这就是为什么回滚只需要几秒钟：**

#### 传统方式为什么慢？

传统方式下，回滚通常需要：

1. 找到旧版本的代码
2. 重新部署（可能需要重新编译、重新安装依赖）
3. 重启服务

这个过程可能需要几十分钟。

### 你可以把它理解成

- 传统方式：你写作业，每次修改都覆盖原来的内容，想找回旧版本只能重写
- Docker 方式：你写作业，每次修改都保存一个新文件，想找回旧版本直接打开旧文件

---

## 5. 快速启动：秒级启动 vs 分钟级启动

### Docker 容器启动有多快

```powershell
docker compose up -d
```

Docker 容器启动通常是秒级的。

你执行：

几秒钟后，所有服务就启动完成了。

### 为什么这么快？深入分析原理

```
第 1 步：加载虚拟机的配置文件
         ↓
第 2 步：分配内存（比如分配 4GB）
         ↓
第 3 步：创建虚拟硬盘、虚拟网卡等虚拟硬件
         ↓
第 4 步：加载虚拟机的 BIOS
         ↓
第 5 步：启动虚拟机的操作系统
         ↓
第 6 步：操作系统初始化（加载驱动、启动系统服务）
         ↓
第 7 步：启动你的应用程序
```

- 第 2-4 步：需要模拟硬件，这本身就很耗时
- 第 5-6 步：需要启动一个完整的操作系统，就像你开机一样

```
第 1 步：读取镜像配置
         ↓
第 2 步：创建容器的文件系统（使用 UnionFS，只是创建一个可写层）
         ↓
第 3 步：创建 Namespace（隔离环境）
         ↓
第 4 步：设置 Cgroups（资源限制）
         ↓
第 5 步：启动你的应用程序
```

- 不需要模拟硬件
- 不需要启动操作系统（直接使用宿主机的内核）
- 不需要初始化操作系统（系统已经在运行了）

```
┌─────────────────────────────────────────────┐
│              虚拟机                          │
│  ┌─────────────────────────────────────┐   │
│  │  你的应用程序                        │   │
│  ├─────────────────────────────────────┤   │
│  │  Guest OS（需要启动的操作系统）       │  ← 需要启动这个！
│  └─────────────────────────────────────┘   │
│                    │                        │
│                    ▼                        │
│            Hypervisor（虚拟化层）            │
└─────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────┐
│              宿主机                          │
│  ┌─────────────────────────────────────┐   │
│  │  你的应用程序                        │   │
│  └─────────────────────────────────────┘   │
│                    │                        │
│                    ▼                        │
│            Host OS（已经在运行的操作系统）    │  ← 不需要启动！
└─────────────────────────────────────────────┘
```

- 你要盖房子（创建虚拟硬件）
- 你要装修（安装操作系统）
- 你要入住（启动应用程序）
- 整个过程需要几个月

- 房子已经盖好了（宿主机操作系统已经在运行）
- 你只需要在这个房子里隔出一个房间（创建 Namespace）
- 然后搬进去住（启动应用程序）
- 整个过程只需要几分钟

要理解这个问题，我们需要对比 **虚拟机** 和 **Docker 容器** 的启动过程。

#### 虚拟机启动过程

当你启动一个虚拟机时，发生了什么？

**这个过程为什么慢？**

通常需要几分钟。

#### Docker 容器启动过程

当你启动一个 Docker 容器时，发生了什么？

**这个过程为什么快？**

通常只需要几秒甚至不到一秒。

#### 关键区别：Docker 容器不需要启动操作系统

这是最核心的区别。

**虚拟机：**

**Docker 容器：**

**为什么 Docker 容器不需要启动操作系统？**

因为 Docker 容器**共享宿主机的操作系统内核**。

当你启动一个容器时，你只是：

1. 创建了一个隔离的环境（Namespace）
2. 在这个隔离环境里启动了一个进程

这个进程直接运行在宿主机的内核上，不需要额外的操作系统。

#### 一个形象的类比

**虚拟机就像"一栋独立的房子"：**

**Docker 容器就像"一个房间"：**

### 这有什么实际意义

- 你需要从 1 个实例扩展到 10 个实例
- 用虚拟机：你要等几分钟让 10 个虚拟机启动
- 用 Docker：几秒钟就能启动 10 个容器

- 用 Docker：用户几乎无感知
- 用虚拟机：用户可能已经等不及走了

假设你的服务需要扩容：

在流量高峰来临时，这个速度差异可能意味着：

---

## 6. 资源利用率：比虚拟机轻量得多

### 先理解虚拟机和 Docker 的区别

- 分配独立的内存（比如 2GB）
- 分配独立的 CPU 核心
- 安装完整的操作系统
- 运行完整的系统进程

- 你开 4 个虚拟机，每个分配 4GB
- 这 4 个虚拟机本身就要占用 16GB 内存
- 其中很多内存是浪费在操作系统上的

- 不需要独立的操作系统
- 共享宿主机的内核
- 只需要运行应用程序本身需要的资源

- 你可能可以跑几十个甚至上百个容器
- 因为容器本身几乎不占用额外资源

**虚拟机：**

每个虚拟机都要：

假设你的物理服务器有 16GB 内存：

**Docker 容器：**

每个容器：

同样的 16GB 内存：

### 为什么会有这种差异？深入分析原理

- 内存（通常至少 512MB）
- CPU 时间（运行系统服务）
- 磁盘空间（通常至少几 GB）

```
┌─────────────────────────────────────────────────────────────┐
│                     物理服务器（16GB 内存）                   │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  虚拟机 1    │  │  虚拟机 2    │  │  虚拟机 3    │         │
│  │             │  │             │  │             │         │
│  │ 分配 4GB    │  │ 分配 4GB    │  │ 分配 4GB    │         │
│  │             │  │             │  │             │         │
│  │ Guest OS    │  │ Guest OS    │  │ Guest OS    │         │
│  │ (约 1GB)    │  │ (约 1GB)    │  │ (约 1GB)    │         │
│  │             │  │             │  │             │         │
│  │ 应用程序    │  │ 应用程序    │  │ 应用程序    │         │
│  │ (约 100MB)  │  │ (约 100MB)  │  │ (约 100MB)  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  总共使用：12GB（其中约 3GB 是操作系统开销）                   │
└─────────────────────────────────────────────────────────────┘
```

- 限制进程组可以使用的资源（CPU、内存、磁盘 I/O 等）
- 统计进程组使用的资源
- 隔离进程组的资源使用

```powershell
# 限制容器最多使用 512MB 内存
docker run --memory=512m myimage

#### 虚拟机的资源开销

当你创建一个虚拟机时，你需要为它分配：

**1. 内存**

每个虚拟机都需要独立的内存。即使虚拟机里的应用程序只用了 100MB，你分配给它的 4GB 也被占用了。

**2. CPU**

每个虚拟机都需要虚拟 CPU。虚拟机管理程序需要在多个虚拟机之间调度 CPU 时间。

**3. 操作系统开销**

每个虚拟机都运行一个完整的操作系统，这个操作系统本身就需要：

#### Docker 容器的资源开销

Docker 容器使用 **Cgroups（Control Groups）** 来限制和隔离资源使用。

**Cgroups 是什么？**

Cgroups 是 Linux 内核提供的一种功能，它可以：

**Docker 怎么使用 Cgroups？**

当你启动一个容器时，Docker 会：

1. 创建一个 Cgroup
2. 把容器进程放进这个 Cgroup
3. 设置这个 Cgroup 的资源限制

# 限制容器最多使用 50% 的 CPU
docker run --cpus=0.5 myimage
```

#### 为什么容器能做到"按需使用"？

- 容器进程直接运行在宿主机内核上
- 容器不需要独立的操作系统
- 容器只使用应用程序实际需要的资源

```
┌─────────────────────────────────────────────────────────────┐
│                     物理服务器（16GB 内存）                   │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │ 容器 1  │ │ 容器 2  │ │ 容器 3  │ │ 容器 4  │ │ 容器 5  ││
│  │         │ │         │ │         │ │         │ │         ││
│  │ 约 100MB│ │ 约 100MB│ │ 约 100MB│ │ 约 100MB│ │ 约 100MB││
│  │         │ │         │ │         │ │         │ │         ││
│  │ 应用程序│ │ 应用程序│ │ 应用程序│ │ 应用程序│ │ 应用程序││
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘│
│                                                             │
│  ... 还可以继续添加更多容器 ...                               │
│                                                             │
│  Host OS（约 1GB）                                          │
│                                                             │
│  总共使用：约 1.5GB（只有应用程序和宿主机操作系统）             │
└─────────────────────────────────────────────────────────────┘
```

- 你分配 4GB 内存给虚拟机
- 即使虚拟机里的应用程序只用了 100MB
- 这 4GB 也被虚拟机"占用"了，其他虚拟机不能用

- 你限制容器最多使用 512MB 内存
- 如果容器里的应用程序只用了 100MB
- 剩下的 412MB 还可以被其他容器使用

**关键理解：**

**关键理解：** Cgroups 限制的是"最大可用资源"，不是"必须分配的资源"。

**虚拟机：**

**Docker 容器：**

### 你可以把它理解成

- 虚拟机：你租了一套房子，不管你住不住，房租都要付，而且一个人住很浪费
- Docker 容器：你住酒店，住多少天付多少钱，不住不付钱，而且可以很多人拼房

---

## 7. 微服务架构的基础设施

### 什么是微服务架构

- 用户服务：负责用户注册、登录
- 订单服务：负责下单、支付
- 商品服务：负责商品展示、库存
- 消息服务：负责发送通知

- 用不同的编程语言
- 用不同的数据库
- 独立部署、独立扩展

简单说，就是把一个大项目拆成多个小服务：

每个服务可以：

### 为什么微服务需要 Docker？

```
┌─────────────────────────────────────────────────────────────┐
│                      一个大项目                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  用户模块 + 订单模块 + 商品模块 + 消息模块            │   │
│  │                                                     │   │
│  │  全部在一个进程里运行                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   用户服务    │  │   订单服务    │  │   商品服务    │  │   消息服务    │
│              │  │              │  │              │  │              │
│  Python      │  │  Java        │  │  Go          │  │  Node.js     │
│  MySQL       │  │  PostgreSQL  │  │  MongoDB     │  │  Redis       │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

- 用户服务用 Python + MySQL
- 订单服务用 Java + PostgreSQL
- 商品服务用 Go + MongoDB

- 在服务器上安装 Python、Java、Go、Node.js
- 安装 MySQL、PostgreSQL、MongoDB、Redis
- 管理不同版本之间的兼容性

- 构建和打包
- 部署到服务器
- 配置和启动
- 监控和日志

- 手动启动更多的实例
- 配置负载均衡
- 管理多个实例的生命周期

```yaml
services:
  user-service:
    image: python:3.13
    # Python 环境

```dockerfile
# 用户服务的 Dockerfile
FROM python:3.13
COPY . /app
RUN pip install -r requirements.txt
CMD ["python", "main.py"]
```

```dockerfile
# 订单服务的 Dockerfile
FROM openjdk:21
COPY . /app
RUN ./mvnw package
CMD ["java", "-jar", "app.jar"]
```

```powershell
# 启动 3 个订单服务实例
docker compose up -d --scale order-service=3
```

#### 单体架构 vs 微服务架构

**单体架构：**

**微服务架构：**

#### 微服务面临的挑战

**挑战 1：环境多样性**

不同的服务可能需要不同的技术栈：

如果没有 Docker，你需要：

这是一场噩梦。

**挑战 2：部署复杂性**

每个服务都需要：

如果有 10 个服务，你就需要重复这些步骤 10 次。

**挑战 3：扩展困难**

如果某个服务的负载增加，你需要：

#### Docker 怎么解决这些问题

**1. 环境隔离**

每个微服务运行在自己的容器里，互不干扰：

  order-service:
    image: openjdk:21
    # Java 环境

  product-service:
    image: golang:1.21
    # Go 环境
```

**2. 标准化部署**

每个服务都用同样的方式打包成镜像：

虽然技术栈不同，但部署流程是一样的：

1. 写 Dockerfile
2. 构建镜像
3. 运行容器

**3. 轻松扩展**

如果订单服务的负载增加，你可以轻松扩展：

Docker 会自动创建 3 个订单服务容器，并配置负载均衡。

### 你可以把它理解成

- 单体架构：一个超级大的瑞士军刀，什么功能都有，但很重，坏了一个功能整个都不能用
- 微服务架构：一套专业的工具，每个工具只做一件事，但组合起来很强大，坏了一个工具不影响其他工具
- Docker：装这套工具的工具箱，让每个工具都能独立存放、独立使用

---

## 8. CI/CD 流水线的好帮手

### Docker 在 CI/CD 中的作用

- Python 项目
- Node.js 项目
- Go 项目
- Java 项目

- Python（可能还要多个版本）
- Node.js（可能还要多个版本）
- Go
- Java

- 某个 Python 包需要 Python 3.11
- 另一个项目需要 Python 3.13
- 某个 Node.js 项目需要 Node 18
- 另一个项目需要 Node 20

```yaml
# Python 项目的 CI 配置
build-python:
  image: python:3.13  # 使用 Python 3.13 镜像
  script:
    - pip install -r requirements.txt
    - pytest

#### 没有 Docker 时的 CI/CD 困境

假设你的 CI/CD 服务器需要构建多种语言的项目：

**问题来了：**

你需要在 CI 服务器上安装：

这些环境之间可能还有冲突：

管理这些环境是一场噩梦。

#### Docker 怎么解决这个问题

**关键思想：** 每个构建任务都在独立的容器里运行。

# Node.js 项目的 CI 配置
build-node:
  image: node:20  # 使用 Node 20 镜像
  script:
    - npm ci
    - npm test

# Go 项目的 CI 配置
build-go:
  image: golang:1.21  # 使用 Go 1.21 镜像
  script:
    - go mod download
    - go test ./...
```

构建完成后，可以直接打包成 Docker 镜像：

```yaml
# CI/CD 流程
stages:
  - test
  - build
  - deploy

**好处：**

1. CI 服务器只需要安装 Docker，不需要安装各种语言环境
2. 每个构建任务用对应的镜像，互不干扰
3. 可以轻松切换版本（只需要改镜像标签）

#### Docker 在部署阶段的作用

test:
  image: python:3.13
  script:
    - pytest

build:
  image: docker:latest
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA

deploy:
  script:
    - docker pull myapp:$CI_COMMIT_SHA
    - docker compose up -d
```

### 为什么这很重要

- 你得在 CI 服务器上安装各种语言的运行环境
- Python、Node、Go、Java……全都得装
- 版本冲突怎么办？

- CI 服务器只需要装 Docker
- 每个构建任务用对应的镜像
- 互不干扰

因为 CI/CD 流水线本身也需要一个运行环境。

如果没有 Docker：

有了 Docker：

---

## 9. 团队协作：新人入职不再花一天配环境

### 传统方式的新人入职

假设一个新同事入职，要参与你的项目：

1. 先安装 Python（版本要对）
2. 再安装 MySQL（版本要对）
3. 再安装 Redis（版本要对）
4. 再安装 RabbitMQ（版本要对）
5. 再配置环境变量
6. 再安装 Python 依赖
7. 再初始化数据库
8. 再配置各种连接

一天过去了，还没开始写代码。

### Docker 方式的新人入职

1. 安装 Docker Desktop
2. 克隆代码仓库
3. `docker compose up -d`
4. 开始写代码

半小时搞定。

### 为什么差距这么大？

```
新人 A：安装 Python → 安装 MySQL → 安装 Redis → 配置环境 → ...
新人 B：安装 Python → 安装 MySQL → 安装 Redis → 配置环境 → ...
新人 C：安装 Python → 安装 MySQL → 安装 Redis → 配置环境 → ...
```

```
新人 A：docker compose up -d → 开始工作
新人 B：docker compose up -d → 开始工作
新人 C：docker compose up -d → 开始工作
```

**传统方式：**

每个人都要自己搭建环境，就像每个人都要自己盖房子。

每个人都要重复同样的步骤，浪费时间，而且容易出错。

**Docker 方式：**

环境已经打包在镜像里，就像房子已经盖好了，直接入住。

### 你可以把它理解成

- 传统方式：新人入职，先让他自己组装办公桌、椅子、电脑，装好了才能开始工作
- Docker 方式：新人入职，直接给他一个已经布置好的工位，马上就能开始工作

---

## 10. 安全隔离：一个容器被黑，不影响其他容器

### 容器之间的隔离性

- 攻击者只能在这个容器里活动
- 不能直接访问其他容器
- 不能直接访问宿主机

Docker 容器之间是相互隔离的。

即使一个容器被攻击者入侵：

### 这有什么实际意义

- Web 前端容器
- API 后端容器
- 数据库容器

- 攻击者只能在前端容器里
- 不能直接访问数据库容器
- 你有时间发现和处理

假设你的架构是：

如果 Web 前端容器被攻击了：

### 为什么能做到隔离？原理是什么

- 如果内核有漏洞，攻击者可能从容器逃逸到宿主机
- 容器的隔离性不如虚拟机

| Namespace 类型    | 隔离的内容 | 安全意义                                            |
| ----------------- | ---------- | --------------------------------------------------- |
| PID Namespace     | 进程 ID    | 攻击者看不到其他容器的进程                          |
| Network Namespace | 网络       | 攻击者不能直接访问其他容器的网络                    |
| Mount Namespace   | 文件系统   | 攻击者不能直接访问其他容器的文件                    |
| User Namespace    | 用户       | 攻击者在容器里是 root，但在宿主机上可能只是普通用户 |

- 不要在容器里运行不可信的代码
- 使用非 root 用户运行容器
- 定期更新宿主机内核和 Docker

这又回到了 **Namespace** 和 **Cgroups**。

#### Namespace 隔离了什么？

#### 容器安全的局限性

**重要：** 容器隔离不是完全安全的。

容器共享宿主机的内核，所以：

**最佳实践：**

### 你可以把它理解成

- 传统部署：所有服务都在一个房间里，小偷进来可以随便翻
- Docker 部署：每个服务在独立的保险柜里，小偷打开一个，拿不到其他的

---

## 11. 开发测试环境快速搭建

### 你以前可能这样测试

- MySQL
- Redis
- RabbitMQ

- 在本机安装 MySQL
- 在本机安装 Redis
- 在本机安装 RabbitMQ

测试完之后，这些软件还留在你电脑上，占用资源。

假设你要测试一个功能，这个功能依赖：

你可能会：

### Docker 方式

```powershell
docker compose -f docker-compose.test.yml up -d
```

测试完之后：

```powershell
docker compose -f docker-compose.test.yml down -v
```

你可以用 Docker 快速启动一个测试环境：

所有容器和数据都删除了，你电脑又干净了。

### 为什么这很重要？

- MySQL
- Redis
- RabbitMQ
- Elasticsearch
- Kafka
- ...

- 占用磁盘空间
- 占用内存
- 可能和你的其他项目冲突
- 卸载起来很麻烦

- 不占用持久资源
- 不会和其他项目冲突
- 删除很干净

**传统方式：**

你为了测试一个功能，在电脑上装了一堆软件：

这些软件：

**Docker 方式：**

你只在需要的时候启动容器，不需要的时候删除容器：

### 你可以把它理解成

- 传统方式：你为了测试一个灯泡，把整个房子的电路都改装了，测试完还得改回来
- Docker 方式：你为了测试一个灯泡，临时接一个插座，测试完拔掉，什么痕迹都不留

---

## 12. 云原生的基础

### 什么是云原生

- 容器化
- 微服务
- DevOps
- 持续交付

云原生（Cloud Native）是一套方法论：

Docker 是云原生的基石。

### 为什么云厂商都支持 Docker

- Docker 容器可以在任何云上运行
- 不需要针对每个云做适配
- 迁移成本低

AWS、阿里云、腾讯云、Google Cloud……所有主流云厂商都支持 Docker。

因为：

### 这对你有什么意义

- 如果你的应用是 Docker 容器化的，迁移很简单
- 只需要把镜像拉到新云上运行
- 不需要重新适配环境

假设你现在用阿里云，以后想迁移到 AWS：

---

---

## 问题 5：Docker 的数据持久化是怎么做的？

### 这是一个考察你理解容器特性的题

### 你应该怎么回答

```
容器存在时：
┌─────────────────────────────────────┐
│  可写层：数据库文件、日志文件等       │  ← 容器的数据
├─────────────────────────────────────┤
│  镜像层（只读）                      │
└─────────────────────────────────────┘

```yaml
services:
  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql

```
使用命名卷后：

```yaml
services:
  backend:
    image: mybackend
    volumes:
      - ./data:/app/data
```

```
使用绑定挂载后：

```yaml
services:
  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql

```powershell
docker compose down
```

```yaml
services:
  backend:
    build: .
    volumes:
      - .:/app
```

```powershell
# 查看所有数据卷
docker volume ls

| 特性     | 命名卷                                        | 绑定挂载             |
| -------- | --------------------------------------------- | -------------------- |
| 管理方式 | Docker 管理                                   | 用户管理             |
| 存储位置 | Docker 默认目录（如 /var/lib/docker/volumes） | 用户指定目录         |
| 可移植性 | 好（不依赖宿主机路径）                        | 差（依赖宿主机路径） |
| 适用场景 | 生产环境                                      | 开发环境             |
| 性能     | 更好（尤其在 Mac/Windows）                    | 一般                 |

#### 1. 先理解问题：为什么需要数据持久化

Docker 容器有一个特点：

**容器删除后，容器里的所有数据都会消失。**

**为什么？**

因为容器的数据存储在容器的可写层，当容器删除时，可写层也被删除了。

容器删除后：
┌─────────────────────────────────────┐
│  镜像层（只读）                      │  ← 可写层没了！
└─────────────────────────────────────┘
```

这对于数据库来说是不可接受的。

#### 2. Docker 提供的解决方案：数据卷（Volume）

Docker 提供了两种主要的数据持久化方式：

**方式一：命名卷（Named Volume）**

volumes:
  mysql-data:
```

**原理：**

命名卷是 Docker 管理的存储区域，独立于容器存在。

┌─────────────────────────────────────┐
│  命名卷 mysql-data                  │  ← 独立存储，不在容器里
│  （存储在 Docker 的数据目录）        │
└─────────────────────────────────────┘
           ↓ 挂载
┌─────────────────────────────────────┐
│  容器                                │
│  /var/lib/mysql → 指向命名卷         │
└─────────────────────────────────────┘

容器删除后：
┌─────────────────────────────────────┐
│  命名卷 mysql-data                  │  ← 数据还在！
└─────────────────────────────────────┘
```

**方式二：绑定挂载（Bind Mount）**

**原理：**

绑定挂载直接把宿主机的目录映射到容器里。

┌─────────────────────────────────────┐
│  宿主机                              │
│  ./data 目录                         │  ← 宿主机的目录
└─────────────────────────────────────┘
           ↓ 映射
┌─────────────────────────────────────┐
│  容器                                │
│  /app/data → 宿主机的 ./data         │
└─────────────────────────────────────┘
```

#### 3. 两种方式的对比

#### 4. 实际例子

**MySQL 数据持久化：**

volumes:
  mysql-data:
```

这样即使你执行：

MySQL 的数据也不会丢失。

**开发环境代码同步：**

这样你在宿主机修改代码，容器里立刻就能看到变化，方便开发调试。

#### 5. 数据卷的常用命令

# 查看某个数据卷的详情
docker volume inspect mysql-data

# 删除某个数据卷
docker volume rm mysql-data

# 删除所有未被使用的数据卷
docker volume prune
```

## 问题 6：Docker 的网络模式有哪些？

```
┌─────────────────────────────────────────────┐
│                  宿主机                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ 容器 A  │  │ 容器 B  │  │ 容器 C  │     │
│  │         │  │         │  │         │     │
│  │ IP:     │  │ IP:     │  │ IP:     │     │
│  │ 172.17. │  │ 172.17. │  │ 172.17. │     │
│  │ 0.2     │  │ 0.3     │  │ 0.4     │     │
│  └────┬────┘  └────┬────┘  └────┬────┘     │
│       │            │            │           │
│       └────────────┼────────────┘           │
│                    │                        │
│            ┌───────┴───────┐               │
│            │  docker0 网桥  │               │
│            │  172.17.0.1    │               │
│            └───────────────┘               │
└─────────────────────────────────────────────┘
```

- 每个容器有自己的 IP
- 容器之间可以通过 IP 互相访问
- 需要端口映射才能从外部访问

```
┌─────────────────────────────────────────────┐
│                  宿主机                      │
│  IP: 192.168.1.100                          │
│  ┌─────────────────────────────────────┐   │
│  │ 容器（直接使用宿主机的网络）          │   │
│  │ IP: 192.168.1.100（和宿主机一样）     │   │
│  │ 端口: 直接使用宿主机的端口            │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

- 容器直接使用宿主机的网络
- 没有网络隔离
- 性能最好
- 但端口容易冲突

- 容器没有网络
- 完全隔离
- 适合安全性要求极高的场景

- 用于跨主机的容器通信
- 通常配合 Docker Swarm 或 Kubernetes 使用

```yaml
services:
  backend:
    image: mybackend

```python
# 在 backend 容器里，可以这样连接 MySQL
# 注意：用的是服务名 mysql，不是 IP
import pymysql
conn = pymysql.connect(host='mysql', port=3306)
```

```yaml
services:
  backend:
    image: mybackend
    networks:
      - frontend
      - backend

- `nginx` 只能访问 `backend`
- `backend` 可以访问 `mysql` 和 `nginx`
- `mysql` 只能被 `backend` 访问

#### 1. Docker 网络的作用

Docker 网络解决的是：

**容器之间怎么通信？**
**容器和宿主机怎么通信？**
**容器和外部网络怎么通信？**

#### 2. Docker 的几种网络模式

**Bridge 模式（默认）：**

**原理：**

Bridge 模式使用 Linux 的 `bridge` 网络设备。

1. Docker 创建一个虚拟网桥 `docker0`
2. 每个容器创建一对虚拟网卡（veth pair）
3. 一端放在容器里，一端连接到网桥
4. 容器之间可以通过网桥通信

**特点：**

**Host 模式：**

**原理：**

Host 模式下，容器共享宿主机的 Network Namespace。

容器没有自己的网络设备，直接使用宿主机的网络。

**特点：**

**None 模式：**

**原理：**

None 模式下，容器只有 loopback 网络设备，没有其他网络。

**特点：**

**Overlay 模式：**

**原理：**

Overlay 模式使用 VXLAN 技术在多台主机之间创建虚拟网络。

**特点：**

#### 3. Docker Compose 中的网络

在 Docker Compose 中，默认会创建一个网络：

  mysql:
    image: mysql:8.0
```

这两个服务会自动加入同一个网络，可以通过服务名互相访问：

**原理：**

Docker Compose 会：

1. 创建一个默认网络（如 `myproject_default`）
2. 所有服务都加入这个网络
3. Docker 内置 DNS 服务器会解析服务名到容器 IP

#### 4. 自定义网络

你也可以自己定义网络：

  mysql:
    image: mysql:8.0
    networks:
      - backend

  nginx:
    image: nginx
    networks:
      - frontend

networks:
  frontend:
  backend:
```

这样：

### 面试加分回答

> "在实际项目中，我通常会使用自定义网络来控制服务之间的访问权限。比如数据库只允许后端访问，不允许前端直接访问。这样可以提高安全性。另外，在微服务架构中，合理的网络划分也能帮助隔离故障域。"

你还可以补充：

---

## 问题 7：什么是多阶段构建？为什么要用它？

### 这是一个考察你镜像优化能力的题

### 你应该怎么回答

```dockerfile
FROM golang:1.21

- `golang:1.21` 基础镜像大约 1.5GB
- 加上你的代码和编译产物
- 最终镜像可能有 1.5GB+

- 镜像太大
- 包含了不必要的编译工具
- 下载和部署都慢

```dockerfile
# 第一阶段：构建
FROM golang:1.21 AS builder

#### 1. 先看问题：传统构建有什么问题

假设你有一个 Go 项目，传统的 Dockerfile 可能是这样：

WORKDIR /app

COPY . .

RUN go build -o myapp

CMD ["./myapp"]
```

这个镜像有多大？

但实际上，你的可执行文件可能只有几十 MB。

**问题：**

#### 2. 多阶段构建的解决方案

WORKDIR /app

COPY . .

RUN go build -o myapp

# 第二阶段：运行
FROM alpine:3.19

- 使用不同的基础镜像
- 执行不同的命令
- 从其他阶段复制文件

```
第一阶段（builder）：
┌─────────────────────────────────────┐
│  golang:1.21 镜像（约 1.5GB）        │
│  + 源代码                           │
│  + 编译后的可执行文件                │
└─────────────────────────────────────┘

- `alpine:3.19` 约 5MB
- 加上你的可执行文件几十 MB
- 总共可能只有 50MB 左右

```dockerfile
# 第一阶段：构建
FROM node:20-alpine AS builder

WORKDIR /app

COPY --from=builder /app/myapp .

CMD ["./myapp"]
```

**原理：**

多阶段构建允许你在同一个 Dockerfile 中定义多个构建阶段。

每个阶段都是独立的，可以：

第二阶段（最终镜像）：
┌─────────────────────────────────────┐
│  alpine:3.19 镜像（约 5MB）          │
│  + 从第一阶段复制的可执行文件        │
└─────────────────────────────────────┘

最终镜像大小：约 50MB
```

**发生了什么：**

1. 第一阶段：用 `golang:1.21` 镜像编译代码
2. 第二阶段：用 `alpine:3.19`（只有 5MB）作为运行环境
3. 从第一阶段复制编译好的可执行文件到第二阶段

**最终镜像大小：**

从 1.5GB 减少到 50MB，差距巨大。

#### 3. 前端项目的多阶段构建

前端项目也很适合多阶段构建：

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# 第二阶段：运行
FROM nginx:1.27-alpine

- 最终镜像不包含 Node.js
- 最终镜像不包含源代码
- 最终镜像不包含 node_modules
- 只有构建后的静态文件和 Nginx

```dockerfile
FROM <镜像> AS <阶段名>

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**发生了什么：**

1. 第一阶段：用 Node 镜像构建前端
2. 第二阶段：用 Nginx 镜像提供静态文件服务
3. 从第一阶段复制 `dist` 目录到第二阶段

**好处：**

#### 4. 多阶段构建的核心语法

# ... 构建过程 ...

- `AS <阶段名>` 给阶段命名
- `COPY --from=<阶段名>` 从指定阶段复制文件

FROM <镜像>

COPY --from=<阶段名> <源路径> <目标路径>
```

**关键点：**

### 面试加分回答

> "多阶段构建不仅能减小镜像体积，还能提高安全性。因为最终镜像不包含编译工具和源代码，攻击者即使进入容器，也拿不到源代码。在实际项目中，我会把所有需要编译的项目（Go、Java、前端）都用多阶段构建来优化。"

你还可以补充：

---

## 问题 8：怎么优化 Docker 镜像大小？

### 这是一个考察你实战经验的题

### 你应该怎么回答

创建 `.dockerignore` 文件，排除不需要的文件：

```dockerfile
FROM python:3.13  # 完整版，约 1GB
```

```dockerfile
FROM python:3.13-slim  # 精简版，约 150MB
```

```dockerfile
FROM python:3.13-alpine  # Alpine 版，约 50MB
```

- `python:3.13`：包含完整的 Debian 系统，很多不必要的包
- `python:3.13-slim`：精简版，只包含运行 Python 必需的包
- `python:3.13-alpine`：基于 Alpine Linux，使用 musl libc，体积最小

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean
```

```dockerfile
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

```
不好的写法：
┌─────────────────────────────────────┐
│  第 4 层：apt-get clean 的结果       │
├─────────────────────────────────────┤
│  第 3 层：apt-get install git 的结果 │
├─────────────────────────────────────┤
│  第 2 层：apt-get install curl 的结果│
├─────────────────────────────────────┤
│  第 1 层：apt-get update 的结果      │
└─────────────────────────────────────┘

```text
.git
.venv
__pycache__
*.pyc
node_modules
```

- .git 目录（可能几百 MB）
- .venv 目录（虚拟环境）
- node_modules 目录（可能几百 MB）

- 构建上下文很大
- 构建速度慢
- 镜像体积大

```dockerfile
RUN myapp > output.log
```

```dockerfile
RUN myapp > /dev/null
```

```dockerfile
RUN pip install --no-cache-dir mypackage
```

```dockerfile
# 先复制依赖文件
COPY requirements.txt .
RUN pip install -r requirements.txt

我会从以下几个方面来优化，并解释每个方法的原理：

#### 1. 选择合适的基础镜像

**不好的选择：**

**好的选择：**

**更好的选择（如果不需要 glibc）：**

**原理：**

**注意：** Alpine 使用 musl libc 而不是 glibc，某些 Python 包可能不兼容，需要测试。

#### 2. 使用多阶段构建

前面已经讲过，这里不重复。

#### 3. 合并 RUN 命令

**不好的写法：**

这样会创建 4 个镜像层。

**好的写法：**

这样只创建 1 个镜像层，而且清理了缓存。

**原理：**

每个 RUN 命令都会创建一个新的镜像层。

好的写法：
┌─────────────────────────────────────┐
│  第 1 层：所有命令的结果              │
└─────────────────────────────────────┘
```

#### 4. 使用 .dockerignore

**原理：**

.dockerignore 的作用类似于 .gitignore。

当执行 `docker build` 时，Docker 会把当前目录的所有文件发送给 Docker daemon（构建上下文）。

如果没有 .dockerignore，很多不必要的文件也会被发送：

这会导致：

#### 5. 不要在镜像里存日志和临时文件

**不好的写法：**

**好的写法：**

或者把日志挂载到数据卷。

#### 6. 使用 --no-cache-dir 安装 Python 包

**原理：**

pip 默认会缓存下载的包，这些缓存会占用空间。

`--no-cache-dir` 告诉 pip 不要缓存，减小镜像体积。

#### 7. 利用构建缓存

Docker 会缓存每一层的构建结果。

把不常变化的放在前面：

# 再复制代码（代码经常变）
COPY . .
```

```
第一次构建：
COPY requirements.txt . → 缓存 miss，构建
RUN pip install ...     → 缓存 miss，构建
COPY . .                → 缓存 miss，构建

**原理：**

Docker 构建镜像时，会检查每一层是否可以使用缓存。

如果某一层的变化了，该层及其后面的所有层都需要重新构建。

第二次构建（只改了代码）：
COPY requirements.txt . → 缓存 hit，跳过
RUN pip install ...     → 缓存 hit，跳过
COPY . .                → 缓存 miss，构建（代码变了）
```

### 面试加分回答

> "在实际项目中，我会把这些优化方法结合起来使用。比如，我之前把一个 1.2GB 的 Python 镜像优化到了 150MB，主要通过：使用 slim 基础镜像、合并 RUN 命令、清理缓存、使用 .dockerignore。镜像变小后，CI/CD 流水线更快了，部署也更迅速了。"

你还可以补充：

---

## 问题 9：怎么排查 Docker 容器的问题？

### 这是一个考察你实战能力的题

### 你应该怎么回答

```powershell
docker ps -a
```

- 容器是不是在运行
- 如果退出了，退出码是什么

```powershell
docker logs <容器名>

| 退出码 | 含义            | 可能原因              |
| ------ | --------------- | --------------------- |
| 0      | 正常退出        | 程序正常结束          |
| 1      | 应用错误        | 程序抛出异常          |
| 137    | 被 SIGKILL 杀死 | 通常是内存不足（OOM） |
| 139    | 段错误          | 程序访问了非法内存    |

我会按以下步骤排查，并解释每个步骤的原理：

#### 1. 查看容器状态

**看什么：**

**退出码的含义：**

**原理：**

当容器退出时，容器内主进程的退出码会被记录下来。

退出码可以帮助我们快速定位问题类型。

#### 2. 查看容器日志

# 实时查看
docker logs -f <容器名>

# 看最后 100 行
docker logs --tail 100 <容器名>
```

```powershell
docker exec -it <容器名> sh
```

- `-i`：保持标准输入打开
- `-t`：分配一个伪终端

- 检查文件是否存在
- 检查环境变量是否正确
- 检查网络连接
- 手动执行命令测试

```powershell
# 检查环境变量
env | grep MYSQL

**原理：**

Docker 会捕获容器内进程的标准输出和标准错误，存储在日志文件中。

默认情况下，日志存储在 `/var/lib/docker/containers/<容器ID>/<容器ID>-json.log`。

#### 3. 进入容器内部排查

**原理：**

`docker exec` 会在运行的容器里启动一个新进程。

`-it` 参数：

这样你就可以像 SSH 一样在容器里执行命令。

进去后可以：

# 检查网络连接
ping mysql

# 检查端口
nc -zv mysql 3306

# 检查文件
ls -la /app
```

```powershell
docker stats <容器名>
```

- CPU 使用率是否异常
- 内存使用是否接近限制

```powershell
# 查看容器的网络配置
docker inspect <容器名> | grep -A 20 "NetworkSettings"

#### 4. 检查容器资源使用

**看什么：**

**原理：**

`docker stats` 读取 Cgroups 的统计数据，显示容器的资源使用情况。

#### 5. 检查容器网络

# 查看容器的 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <容器名>
```

#### 6. 常见问题和解决思路

**问题：容器启动后立刻退出**

排查步骤：

1. 看日志：`docker logs <容器名>`
2. 检查启动命令是否正确
3. 检查配置文件是否存在
4. 检查依赖服务是否就绪

**问题：容器内连不上数据库**

排查步骤：

1. 进入容器：`docker exec -it <容器名> sh`
2. 检查环境变量：`env | grep MYSQL`
3. 测试网络：`ping mysql`
4. 测试端口：`nc -zv mysql 3306`
5. 检查数据库容器是否启动

**问题：容器内存不足被杀**

排查步骤：

1. 查看退出码是否是 137
2. 查看内存使用：`docker stats`
3. 增加内存限制或优化程序

### 面试加分回答

> "在排查问题时，我通常会先看日志，因为日志里通常有最直接的错误信息。如果日志不够清晰，我会进入容器内部手动测试。对于复杂的网络问题，我还会用 tcpdump 或 wireshark 抓包分析。"

你还可以补充：

---

## 问题 10：你在实际项目中遇到过什么 Docker 相关的问题？怎么解决的？

### 这是一个开放性问题，考察你的实战经验

### 你可以参考以下回答模板

#### 问题排查

#### 问题原因（深入分析）

问题就解决了。"

```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server on 'localhost'")
```

```
传统部署：
┌─────────────────────────────────────────────┐
│              服务器                          │
│  ┌─────────────────────────────────────┐   │
│  │  Python 后端                         │   │
│  │  localhost → 指向本机的 MySQL        │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────────────────────────────────┐   │
│  │  MySQL                               │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘

- `.env`：本地开发用，数据库地址是 `localhost`
- `.env.docker`：Docker 部署用，数据库地址是 `mysql`（服务名）

```yaml
services:
  backend:
    env_file:
      - .env.docker
```

我会分享一个我实际遇到的问题：

#### 场景描述

"我在部署一个 Python 后端项目时，遇到了一个奇怪的问题。本地开发环境一切正常，但部署到 Docker 容器后，访问数据库总是超时。"

"我首先查看了容器日志，发现错误信息是：

然后我意识到问题所在：我在 `.env` 文件里配置的数据库地址是 `localhost`。"

"在 Docker 容器里，`localhost` 指的是容器自己，而不是宿主机或其他容器。

Docker 部署：
┌─────────────────────────────────────────────┐
│              Docker 网络                     │
│  ┌─────────────────┐  ┌─────────────────┐   │
│  │  后端容器        │  │  MySQL 容器     │   │
│  │                 │  │                 │   │
│  │  localhost      │  │                 │   │
│  │  → 指向自己！    │  │                 │   │
│  │  → 找不到 MySQL │  │                 │   │
│  └─────────────────┘  └─────────────────┘   │
└─────────────────────────────────────────────┘
```

数据库在另一个容器里，所以连不上。"

#### 解决方案

"我创建了两个环境变量文件：

然后在 `docker-compose.yml` 里指定使用 `.env.docker`：

#### 经验总结

"通过这个问题，我深刻理解了 Docker 容器网络隔离的概念。现在我会在项目一开始就考虑 Docker 部署的需求，提前规划好环境变量的配置方式。"

---

## 问题 11：Docker 和 Kubernetes 有什么关系？

### 这是一个考察你对技术生态理解的题

### 你应该怎么回答

- 把货物打包好
- 可以运输
- 可以堆放

- 管理很多集装箱
- 决定哪个集装箱放在哪里
- 监控集装箱状态
- 坏了自动换一个

- 你要手动启动 100 个容器
- 你要手动监控它们是否正常
- 如果某个容器挂了，你要手动重启
- 如果流量增加，你要手动扩容
- 如果某台服务器挂了，你要手动迁移

- 你告诉 Kubernetes：我要 100 个这个容器
- Kubernetes 自动分配到多台服务器
- Kubernetes 自动监控
- 容器挂了自动重启
- 流量增加自动扩容
- 服务器挂了自动迁移

- 自动分配流量

- 滚动更新，零停机部署

- 根据资源需求自动调度

- 容器挂了自动重启
- 节点挂了自动迁移

- 管理敏感信息

- 根据负载自动增减实例

```
┌─────────────────────────────────────────────┐
│                Kubernetes                    │
│  ┌─────────────────────────────────────┐   │
│  │            集群管理                  │   │
│  └─────────────────────────────────────┘   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ Docker  │  │ Docker  │  │ Docker  │     │
│  │ 容器    │  │ 容器    │  │ 容器    │     │
│  └─────────┘  └─────────┘  └─────────┘     │
└─────────────────────────────────────────────┘
```

- 开发环境
- 小项目
- 单机部署
- 学习和测试

- 大规模生产环境
- 微服务架构
- 需要高可用
- 需要自动扩缩容
- 多团队协作

#### 1. 简单回答

**Docker：** 负责打包和运行单个容器。

**Kubernetes：** 负责管理大规模的容器集群。

#### 2. 用类比来理解

**Docker 就像"一个集装箱"：**

**Kubernetes 就像"一个集装箱码头"：**

#### 3. 为什么需要 Kubernetes

假设你有 100 个容器要管理：

**只用 Docker：**

**用 Kubernetes：**

#### 4. Kubernetes 的核心功能

**服务发现和负载均衡：**

**自动部署和回滚：**

**自动装箱：**

**自我修复：**

**密钥和配置管理：**

**水平扩缩容：**

#### 5. 它们的关系

Docker 是 Kubernetes 的运行时之一（虽然现在 Kubernetes 也支持其他运行时，如 containerd）。

#### 6. 什么时候用 Docker，什么时候用 Kubernetes

**只适合用 Docker 的场景：**

**需要 Kubernetes 的场景：**

### 面试加分回答

> "在实际项目中，我通常会在开发阶段用 Docker Compose，因为它简单易用。到了生产环境，如果规模较大，会考虑迁移到 Kubernetes。这种渐进式的方案既能保证开发效率，又能满足生产需求。"

你还可以补充：

---
