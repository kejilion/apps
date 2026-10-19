# Coolify 安装前的 SSH 公钥认证检查

Coolify 需要通过 SSH 连接本机执行构建和部署。应用配置会检查主机的 OpenSSH 设置；出现 `OpenSSH 未允许公钥认证，请先配置 PubkeyAuthentication。` 时，先处理 SSH 配置，再重试安装。应用不会自动改写主机的 SSH 登录策略。

以下命令在**目标 Linux 服务器**以 root 执行。保留当前 SSH 或供应商控制台会话，直到新连接测试成功。

## 1. 查看实际生效的配置

```bash
sshd -t
sshd -T | grep -E '^(pubkeyauthentication|permitrootlogin|passwordauthentication|authenticationmethods|authorizedkeysfile|port) '
```

`sshd -t` 报错时先修正错误；不要继续重新加载服务。`sshd -T` 显示磁盘配置的解析结果，修改后仍需重新加载才能用于新连接。若服务用 `-f` 或 `-o` 指定了其他配置，检查时也要使用相同选项。

需要看到 `pubkeyauthentication yes`。`#PubkeyAuthentication yes` 是注释，不代表禁用了公钥认证；OpenSSH 默认值是 `yes`。若结果为 `no`，查找显式配置：

```bash
grep -nEi '^[[:space:]]*(Include|Match|PubkeyAuthentication|PermitRootLogin|PasswordAuthentication|AuthenticationMethods)[[:space:]]' /etc/ssh/sshd_config
grep -nREi '^[[:space:]]*(Include|Match|PubkeyAuthentication|PermitRootLogin|PasswordAuthentication|AuthenticationMethods)[[:space:]]' /etc/ssh/sshd_config.d 2>/dev/null
```

同时检查 `Include` 引入的其他路径。OpenSSH 通常采用同一设置最先取得的值；`Include` 在出现的位置读取，通配文件按文件名顺序处理。不要直接在文件末尾追加一条 `yes`。

有 `Match` 时，需结合实际 root 用户、Coolify 容器源 IP、接收地址及 SSH 端口，用 `sshd -T -C` 检查该连接的结果。只检查 `127.0.0.1` 不能代表 Docker 容器连接。当前应用的安装前检查使用全局 `sshd -T`；如果仅在 `Match` 条件中允许密钥认证，仍可能被全局检查拒绝。此时应核对实际连接条件并反馈配置差异，不要删除已有 `Match` 限制来绕过检查。

## 2. 备份并修改产生该结果的文件

先确定真正生效的那条 `PubkeyAuthentication no` 所在文件。下面以主文件为例；如果实际来自被包含的文件，修改变量为那个文件的真实路径。

```bash
ssh_config_file=/etc/ssh/sshd_config
ssh_backup_dir=$(mktemp -d /root/coolify-ssh-backup.XXXXXX)
cp -a -- "$ssh_config_file" "$ssh_backup_dir/config.original"
printf '备份位置：%s\n' "$ssh_backup_dir/config.original"
vi "$ssh_config_file"
```

只将已定位的设置改为：

```text
PubkeyAuthentication yes
```

保持端口、监听地址、`PasswordAuthentication` 和原有 root 登录策略不变。开启公钥认证不要求关闭密码认证。

本应用还需要允许 root 使用密钥登录：已有 `PermitRootLogin yes` 或 `prohibit-password` 可保留，`without-password` 是后者的旧别名。若为 `no` 或 `forced-commands-only`，需由管理员决定是否允许 Coolify 所需的 root SSH 操作。`PermitRootLogin prohibit-password` 允许 root 密钥登录，但会禁止 root 密码登录；仍依赖 root 密码连接时，不要照搬此设置。还需确认 `AuthenticationMethods` 未强制要求密钥之外的交互认证。

## 3. 检查通过后重新加载服务

```bash
sshd -t
sshd -T | grep -E '^(pubkeyauthentication|permitrootlogin|passwordauthentication|port) '
systemctl status ssh.service sshd.service --no-pager
```

确认检查通过、原有登录方式和端口保持不变，再对实际运行的服务执行**其中一条**：

```bash
# Debian / Ubuntu 常见服务名
systemctl reload ssh.service

# RHEL 等发行版常见服务名
systemctl reload sshd.service
```

使用其他服务管理器时，按该系统实际的 OpenSSH 服务执行重新加载，不要同时套用以上命令。另开一个终端验证原有 SSH 登录成功后，再关闭旧会话。

回到 KPanel 重试安装。安装完成后，在 Coolify 的 **Servers → localhost** 检查连接与验证状态。若失败，应继续检查密钥授权、实际 SSH 端口和容器到主机的连通性，无需关闭整个防火墙。

需要回退时，保留上文变量，或先将它们设置为真实文件和备份目录，再恢复原文件：

```bash
cp -a -- "$ssh_backup_dir/config.original" "$ssh_config_file"
sshd -t
```

语法检查通过后，重新加载上面确认的同一个 SSH 服务，并另开连接验证。备份只还原本次修改的文件；不要覆盖之后由其他操作新增的配置。

参考：[Coolify OpenSSH 文档](https://coolify.io/docs/core/infrastructure/servers/openssh)、[OpenSSH sshd_config 手册](https://man.openbsd.org/sshd_config)、[OpenSSH sshd 手册](https://man.openbsd.org/sshd)。
