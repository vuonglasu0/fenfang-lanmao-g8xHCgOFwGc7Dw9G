大家好，我是 **XuHe**。在上一篇博客[告别 LaTeX 配置地狱：我的 Overleaf (ShareLaTeX-CE) 完整版私有化部署方案](https://github.com)，我想详细解析一下我为 **Overleaf Community Edition (ShareLaTeX CE)** 项目所配置的 `docker-compose.yaml` 文件。

一个稳定、可维护的 Overleaf 私有部署，其灵魂就在于如何配置好三个核心服务：**ShareLaTeX 应用**、**MongoDB 数据库**和 **Redis 缓存**。

我这份配置不仅解决了**AVX 兼容性问题**，还通过一系列环境变量优化了**使用体验**和**编译稳定性**。

```
services:
    sharelatex:
        restart: always
        # image: sharelatex/sharelatex:4.2.3
        image: xuhe-sharelatex-ce:latest # custom image(include new texlive)
        container_name: leaf-sharelatex
        depends_on:
            mongo:
                condition: service_healthy
            redis:
                condition: service_started
        ports:
            - 8008:80
        links:
            - mongo
            - redis
        stop_grace_period: 60s
        volumes:
            - ./sharelatex:/var/lib/sharelatex
        environment:
            SHARELATEX_APP_NAME: XuHe's Overleaf

            SHARELATEX_MONGO_URL: mongodb://mongo/sharelatex

            # Same property, unfortunately with different names in
            # different locations
            SHARELATEX_REDIS_HOST: redis
            REDIS_HOST: redis

            ENABLED_LINKED_FILE_TYPES: "project_file,project_output_file"

            # Enables Thumbnail generation using ImageMagick
            ENABLE_CONVERSIONS: "true"

            # Disables email confirmation requirement
            EMAIL_CONFIRMATION_DISABLED: "true"

            # temporary fix for LuaLaTex compiles
            # see https://github.com/overleaf/overleaf/issues/695
            TEXMFVAR: /var/lib/sharelatex/tmp/texmf-var

    mongo:
        restart: always
        image: mongo:4.4
        container_name: leaf-mongo
        command: "--replSet overleaf"
        expose:
            - 27017
        volumes:
            - ./mongo:/data/db
        healthcheck:
            test: echo 'db.stats().ok' | mongo localhost:27017/test --quiet
            interval: 10s
            timeout: 10s
            retries: 5

    redis:
        restart: always
        image: redis:6.2
        container_name: leaf-redis
        expose:
            - 6379
        volumes:
            - ./redis:/data
```

### 一、核心服务总览

我的部署方案中包含以下三个服务：

| 服务名称 | 角色 | 关键版本（兼容性考量） |
| --- | --- | --- |
| `sharelatex` | 主应用服务 | 自定义镜像 (`xuhe-sharelatex-ce:latest`) |
| `mongo` | 数据存储服务 | `mongo:4.4` (为兼容无 AVX 指令集 CPU) |
| `redis` | 缓存与实时数据服务 | `redis:6.2` |

---

### 二、ShareLaTeX 应用配置详解 (`sharelatex` Service)

`sharelatex` 服务是整个部署的核心。这里我们不仅指定了自定义镜像，还通过环境变量进行了大量功能定制。

#### 1. 启动与依赖

| 配置项 | 值 | 核心作用 |
| --- | --- | --- |
| `image` | `xuhe-sharelatex-ce:latest` | 使用我们**自定义的完整 TeX Live 镜像**。 |
| `restart` | `always` | 保证服务崩溃或重启后自动恢复运行。 |
| `depends_on` | `mongo` (healthy), `redis` (started) | 确保 ShareLaTeX 只有在数据库和缓存服务准备就绪后才启动。 |

#### 2. 核心环境变量 (`environment`)

以下环境变量是定制 Overleaf 行为的关键，我重点介绍了几个提高用户体验和稳定性的配置：

| 环境变量 | 值 | 目的与说明 |
| --- | --- | --- |
| `SHARELATEX_APP_NAME` | `XuHe's Overleaf` | **应用名称**。自定义浏览器 Tab 和登录页面的名称，增强私有化标识。 |
| `SHARELATEX_MONGO_URL` | `mongodb://mongo/sharelatex` | **MongoDB 连接**。使用 Docker Compose 内部服务名 `mongo` 进行连接。 |
| `EMAIL_CONFIRMATION_DISABLED` | `"true"` | **跳过邮件验证**。默认情况下，新用户注册需要邮件验证。将其设置为 `true` 可直接登录使用，非常适合内部或私有部署。 |
| `ENABLE_CONVERSIONS` | `"true"` | **启用图片/缩略图转换**。允许 Overleaf 使用 ImageMagick 进行图片处理和生成 PDF 缩略图。 |
| `TEXMFVAR` | `/var/lib/sharelatex/tmp/texmf-var` | **LuaLaTeX 兼容性修复**。这是一个针对特定版本 LuaLaTeX 编译问题的临时修复，提升编译稳定性。 |
| `ENABLED_LINKED_FILE_TYPES` | `"project_file,..."` | **启用外部文件类型链接**（高级）。用于控制哪些文件类型可以被链接或引用。 |

#### 3. 数据持久化与网络

* **端口映射 (`ports`)**: 我将容器内部的 `80` 端口映射到宿主机的 `8008` 端口 (`8008:80`)，方便通过浏览器访问。您可以根据需要修改 `8008`。
* **数据卷 (`volumes`)**: `./sharelatex:/var/lib/sharelatex`。这是**至关重要**的配置，它将容器内的项目数据目录持久化映射到宿主机上的 `./sharelatex` 文件夹，确保项目文件不会丢失。

### 三、数据库和缓存服务配置

#### 1. MongoDB (`mongo` Service)

* **版本 (`image`)**: 选择了 **`mongo:4.4`**。这是为了规避最新版 MongoDB 对 **AVX 指令集**的强制依赖，以确保在旧硬件上顺利运行。
* **副本集 (`command`)**: `"--replSet overleaf"`。这是 Overleaf 运行所必需的配置，用于启用 MongoDB 的**副本集**功能。启动后，还需要执行一次 `docker exec` 命令进行初始化（详见我的 README）。
* **健康检查 (`healthcheck`)**: 保证 ShareLaTeX 服务只在 MongoDB 确认可以连接并工作正常时才启动，提升启动可靠性。
* **数据卷 (`volumes`)**: `./mongo:/data/db`，确保数据库数据持久化。

#### 2. Redis (`redis` Service)

* **版本 (`image`)**: `redis:6.2`，一个稳定且功能完整的 Redis 版本。
* **作用**: Redis 主要用于 Overleaf 的实时数据、会话管理和缓存，是实现协同编辑功能的基础。
* **数据卷 (`volumes`)**: `./redis:/data`，用于持久化 Redis 的持久化数据。

---

### 四、结语

这份 `docker-compose.yaml` 文件是我在保证**最大硬件兼容性**的前提下，对 Overleaf 部署进行定制和优化的成果。通过精准的环境变量配置，我们不仅解决了常见的部署难题，还让 Overleaf 的使用体验更加贴合私有化需求。

欢迎访问我的 GitHub 仓库 [xuhe2/sharelatex-ce](https://github.com):[juzi橘子](https://juzi996.com) 获取完整的配置文件和部署指南。

---

**🙏 特别致谢**

本文的结构梳理与内容精炼得到了 **Gemini** 智能助手的协助。
