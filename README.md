# 🚀 开发者应用入驻指南 (kejilion.sh)

欢迎为 `kejilion.sh` 贡献应用！通过在 `apps/` 目录下创建 `.conf` 配置文件，您的应用将自动集成到脚本的应用市场中。

---

## 1. 快速开始
在 `apps/` 目录下创建一个以应用名命名的配置文件，例如：`myapp.conf`。

## 2. 配置文件模板
每个配置文件应严格遵循以下结构，确保变量和函数命名规范：

```bash
# --- 基础信息 / Basic Information ---
local app_id="My File Name" # 和文件名一致，用于安装状态检测 / Same as the filename, used for installation status detection.
local app_name="My App Name" # 应用显示名称
local app_text="A brief description" # 一句话简介，说明应用用途
local app_url="https://github.com/..." # 官网链接 / Official URL
local docker_name="myapp_container" # 容器启动后的名称 / Container Name
local docker_port="8080" # 默认访问端口 / Default Port
local app_size="1" # 占用空间大小(GB) / Size in GB required (1-10)

# --- 核心逻辑 / Core Logic ---
docker_app_install() {
    # 必须在 /home/docker/ 下创建应用目录 / Directory creation (Mandatory)
    mkdir -p /home/docker/myapp && cd /home/docker/myapp
    # 下载并配置 compose 文件 / Download compose file
    # 务必使用 gh_proxy 变量 / Use gh_proxy for China access
    curl -L -o docker-compose.yml "${gh_proxy}https://raw.githubusercontent.com/..."
    # 端口处理（使用变量以便用户自定义）/ Port configuration (Customizable)
    sed -i "s/8080:8080/${docker_port}:8080/g" docker-compose.yml
    # 启动容器 / Start container
    docker compose up -d
    echo "安装完成 / Install Complete"

    # 显示访问地址的函数保留即可 / Show the function that is reserved
    check_docker_app_ip
}

docker_app_update() {
    cd /home/docker/myapp
    docker compose pull
    docker compose up -d
    echo "更新完成 / Update Complete"
}

docker_app_uninstall() {
    cd /home/docker/myapp
    # 停止并删除镜像 / Stop and remove images
    docker compose down --rmi all
    # 彻底物理删除目录 / Clean up directory
    rm -rf /home/docker/myapp
    echo "卸载完成 / Uninstall Complete"
}

# --- 注册 (必须包含) / Registration (Mandatory) ---
docker_app_plus



```

## 3. 强制规范与原则

### 📁 目录路径规范
> **核心原则：数据不入系统盘，统一归档。**

* **【必须】**：所有持久化数据（Volume/Bind Mount）必须存储在 `/home/docker/[应用名]` 目录下。
* **【禁止】**：严禁将数据存放在 `/root`、`/etc`、`/var/lib` 或其他非指定根目录。
* **【理由】**：统一路径方便用户进行一键备份、整机迁移以及权限的统一管理。

### 🔄 容器生命周期
* **开机自启**：生成的 `docker-compose.yml` 中必须包含 `restart: always` 或 `restart: unless-stopped`。
* **干净卸载**：`docker_app_uninstall` 函数必须执行闭环操作，包含：
    * 停止并删除容器 (`docker compose down`)
    * 删除对应的镜像 (`--rmi all`)
    * **彻底物理删除** `/home/docker/[应用名]` 目录。

### 🆔 变量与语法说明
* **App ID**：当前版本已弱化 ID 概念，您可以省略或填入任意数值，系统目前主要以 `.conf` 文件名作为唯一识别依据。
* **Local 关键字**：由于配置文件是在函数内部被 `source` 加载的，请务必保留 `local` 声明，这能有效防止变量污染脚本的全局环境。

### 🌐 网络优化
* **镜像加速**：下载 GitHub 资源（如 `.yml` 或 `脚本`）时，请务必在 URL 前加上 `${gh_proxy}` 变量，以确保国内服务器的访问成功率。


---


## 4. 快捷启动与调用

一旦您的 `.conf` 文件被合入仓库，该应用将自动进入 **“第三方应用入驻”** 模块。此外，开发者可以向用户提供专属的极简安装指令：

#### 🚀 快捷安装指令模板：
```bash
bash <(curl -sL kejilion.sh) app [文件名]
```
示例：如果您的配置文件名为 myapp.conf，则调用指令为： bash <(curl -sL kejilion.sh) app myapp


---


## 5. 入驻流程

1.  **本地自测**：把 `[应用名].conf` 放到 VPS 的 `~/apps-local/` 目录（见第 6 节），在自己的 VPS 上完整运行安装、更新、卸载流程，确保无报错。
2.  **路径审计**：检查 `/home/docker/` 目录下是否正确生成了应用文件夹，且没有文件“溢出”到其他地方。
3.  **提交申请**：将您的 `[应用名].conf` 文件通过 **Pull Request** 提交至本仓库的 `sh/apps/` 目录。
4.  **审核发布**：维护者审核逻辑安全后，您的应用将正式上线 `kejilion.sh` 菜单。


---


## 6. 本地应用（仅本机使用 / 提交前自测）

不想公开、或还在调试的应用，可以只放在自己的服务器上，无需提交 PR：

1. 按第 2 节模板写好配置，保存为 `~/apps-local/[应用名].conf`（root 用户即 `/root/apps-local/`）。
2. 收紧权限：文件必须属于当前用户，且不能被其他用户写入，软链接会被忽略：
   ```bash
   chown root:root /root/apps-local/myapp.conf && chmod 644 /root/apps-local/myapp.conf
   ```
3. 打开 `kejilion.sh` 应用市场，应用出现在 **“本地应用列表”** 中；KPanel 应用市场点击“刷新状态”后出现在 **“本地应用”** 分类，两边读的是同一个目录。
4. 安装、更新、卸载与正式应用完全一致，也可直接运行：`bash <(curl -sL kejilion.sh) app [文件名]`。

注意事项：

* **官方优先**：与本仓库中的应用同名时以官方配置为准，本地配置会被忽略，请换一个文件名。
* **安全自负**：本地应用未经审核，会以 root 身份执行，请只使用自己编写或信任的配置。
* **版本要求**：需要支持 `~/apps-local` 的新版 `kejilion.sh`；KPanel 检测到旧脚本时会禁用本地应用的安装与管理按钮。
* 自测通过后，把同一个 `.conf` 按第 5 节提交 PR，即可上架给所有用户。

## 7. 应用使用说明

* [Coolify：安装前的 SSH 公钥认证检查与配置](docs/coolify.md)
