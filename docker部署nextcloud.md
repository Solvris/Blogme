以下是一个完整的 `docker-compose.yml` 配置文件和 `.env` 文件的示例，以及如何使用它们的详细教程。

---

### 1. **项目结构**
假设你的项目目录如下：
```
my-nextcloud/
├── docker-compose.yml
├── .env
├── postgresql/
└── nextcloud/
```

- `postgresql/`：PostgreSQL 数据存储目录。
- `nextcloud/`：Nextcloud 数据存储目录。
- `.env`：环境变量文件。
- `docker-compose.yml`：Docker Compose 配置文件。

---

### 2. **完整配置文件**

#### **docker-compose.yml**
```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    container_name: nextcloud_db
    restart: unless-stopped
    volumes:
      - ./postgresql:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - 9001:80
    volumes:
      - ./nextcloud:/var/www/html
    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
```

---

#### **.env**
在项目根目录下创建一个 `.env` 文件，并定义所有敏感信息：

```env
# PostgreSQL 配置
POSTGRES_DB=mydatabase
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword

# Nextcloud 管理员账户配置
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=adminpassword
```

---

### 3. **教程：如何使用**

#### **步骤 1：准备项目目录**
1. 创建项目目录（如 `my-nextcloud`）。
2. 在项目目录中创建以下子目录：
   - `postgresql/`：用于存储 PostgreSQL 数据。
   - `nextcloud/`：用于存储 Nextcloud 数据。
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
   确保 `nextcloud_db` 和 `nextcloud` 容器正在运行。

2. 查看日志：
   如果服务启动失败，可以通过以下命令查看日志：
   ```bash
   docker logs nextcloud_db
   docker logs nextcloud
   ```

3. 访问 Nextcloud：
   打开浏览器，访问 `http://localhost:9001`，并使用 `.env` 文件中定义的管理员账户登录（`NEXTCLOUD_ADMIN_USER` 和 `NEXTCLOUD_ADMIN_PASSWORD`）。

---

### 4. **注意事项**

#### **1. 数据持久化**
- `postgresql/` 和 `nextcloud/` 目录用于存储数据。如果删除这些目录，数据将丢失。
- 确保定期备份这些目录中的数据。

#### **2. 安全性**
- 不要将 `.env` 文件提交到版本控制系统（如 Git）。可以在 `.gitignore` 文件中添加以下内容：
  ```
  .env
  ```
- 如果需要在生产环境中部署，建议启用 HTTPS，并使用更复杂的密码。

#### **3. 健康检查**
- `db` 服务的健康检查确保 PostgreSQL 已完全启动后再启动 `app` 服务。
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
