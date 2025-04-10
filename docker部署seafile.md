以下是基于你提供的 `docker-compose.yml` 文件的完整 **Seafile 文档管理系统 Docker 部署教程**，使用 `.env` 文件管理敏感信息，并优化了部分配置。

---

## **Seafile 文档管理系统 Docker 部署教程**

### **1. 创建安装目录**
在服务器上创建 Seafile 的安装目录，并进入该目录：
```bash
mkdir -p /docker/seafile
cd /docker/seafile
```

---

### **2. 编写 `docker-compose.yml` 文件**
在 `/docker/seafile` 目录下创建 `docker-compose.yml` 文件，并添加以下内容：

```yaml
version: '3.9'

services:
  db:
    image: mariadb:latest
    container_name: seafile-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_LOG_CONSOLE: "true"
      MARIADB_AUTO_UPGRADE: 1
    volumes:
      - ./seafile/seafile-mysql/db:/var/lib/mysql
    networks:
      - seafile-net
    restart: unless-stopped

  memcached:
    image: memcached:latest
    container_name: seafile-memcached
    entrypoint: memcached -m 256
    networks:
      - seafile-net
    restart: unless-stopped

  seafile:
    image: seafileltd/seafile-mc:latest
    container_name: seafile
    ports:
      - "${SEAFILE_PORT}:80"
    volumes:
      - ./seafile-data:/shared
    environment:
      DB_HOST: db
      DB_ROOT_PASSWD: ${MYSQL_ROOT_PASSWORD}
      TIME_ZONE: ${TIME_ZONE}
      SEAFILE_ADMIN_EMAIL: ${SEAFILE_ADMIN_EMAIL}
      SEAFILE_ADMIN_PASSWORD: ${SEAFILE_ADMIN_PASSWORD}
      SEAFILE_SERVER_LETSENCRYPT: "false"
      SEAFILE_SERVER_HOSTNAME: ${SEAFILE_SERVER_HOSTNAME}
    depends_on:
      - db
      - memcached
    networks:
      - seafile-net
    restart: unless-stopped

networks:
  seafile-net:
```

---

### **3. 创建 `.env` 文件**
在 `/docker/seafile` 目录下创建 `.env` 文件，并添加以下内容：

```env
# 数据库配置
MYSQL_ROOT_PASSWORD=db_dev

# Seafile 配置
SEAFILE_PORT=8010
TIME_ZONE=Asia/Shanghai
SEAFILE_ADMIN_EMAIL=admin@me.com
SEAFILE_ADMIN_PASSWORD=sf2024
SEAFILE_SERVER_HOSTNAME=seafile.local.com
```

> **注意**: 如果需要，可以根据你的环境修改这些值（如端口号、管理员邮箱、密码等）。

---

### **4. 创建相关文件夹**
在 `/docker/seafile` 目录下运行以下命令，创建所需的文件夹：
```bash
# 创建 MySQL 数据目录
mkdir -p ./seafile/seafile-mysql/db

# 创建 Seafile 数据目录
mkdir -p ./seafile-data
```

---

### **5. 启动服务**
使用 Docker Compose 启动 Seafile 和相关服务：
```bash
docker compose up -d
```

检查服务状态：
```bash
docker ps
```
确保 `seafile-mysql`、`seafile-memcached` 和 `seafile` 容器正在运行。

查看日志（可选）：
```bash
docker logs seafile-mysql
docker logs seafile-memcached
docker logs seafile
```

---

### **6. 初始化 Seafile**
打开浏览器，访问 `http://<主机IP>:8010`，开始初始化 Seafile。

#### **步骤 1：登录管理员账户**
- 默认管理员账户为 `.env` 文件中定义的邮箱和密码：
  - **邮箱**: `admin@me.com`
  - **密码**: `sf2024`

#### **步骤 2：完成初始设置**
登录后，根据提示完成 Seafile 的初始设置，例如：
- 配置存储路径。
- 设置共享链接选项。
- 配置邮件服务器（可选）。

---

### **7. 使用 Seafile**
完成初始化后，你可以通过 `http://<主机IP>:8010` 访问 Seafile 文档管理系统。使用管理员账户登录后，开始上传文档、管理用户等操作。

---

### **8. 注意事项**

#### **1. 数据持久化**
- 所有数据存储在 `./seafile/seafile-mysql/db` 和 `./seafile-data` 目录中。如果需要迁移或备份，请确保复制这些目录。
- 如果删除这些目录，所有数据将丢失。

#### **2. 安全性**
- 不要将 `.env` 文件提交到版本控制系统（如 Git）。可以在 `.gitignore` 文件中添加以下内容：
  ```
  .env
  ```
- 在生产环境中启用 HTTPS，并定期更新密码。

#### **3. 升级**
- 要升级 Seafile，请拉取最新镜像并重新启动服务：
  ```bash
  docker pull seafileltd/seafile-mc:latest
  docker compose down
  docker compose up -d
  ```

#### **4. 日志与排错**
- 如果遇到问题，可以通过以下命令查看日志：
  ```bash
  docker logs seafile-mysql
  docker logs seafile-memcached
  docker logs seafile
  ```

---

### **总结**
通过以上步骤，你已经成功部署了一个基于 Docker 的 Seafile 文档管理系统，并使用 `.env` 文件管理敏感信息。希望这个教程对你有所帮助！如果有其他问题，请随时提问。
