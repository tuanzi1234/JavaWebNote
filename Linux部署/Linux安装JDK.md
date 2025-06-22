* **Linux安装JDK：**

```bash
# 1. 安装依赖
sudo apt update && sudo apt install -y curl zip unzip

# 2. 安装 SDKMAN!（支持多版本管理）
curl -s "https://get.sdkman.io" | bash

# 3. 初始化环境（新开终端或执行）
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 4. 查看可用 JDK 版本
sdk list java

# 5. 安装多个 JDK 版本（示例）
sdk install java 17.0.11-tem    # 安装 Eclipse Temurin 17
sdk install java 21.0.2-amzn    # 安装 Amazon Corretto 21
sdk install java 22.3.1-grl     # 安装 GraalVM 22

# 6. 查看已安装版本
sdk current java

# 7. 切换 JDK 版本
sdk use java 17.0.11-tem

# 8. 设置默认版本
sdk default java 21.0.2-amzn
```

* **查看JDK版本**：

```bash
# 检查当前 Java 版本
java -version

# 检查 SDKMAN! 管理的版本
sdk list java | grep "installed" -A 2

# 检查系统备选版本
update-alternatives --list java
```
