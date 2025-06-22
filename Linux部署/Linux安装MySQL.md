### 🚀 Linux Ubuntu 安装 MySQL 8.0.32 终极指南

#### 📥 安装方法总结（任选其一）

**方法一：APT 仓库安装（推荐）**

```bash
# 添加官方仓库
sudo wget https://dev.mysql.com/get/mysql-apt-config_0.8.28-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.28-1_all.deb  # 选择 Ubuntu Jammy → MySQL 8.0

# 解决GPG错误
sudo wget https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/mysql.gpg RPM-GPG-KEY-mysql-2023

# 安装指定版本
sudo apt update
sudo apt install mysql-server=8.0.32-1ubuntu22.04
```

**方法二：二进制包安装（离线首选）**

```bash
# 下载解压
sudo wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.32-linux-glibc2.17-x86_64-minimal.tar.xz
sudo tar -xvf mysql-8.0.32-linux-glibc2.17-x86_64-minimal.tar.xz -C /usr/local
sudo mv /usr/local/mysql-8.0.32-* /usr/local/mysql

# 初始化数据库
sudo groupadd mysql
sudo useradd -r -g mysql -s /bin/false mysql
sudo mkdir /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo /usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/var/lib/mysql
# 记录输出的临时密码
```

**方法三：Docker 容器安装（最简方式）**

```bash
sudo docker run -d --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=yourpassword \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0.32
```

---

### ⚙️ 环境变量配置

#### 1. **永久添加 MySQL 到 PATH**

```bash
# 编辑profile文件
echo 'export PATH=$PATH:/usr/local/mysql/bin' | sudo tee -a /etc/profile.d/mysql.sh

# 立即生效
source /etc/profile.d/mysql.sh
```

#### 2. **创建常用命令别名**

```bash
# 添加到 ~/.bashrc
echo "alias mysqlstart='sudo systemctl start mysql'" >> ~/.bashrc
echo "alias mysqlstop='sudo systemctl stop mysql'" >> ~/.bashrc
echo "alias mysqlstatus='sudo systemctl status mysql'" >> ~/.bashrc
source ~/.bashrc
```

#### 3. **配置 MySQL 环境变量**

```bash
# 设置默认socket路径
echo '[client]' >> ~/.my.cnf
echo 'socket=/var/run/mysqld/mysqld.sock' >> ~/.my.cnf
chmod 600 ~/.my.cnf
```

---

### ✅ 安装验证步骤

#### 1. **基础验证**

```bash
# 检查服务状态
sudo systemctl status mysql

# 验证版本
mysql --version
# 应输出：mysql  Ver 8.0.32 for Linux on x86_64

# 检查监听端口
sudo netstat -tulnp | grep 3306
# 应显示：tcp6 0 0 :::3306 :::* LISTEN
```

#### 2. **数据库连接测试**

```bash
# 使用root登录（输入安装时设置的密码）
mysql -u root -p -e "SELECT VERSION(), NOW()"

# 输出示例：
# +-----------+---------------------+
# | VERSION() | NOW()               |
# +-----------+---------------------+
# | 8.0.32    | 2024-06-20 14:30:00 |
# +-----------+---------------------+
```

#### 3. **深度功能测试**

```bash
# 创建测试数据库
mysql -u root -p -e "CREATE DATABASE install_test"

# 创建测试用户
mysql -u root -p <<EOF
CREATE USER 'testuser'@'localhost' IDENTIFIED BY 'Test@1234';
GRANT ALL PRIVILEGES ON install_test.* TO 'testuser'@'localhost';
FLUSH PRIVILEGES;
EOF

# 验证用户权限
mysql -u testuser -pTest@1234 -e "SHOW DATABASES"
```

---

### 🔧 安装后必需配置

#### 1. **安全加固**

```bash
sudo mysql_secure_installation
```

按提示操作：

```text
1. 设置root密码强度（推荐Y）
2. 选择密码策略（生产环境选2=STRONG）
3. 移除匿名用户? [Y]
4. 禁止远程root登录? [Y]
5. 删除测试数据库? [Y]
6. 重新加载权限表? [Y]
```

#### 2. **性能优化配置**

```bash
sudo nano /etc/mysql/my.cnf
```

添加：

```bash
[mysqld]
innodb_buffer_pool_size = 1G  # 设置为物理内存的70%
max_connections = 200
default_authentication_plugin=mysql_native_password
```

#### 3. **远程访问配置**

```sql
-- 创建远程管理用户
CREATE USER 'admin'@'%' IDENTIFIED BY 'StrongPass!2024';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

-- 更新绑定地址
SET GLOBAL bind_address = '0.0.0.0';
```

---

### 🛠️ 故障排查工具箱

#### 1. **日志分析**

```bash
# 实时查看错误日志
sudo tail -f /var/log/mysql/error.log

# 查看慢查询日志
sudo nano /var/log/mysql/mysql-slow.log
```

#### 2. **密码重置**

```bash
sudo systemctl stop mysql
sudo mysqld_safe --skip-grant-tables &
mysql -u root

# MySQL内执行：
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPass!2024';
exit
sudo killall mysqld_safe
sudo systemctl start mysql
```

#### 3. **连接问题诊断**

```bash
# 检查防火墙
sudo ufw status
sudo ufw allow 3306/tcp

# 测试本地连接
mysqladmin -u root -p ping
# 应输出：mysqld is alive
```

---

### 📊 验证清单

| **项目** | **命令**                                                        | **预期结果**   |
| -------------- | --------------------------------------------------------------------- | -------------------- |
| 服务状态       | `systemctl status mysql`                                            | `active (running)` |
| 版本验证       | `mysql --version`                                                   | `8.0.32`           |
| 端口监听       | `netstat -tulnp \| grep 3306`                                        | `:::3306` 监听状态 |
| 本地连接       | `mysqladmin -u root -p ping`                                        | `mysqld is alive`  |
| 远程连接       | `telnet 服务器IP 3306`                                              | 建立连接             |
| 性能状态       | `mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'Threads_connected'"` | 连接数正常           |

> **终极建议** ：
>
> 1. 生产环境使用 **APT安装** + **密码策略2级**
> 2. 开发环境使用 **Docker容器**
> 3. 每次修改配置后重启服务：
>    `sudo systemctl restart mysql`
> 4. 定期备份：
>    `mysqldump -u root -p --all-databases \| gzip > backup_$(date +%F).sql.gz`
>
## 要在Windows物理机上远程访问Linux Ubuntu虚拟机中的MySQL数据库，请按照以下步骤操作：

 ### 1. 在Ubuntu虚拟机中配置MySQL允许远程访问

 **bash**


 ```bash
 # 编辑MySQL配置文件
 sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

 # 找到 [mysqld] 部分下的 bind-address 行
 # 将 bind-address = 127.0.0.1 改为 ↓
 bind-address = 0.0.0.0

 # 保存并退出 (Ctrl+X → Y → Enter)
 # 重启MySQL服务
 sudo systemctl restart mysql
 ```
 ### 2. 创建远程访问用户并授权
 ```bash
 # 登录MySQL
 sudo mysql -u root -p

 -- 创建新用户 (替换 'username' 和 'password')
 CREATE USER 'username'@'%' IDENTIFIED BY 'password';

 -- 授予所有数据库的访问权限
 GRANT ALL PRIVILEGES ON *.* TO 'username'@'%' WITH GRANT OPTION;

 -- 刷新权限
 FLUSH PRIVILEGES;

 -- 退出
 EXIT;
 ```
 ### 3. 配置Ubuntu防火墙
 ```bash
 # 允许MySQL端口 (默认3306)
 sudo ufw allow 3306/tcp

 # 重载防火墙
 sudo ufw reload
 ```
 ### 4. 获取Ubuntu虚拟机的IP地址
 ```bash
 ip a
 # 查看 ens33 或 eth0 接口的 inet 地址 (如 192.168.x.x)
 ```
