以下是基于你之前改建的 `docker-compose.yml` 文件（使用 `.env` 管理变量）的完整 **SeedDMS 文档管理系统 Docker 部署教程**，去除了安装 Docker 的部分，并假设你已经配置好了 Docker 和 Docker Compose 环境。

---

## **SeedDMS 文档管理系统 Docker 部署教程**

### **1. 创建安装目录**
在服务器上创建 SeedDMS 的安装目录，并进入该目录：
```bash
mkdir -p /docker/seeddms
cd /docker/seeddms
```

---

### **2. 编写 `docker-compose.yml` 文件**
在 `/docker/seeddms` 目录下创建 `docker-compose.yml` 文件，并添加以下内容：

```yaml
version: '3.8'

# 定义服务
services:
  seeddms:
    image: usteinm/seeddms:latest
    container_name: seeddms
    ports:
      - ${SEEDDMS_PORT}:80
    volumes:
      - ./seeddms/seeddms-data:/home/www-data/seeddms60x/data
      - ./seeddms/seeddms-conf:/var/lib/seeddms/conf
      - ./seeddms/seeddms-import:/var/lib/seeddms/import
      - ./seeddms/seeddms-checkout:/var/lib/seeddms/checkout
      - ./seeddms/seeddms-ext:/home/www-data/seeddms60x/www/ext
    environment:
      DB_HOST: mysql
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
    depends_on:
      mysql:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  mysql:
    image: mysql:latest
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASS}
    volumes:
      - ./mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
      retries: 3
      timeout: 5s
    restart: unless-stopped

# 定义卷，用于数据持久化
volumes:
  seeddms-data:
  seeddms-conf:
  seeddms-import:
  seeddms-checkout:
  seeddms-ext:
  mysql-data:
```

---

### **3. 创建 `.env` 文件**
在 `/docker/seeddms` 目录下创建 `.env` 文件，并添加以下内容：

```env
# SeedDMS 配置
SEEDDMS_PORT=9021
DB_NAME=seeddms
DB_USER=seeddms
DB_PASS=password

# MySQL 配置
MYSQL_ROOT_PASSWORD=pass_root
```

> **注意**: 如果需要，可以根据你的环境修改这些值（如端口号、数据库名称、用户名和密码等）。

---

### **4. 创建相关文件夹和文件**
在 `/docker/seeddms` 目录下运行以下命令，创建所需的文件夹和文件：
```bash
# 创建 SeedDMS 数据目录
mkdir -p ./seeddms/{seeddms-data,seeddms-conf,seeddms-import,seeddms-checkout,seeddms-ext}
touch ./seeddms/seeddms-conf/ENABLE_INSTALL_TOOL

# 创建 Lucene、staging 和 cache 子目录
mkdir -p ./seeddms/seeddms-data/{lucene,staging,cache}

# 创建 MySQL 数据目录
mkdir -p ./mysql-data
```

---

### **5. 设置权限**
设置文件夹权限，确保容器内的 `www-data` 用户可以访问这些目录：
```bash
sudo chown -R www-data:www-data ./seeddms/
sudo chown -R www-data:www-data ./mysql-data/
```

---

### **6. 启动服务**
使用 Docker Compose 启动 SeedDMS 和 MySQL 服务：
```bash
docker compose up -d
```

检查服务状态：
```bash
docker ps
```
确保 `seeddms` 和 `mysql` 容器正在运行。

查看日志（可选）：
```bash
docker logs seeddms
docker logs mysql
```

---

### **7. 初始化 SeedDMS**
打开浏览器，访问 `http://<主机IP>:9021`，开始初始化 SeedDMS。

#### **步骤 1：数据库配置**
在初始化页面中，填写以下数据库信息：
- **Database Type**: `MySQL`
- **Server name**: `mysql`
- **Database**: `seeddms`（与 `.env` 文件中的 `DB_NAME` 对应）
- **Username**: `seeddms`（与 `.env` 文件中的 `DB_USER` 对应）
- **Password**: `password`（与 `.env` 文件中的 `DB_PASS` 对应）
- **Create database tables**: 勾选 `Yes`

点击 **Submit** 按钮继续。

#### **步骤 2：删除 ENABLE_INSTALL_TOOL 文件**
初始化完成后，系统会提示你删除 `ENABLE_INSTALL_TOOL` 文件。执行以下命令删除文件：
```bash
rm ./seeddms/seeddms-conf/ENABLE_INSTALL_TOOL
```

#### **步骤 3：设置管理员账户**
系统会跳转到登录页面，默认管理员账户为：
- **用户名**: `admin`
- **密码**: `admin`

首次登录后，系统会要求你修改管理员密码。请根据提示设置一个安全的密码。

---

### **8. 使用 SeedDMS**
完成初始化后，你可以通过 `http://<主机IP>:9021` 访问 SeedDMS 文档管理系统。使用管理员账户登录后，开始上传文档、管理用户等操作。

---

### **9. 注意事项**

#### **1. 数据持久化**
- 所有数据存储在 `./seeddms` 和 `./mysql-data` 目录中。如果需要迁移或备份，请确保复制这些目录。
- 如果删除这些目录，所有数据将丢失。

#### **2. 安全性**
- 不要将 `.env` 文件提交到版本控制系统（如 Git）。可以在 `.gitignore` 文件中添加以下内容：
  ```
  .env
  ```
- 在生产环境中启用 HTTPS，并定期更新密码。

#### **3. 升级**
- 要升级 SeedDMS，请拉取最新镜像并重新启动服务：
  ```bash
  docker pull usteinm/seeddms:latest
  docker compose down
  docker compose up -d
  ```

#### **4. 日志与排错**
- 如果遇到问题，可以通过以下命令查看日志：
  ```bash
  docker logs seeddms
  docker logs mysql
  ```

---

### **总结**
通过以上步骤，你已经成功部署了一个基于 Docker 的 SeedDMS 文档管理系统，并使用 `.env` 文件管理敏感信息。希望这个教程对你有所帮助！如果有其他问题，请随时提问。
