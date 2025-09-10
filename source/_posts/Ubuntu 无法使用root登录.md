---
tags:
  - OS
  - Linux
  - Ubuntu
title: Ubuntu 无法使用root登录
excerpt: 说明Ubuntu系统无法通过root直接登录的原因
categories: [操作系统, linux, ubuntu]
date: 2025-09-10 21:46:04
---
## Ubuntu root用户登录

在 Ubuntu 系统中，默认情况下 `root` 用户是无法直接通过密码登录的。Ubuntu 的设计理念是不建议直接使用 `root` 用户登录系统（包括本地和远程登录），而是使用普通用户并通过 `sudo` 提升权限。

### 原因

Ubuntu 默认禁用 `root` 用户的密码，因此 `root` 用户在本地控制台尝试登录时，也会因为没有设置密码而失败。即使为 `root` 用户设置了密码，许多 Ubuntu 版本仍然通过 PAM（可插入认证模块）配置禁止 `root` 用户登录到 GUI。

### Ubuntu 系统默认的 `root` 用户登录策略

1. **默认禁用 `root` 用户密码**：
   - 在 Ubuntu 中，`root` 用户的密码是禁用的，因此即使尝试登录 `root`，系统也不会允许。
   - 默认情况下，Ubuntu 创建的第一个用户是管理员，拥有 `sudo` 权限，可以通过 `sudo` 提升权限来完成管理任务。

2. **推荐使用 `sudo`**：
   - Ubuntu 通过 `sudo` 提供了一个更安全的权限提升方式，避免直接使用 `root` 用户。
   - 使用 `sudo` 的好处是，管理员任务可以由普通用户临时获得权限，并且所有使用 `sudo` 的操作都会被记录下来，便于审计。

### 允许 `root` 通过密码本地登录

如果确有需要启用 `root` 用户的本地登录，可以执行以下步骤：

1. **为 `root` 用户设置密码**（如果还没有设置）：
   ```bash
   sudo passwd root
   ```
   设置密码后，`root` 用户的密码认证将被启用。

2. **检查并修改 PAM 配置**：
   - 确保 `/etc/pam.d/gdm-password`（适用于 GNOME 桌面环境）或 `/etc/pam.d/login` 文件中没有阻止 `root` 登录的配置。
   - 检查是否有类似 `auth required pam_succeed_if.so user != root quiet_success` 的行。这一行会阻止 `root` 用户通过 GUI 登录。

	 如需允许 `root` 用户登录，可以将这一行注释掉：
```bash
# auth required pam_succeed_if.so user != root quiet_success
```

3. **重新启动登录管理器**（例如 `gdm`、`lightdm`）以使更改生效：
   ```bash
   sudo systemctl restart gdm
   ```

### 允许 `root` 通过密码ssh登录

如果确实需要允许 `root` 用户直接登录，可以按照以下步骤进行配置：

1. **为 `root` 用户设置密码**：
   - 你需要先为 `root` 用户设置一个密码：
     ```bash
     sudo passwd root
     ```
   - 系统会提示输入并确认 `root` 用户的新密码。

2. **允许 `root` 用户通过 SSH 登录**：
   - 打开 SSH 配置文件 `/etc/ssh/sshd_config`：
     ```bash
     sudo nano /etc/ssh/sshd_config
     ```
   - 找到 `PermitRootLogin` 配置，将其修改为 `yes`（去掉注释符号 `#`）：
     ```bash
     PermitRootLogin yes
     ```
   - 保存并退出文件，然后重启 SSH 服务：
     ```bash
     sudo systemctl restart sshd
     ```

### 安全性提示


>[!error] 警告
>
启用 `root` 用户的本地登录存在安全风险，因为它会绕过 `sudo` 的审计和日志记录功能。一般建议使用普通用户并通过 `sudo` 提升权限来执行管理任务，而不是直接使用 `root` 用户。
>
启用 `root` ssh登录后，也存在较高的安全风险，容易受到暴力破解攻击。因此，强烈建议仅在必要时临时启用 `root` 用户登录，并尽可能使用 SSH 公钥认证方式。通过上述配置，`root` 用户将能够直接登录到系统。但启用后，请确保使用强密码或通过防火墙限制 `root` 登录的 IP 地址，以提高安全性。

## 如何更新普通用户的密码

```bash
sudo su root

passwd user #更改user用户密码
passwd root #更改root用户密码
```