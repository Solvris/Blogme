以下是改进后的配置文件，改为使用 `.env` 文件管理敏感信息和变量，并对部分细节进行了优化。这样可以避免在 `docker-compose.yml` 文件中硬编码敏感信息，同时提高可维护性和安全性。

---

### 1. **项目结构**
假设你的项目目录如下：
```
owncloud/
├── docker-compose.yml
├── .env
├── mariadb/
├── redis/
└── owncloud/
```

- `mariadb/`：MariaDB 数据存储目录。
- `redis/`：Redis 数据存储目录。
- `owncloud/`：OwnCloud 数据存储目录。
- `.env`：环境变量文件。
- `docker-compose.yml`：Docker Compose 配置文件。

---

### 2. **完整配置文件**

#### **docker-compose.yml**
```yaml
version: "3"

volumes:
  files: # 定义名为 "files" 的数据卷
    driver: local
  mysql: # 定义名为 "mysql" 的数据卷
    driver: local
  redis: # 定义名为 "redis" 的数据卷
    driver: local

services:
  owncloud:
    image: owncloud/server:${OWNCLOUD_VERSION}
    container_name: owncloud_server
    restart: unless-stopped
    ports:
      - ${HTTP_PORT}:8080
    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      OWNCLOUD_DOMAIN: ${OWNCLOUD_DOMAIN}
      OWNCLOUD_TRUSTED_DOMAINS: ${OWNCLOUD_TRUSTED_DOMAINS}
      OWNCLOUD_DB_TYPE: mysql
      OWNCLOUD_DB_NAME: ${OWNCLOUD_DB_NAME}
      OWNCLOUD_DB_USERNAME: ${OWNCLOUD_DB_USER}
      OWNCLOUD_DB_PASSWORD: ${OWNCLOUD_DB_PASSWORD}
      OWNCLOUD_DB_HOST: mariadb
      OWNCLOUD_ADMIN_USERNAME: ${ADMIN_USERNAME}
      OWNCLOUD_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      OWNCLOUD_MYSQL_UTF8MB4: true
      OWNCLOUD_REDIS_ENABLED: true
      OWNCLOUD_REDIS_HOST: redis
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - ./owncloud:/mnt/data

  mariadb:
    image: mariadb:latest
    container_name: owncloud_mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_USER: ${OWNCLOUD_DB_USER}
      MYSQL_PASSWORD: ${OWNCLOUD_DB_PASSWORD}
      MYSQL_DATABASE: ${OWNCLOUD_DB_NAME}
      MARIADB_AUTO_UPGRADE: 1
    command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "--password=${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ./mariadb:/var/lib/mysql

  redis:
    image: redis:latest
    container_name: owncloud_redis
    restart: unless-stopped
    command: ["--databases", "1"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ./redis:/data
```

---

#### **.env**
在项目根目录下创建一个 `.env` 文件，并定义所有变量：

```env
# OwnCloud 配置
OWNCLOUD_VERSION=latest
OWNCLOUD_DOMAIN=localhost:9011
OWNCLOUD_TRUSTED_DOMAINS=localhost,10.0.0.20
ADMIN_USERNAME=admin
ADMIN_PASSWORD=oc2024
HTTP_PORT=9011

# 数据库配置
OWNCLOUD_DB_NAME=owncloud
OWNCLOUD_DB_USER=owncloud
OWNCLOUD_DB_PASSWORD=owncloud
MYSQL_ROOT_PASSWORD=owncloud
```

---

### 3. **教程：如何使用**

#### **步骤 1：准备项目目录**
1. 创建项目目录（如 `owncloud`）。
2. 在项目目录中创建以下子目录：
   - `mariadb/`：用于存储 MariaDB 数据。
   - `redis/`：用于存储 Redis 数据。
   - `owncloud/`：用于存储 OwnCloud 数据。
3. 在项目根目录下创建 `.env` 文件，并复制上述内容。

#### **步骤 2：编辑配置文件**
1. 将上述 `docker-compose.yml` 文件保存到项目根目录。
2. 确保 `.env` 文件中的变量值符合你的需求（例如，修改数据库名、用户名、密码等）。

#### **步骤 3：启动服务**
在项目根目录下运行以下命令：

```bash
docker-compose up -d
```

- `-d` 参数表示以后台模式运行容器。
- Docker Compose 会自动读取 `.env` 文件中的变量，并将其注入到 `docker-compose.yml` 中。

#### **步骤 4：验证服务是否正常运行**
1. 检查容器状态：
   ```bash
   docker ps
   ```
   确保 `owncloud_server`、`owncloud_mariadb` 和 `owncloud_redis` 容器正在运行。

2. 查看日志：
   如果服务启动失败，可以通过以下命令查看日志：
   ```bash
   docker logs owncloud_server
   docker logs owncloud_mariadb
   docker logs owncloud_redis
   ```

3. 访问 OwnCloud：
   打开浏览器，访问 `http://localhost:9011`，并使用 `.env` 文件中定义的管理员账户登录（`ADMIN_USERNAME` 和 `ADMIN_PASSWORD`）。

---

### 4. **注意事项**

#### **1. 数据持久化**
- `mariadb/`、`redis/` 和 `owncloud/` 目录用于存储数据。如果删除这些目录，数据将丢失。
- 确保定期备份这些目录中的数据。

#### **2. 安全性**
- 不要将 `.env` 文件提交到版本控制系统（如 Git）。可以在 `.gitignore` 文件中添加以下内容：
  ```
  .env
  ```
- 如果需要在生产环境中部署，建议启用 HTTPS，并使用更复杂的密码。

#### **3. 健康检查**
- `mariadb` 和 `redis` 服务的健康检查确保它们已完全启动后再启动 `owncloud` 服务。
- 如果健康检查失败，可以调整 `interval`、`timeout` 和 `retries` 参数。

#### **4. 版本控制**
- 如果你使用的是较新的 Docker Compose 版本（如 v2.x），可以直接运行 `docker compose`（无需连字符）：
  ```bash
  docker compose up -d
  ```

---

### 5. **总结**

通过 `.env` 文件管理敏感信息是一种安全且灵活的方式，适合在开发和生产环境中使用。以下是关键点回顾：
- 使用 `.env` 文件存储敏感信息。
- 在 `docker-compose.yml` 中通过 `${VARIABLE_NAME}` 引用变量。
- 确保数据目录存在并正确挂载。
- 定期备份数据，避免意外丢失。

希望这个教程对你有帮助！如果有其他问题，请随时提问。
