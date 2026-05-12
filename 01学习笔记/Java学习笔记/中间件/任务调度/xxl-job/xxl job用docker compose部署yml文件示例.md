```
# 命令执行 docker-compose up -d
version: '3.9'
services:
  # http://127.0.0.1:9090/xxl-job-admin admin/123456 - 安装后稍等会访问即可
  # 官网镜像为 xuxueli/xxl-job-admin 但不支持ARM架构【需要自己打包】，所以找了一个 kuschzzp/xxl-job-aarch64:2.4.0 镜像支持 AMD/ARM
  xxl-job-admin:
    image: kuschzzp/xxl-job-aarch64:2.4.0
    container_name: xxl-job-admin
    restart: always
    depends_on:
      - mysql
    ports:
      - "9090:9090"
    links:
      - mysql
    volumes:
      - ./data/logs:/data/applogs
      - ./data/xxl-job/:/xxl-job
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/xxl_job?serverTimezone=UTC&characterEncoding=utf8&autoReconnect=true&serverTimezone=Asia/Shanghai
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=123456
      - SERVER_PORT=9090

  # MySQL 8.0.32 支持 AMD/ARM
  mysql:
    image: mysql:8.0.32
    container_name: mysql
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
      TZ: Asia/Shanghai
      # MYSQL_ALLOW_EMPTY_PASSWORD: 'yes' 可配置无密码，注意配置 SPRING_DATASOURCE_PASSWORD=
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_USER: xfg
      MYSQL_PASSWORD: 123456
    depends_on:
      - mysql-job-dbdata
    ports:
      - "13306:3306"
    volumes:
      - ./sql:/docker-entrypoint-initdb.d
    volumes_from:
      - mysql-job-dbdata

  # 自动加载数据
  mysql-job-dbdata:
    image: alpine:3.18.2
    container_name: mysql-job-dbdata
    volumes:
      - /var/lib/mysql


这是一份xxl-job的配置文件，包含了什么信息呢？
```

这份 Docker Compose 配置文件用于部署 **XXL-JOB** 任务调度平台及其依赖的 MySQL 数据库。它包含了以下核心信息：

## 1. 整体结构
- 定义了三个服务：`xxl-job-admin`（调度中心）、`mysql`（数据库）、`mysql-job-dbdata`（数据卷容器）
- 使用了 Docker Compose 3.9 版本语法
- 支持 **AMD64** 和 **ARM64** 架构（特意替换了官方镜像以兼容 ARM）

## 2. 服务详解

### 2.1 `xxl-job-admin` – 调度中心
| 配置项 | 值 | 说明 |
|--------|-----|------|
| 镜像 | `kuschzzp/xxl-job-aarch64:2.4.0` | 第三方构建的镜像，可运行于 ARM 架构（如 Apple M1/M2、树莓派等） |
| 容器名 | `xxl-job-admin` | 固定名称 |
| 端口映射 | `9090:9090` | 宿主机 9090 端口 → 容器 9090 端口（Web 管理界面） |
| 依赖 | `mysql` | 等待 MySQL 容器启动后再启动 |
| 数据卷 | `./data/logs:/data/applogs`<br>`./data/xxl-job/:/xxl-job` | 挂载日志和程序数据到宿主机当前目录下的 `data` 文件夹 |
| 环境变量 | `SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/xxl_job?...` | 数据库连接地址（通过服务名 `mysql` 访问） |
|  | `SPRING_DATASOURCE_USERNAME=root` | 数据库用户名 |
|  | `SPRING_DATASOURCE_PASSWORD=123456` | 数据库密码（与 MySQL 配置保持一致） |
|  | `SERVER_PORT=9090` | 容器内服务端口 |

> **访问方式**：启动后通过 `http://127.0.0.1:9090/xxl-job-admin` 访问，默认管理员账号 `admin` / `123456`。

### 2.2 `mysql` – 数据库服务
| 配置项 | 值 | 说明 |
|--------|-----|------|
| 镜像 | `mysql:8.0.32` | 官方 MySQL 8.0.32，原生支持 ARM/AMD |
| 容器名 | `mysql` | 固定名称 |
| 端口映射 | `13306:3306` | 宿主机 13306 端口 → 容器 3306 端口（避免与本地 MySQL 冲突） |
| 命令 | `--default-authentication-plugin=mysql_native_password` | 使用传统密码认证，兼容老版本客户端 |
| 环境变量 | `MYSQL_ROOT_PASSWORD=123456` | root 密码 |
|  | `MYSQL_USER=xfg`<br>`MYSQL_PASSWORD=123456` | 额外创建普通用户 `xfg` |
| 数据卷 | `./sql:/docker-entrypoint-initdb.d` | 将宿主机 `./sql` 目录下的 `.sql` 脚本自动导入（用于初始化 XXL-JOB 表结构） |
| 卷依赖 | `volumes_from: mysql-job-dbdata` | 共享 `mysql-job-dbdata` 容器的数据卷，实现数据持久化 |

### 2.3 `mysql-job-dbdata` – 数据卷容器（辅助）
| 配置项 | 值 | 说明 |
|--------|-----|------|
| 镜像 | `alpine:3.18.2` | 极简 Linux 发行版，仅用于挂载卷 |
| 容器名 | `mysql-job-dbdata` | 固定名称 |
| 数据卷 | `/var/lib/mysql` | MySQL 的实际数据存储路径 |
| 运行方式 | 没有长期运行的进程，只是提供一个卷定义 | 生命周期随 docker-compose 管理，数据不会因 `mysql` 容器重启而丢失 |

## 3. 关键依赖关系
- **启动顺序**：`mysql-job-dbdata`（卷准备） → `mysql`（数据库进程） → `xxl-job-admin`（调度中心）
- **网络通信**：通过 `links: mysql`（老式写法）使 `xxl-job-admin` 容器能解析到 `mysql` 主机名。实际现代 Compose 会自动创建网络，也可以省略。
- **数据初始化**：`mysql` 容器首次启动时会执行 `./sql/*.sql` 文件，自动创建 `xxl_job` 数据库及表。

## 4. 使用提示
- 执行 `docker-compose up -d` 即可后台启动整套环境。
- 确保宿主机当前目录下存在 `sql/` 文件夹，并放入 XXL-JOB 官方提供的 `tables_xxl_job.sql` 脚本。
- 运行后等待几十秒（MySQL 初始化 + 调度中心启动），然后浏览器访问 `http://127.0.0.1:9090/xxl-job-admin`。
- 数据持久化：日志存于 `./data/logs`，数据库文件存于 Docker 卷（由 `mysql-job-dbdata` 维护）。

## 5. 潜在注意点
- `volumes_from` 已被 Docker Compose 新版本标记为过时，推荐使用命名卷替代。但当前配置依然有效。
- 数据库密码明文写在环境变量中，生产环境建议改用 Docker secrets 或外部配置。
- `links` 可用 `networks` 替代，但无功能性问题。

总结：这是一份**开箱即用、支持 ARM 架构的 XXL-JOB 开发/测试环境配置**，包含了调度中心、MySQL 数据库及自动初始化、数据持久化方案。