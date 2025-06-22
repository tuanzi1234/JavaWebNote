在 Ubuntu 上安装 Nginx 非常简单，只需使用 `apt` 包管理器即可完成。以下是详细步骤：

---

### **1. 更新系统软件包列表**

打开终端（Ctrl+Alt+T），执行：

```
sudo apt update
```

---

### **2. 安装 Nginx**

```
sudo apt install nginx
```

输入 `Y` 确认安装。

---

### **3. 验证安装**

* **检查 Nginx 服务状态** ：

```
  sudo systemctl status nginx
```

  若看到 `active (running)` 表示运行成功。

* **测试默认页面** ：
  在浏览器中输入服务器 IP 地址（或 `http://localhost`），看到 **"Welcome to nginx!"** 页面即安装成功。

---

### **4. 常用命令**

* 启动 Nginx：

  ```
  sudo systemctl start nginx
  ```
* 停止 Nginx：

  ```
  sudo systemctl stop nginx
  ```
* 重启 Nginx（修改配置后需执行）：

  ```
  sudo systemctl restart nginx
  ```
* 开机自启：

  ```
  sudo systemctl enable nginx
  ```

---

### **5. 防火墙设置（如启用）**

允许 HTTP/HTTPS 流量：

```
sudo ufw allow 'Nginx Full'  # 或单独允许 HTTP/HTTPS
sudo ufw reload
```

---

### **6. 配置文件与网站目录**

* **配置文件位置** ：
* 主配置：`/etc/nginx/nginx.conf`
* 站点配置：`/etc/nginx/sites-available/`
* **默认网站根目录** ：
  `/var/www/html/`

---

### **卸载 Nginx（可选）**

```
sudo apt remove --purge nginx*  # 彻底删除
sudo apt autoremove             # 清理依赖
```

> ⚠️ 提示：
>
> * 安装前确保  **80/443 端口未被占用** （如 Apache）。
> * 需要 sudo 权限执行命令。
> * 配置修改后使用 `sudo nginx -t` 测试语法正确性。

完成以上步骤后，Nginx 即可正常使用！后续可配置虚拟主机、SSL 证书等进阶功能。
