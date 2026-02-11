# Git SSH 配置与连接故障排查

## 1. 一些概念

*   **Git Commit**: 纯本地操作，保存快照，与网络无关。
*   **Git Push**: 将本地快照上传到远程。
*   **SSH Key**: 是身份凭证。**同一公钥可以同时在 GitLab 和 GitHub 上登记**，互不冲突。

---

## 2. 配置过程

#### 1. 生成新的 SSH Key

​	在终端输入：

```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

​	设置密码可以直接回车跳过（不设置密码）。

- - 如果你在这个步骤输入了密码，以后每次 git push 都要输一遍，或者需要配置 ssh-agent。

#### 2. 启动 SSH 代理 (可选)

​	SSH Agent 是一个后台程序，帮你管理密钥，避免即使设置了密码也频繁输入。

​	**后台启动 ssh-agent**:

```
eval "$(ssh-agent -s)"
```

​	*输出类似 Agent pid 12345 表示成功。*

​	**将私钥添加到代理**:

```
ssh-add ~/.ssh/id_ed25519
```

​	*如果用的是旧的 RSA 算法，则是 ssh-add ~/.ssh/id_rsa*

#### 3. 上传公钥

在以下目录

```
~/.ssh/id_ed25519.pub
```

从 ssh-ed25519 开头，一直复制到你的邮箱结尾。确保不要多复制空格或换行。

添加到 GitHub:

- **GitHub**: 头像 -> Settings -> SSH and GPG keys -> **New SSH key**。

#### 4. 测试连通

**测试 GitHub**:

```
ssh -T git@github.com
```

成功看到 Hi <用户名>! You've successfully authenticated...即可

------



## 3. 故障排查过程

### 1. 执行 `git push origin main` 时报错：

```text
\302\226\302\226\302\226\302\226git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**排查步骤**:

1. **检查本地密钥**: 使用 ls -al ~/.ssh 确认存在密钥
2. **测试连接**: 运行 ssh -T git@github.com。*结果*: Hi Fallmoon74! ... 即为连接成功。但无法正常push
3. 日志显示认证使用的用户名是 \302\226\302\226git 而不是纯净的 git。是因为配置远程地址时，**复制粘贴引入了不可见的控制字符（乱码）**。


于是删除并重新手动添加干净的远程地址。

```
# 1. 删除脏地址
git remote remove origin

# 2. 添加干净地址
git remote add origin git@github.com:Fallmoon74/Arch-Linux-Debug-Log.git
```

------



### 2. 网络连接中断

解决了乱码后，推送时报错：

```
kex_exchange_identification: read: Software caused connection abort
# 或者
Connection closed by 20.205.243.166 port 443
```

**原因**: 尝试通过 ~/.ssh/config 强制走 443 端口，但配置与本地代理冲突。由于本地已开启代理工具，不需要在 SSH Config 中强制指定 Port 443 或修改 Hostname。代理会自动处理连接。

**最终有效的 SSH Config (~/.ssh/config)**:

```bash
# GitHub specific configuration
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
```

------



### 3. 拒绝合并

**现象**: 推送报错

```
! [rejected]        main -> main (fetch first)
hint: Updates were rejected because the remote contains work that you do not have locally.
```

**原因**:
GitHub 上的仓库在创建时生成了 README.md 或 LICENSE 文件，而本地仓库是独立初始化的。Git 认为这是两个不相关的历史线，默认拒绝合并。

**解决方案**:
强制拉取远程更改，并允许不相关的历史合并。

```bash
# 1. 建立追踪关系
git branch --set-upstream-to=origin/main main

# 2. 拉取并强制合并
git pull origin main --allow-unrelated-histories
```