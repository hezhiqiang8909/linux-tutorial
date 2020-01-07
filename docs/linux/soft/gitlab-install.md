# Gitlab 安装

> 环境：
>
> OS: CentOS7

* [安装 gitlab](gitlab-install.md#安装-gitlab)
  * [常规安装 gitlab](gitlab-install.md#常规安装-gitlab)
  * [Docker 安装 gitlab](gitlab-install.md#docker-安装-gitlab)
* [安装 gitlab-ci-multi-runner](gitlab-install.md#安装-gitlab-ci-multi-runner)
  * [常规安装 gitlab-ci-multi-runner](gitlab-install.md#常规安装-gitlab-ci-multi-runner)
  * [Docker 安装 gitlab-ci-multi-runner](gitlab-install.md#docker-安装-gitlab-ci-multi-runner)
* [自签名证书](gitlab-install.md#自签名证书)
  * [创建证书](gitlab-install.md#创建证书)
* [gitlab 配置](gitlab-install.md#gitlab-配置)
* [更多内容](gitlab-install.md#更多内容)

## 安装 gitlab

### 常规安装 gitlab

进入官方下载地址：[https://about.gitlab.com/install/](https://about.gitlab.com/install/) ，如下图，选择合适的版本。

![img](http://dunwu.test.upcdn.net/snap/20190129155838.png!zp)

以 CentOS7 为例：

#### 安装和配置必要依赖

在系统防火墙中启用 HTTP 和 SSH

```text
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```

安装 Postfix ，使得 Gitlab 可以发送通知邮件

```text
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```

#### 添加 Gitlab yum 仓库并安装包

添加 Gitlab yum 仓库

```text
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

通过 yum 安装 gitlab-ce

```text
sudo EXTERNAL_URL="http://gitlab.example.com" yum install -y gitlab-ce
```

安装完成后，即可通过默认的 root 账户进行登录。更多细节可以参考：[documentation for detailed instructions on installing and configuration](https://docs.gitlab.com/omnibus/README.html#installation-and-configuration-using-omnibus-package)

### Docker 安装 gitlab

拉取镜像

```text
docker pull docker.io/gitlab/gitlab-ce
```

启动

```text
docker run -d \
    --hostname gitlab.zp.io \
    --publish 8443:443 --publish 80:80 --publish 2222:22 \
    --name gitlab \
    --restart always \
    --volume $GITLAB_HOME/config:/etc/gitlab \
    --volume $GITLAB_HOME/logs:/var/log/gitlab \
    --volume $GITLAB_HOME/data:/var/opt/gitlab \
    gitlab/gitlab-ce
```

![img](http://dunwu.test.upcdn.net/snap/20190131150515.png!zp)

## 安装 gitlab-ci-multi-runner

> 参考：[https://docs.gitlab.com/runner/install/](https://docs.gitlab.com/runner/install/)

### 常规安装 gitlab-ci-multi-runner

#### 下载

```text
sudo wget -O /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
```

#### 配置执行权限

```text
sudo chmod +x /usr/local/bin/gitlab-runner
```

#### 如果想使用 Docker，安装 Docker（可选的）

```text
curl -sSL https://get.docker.com/ | sh
```

#### 创建 CI 用户

```text
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

#### 安装并启动服务

```text
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

#### 注册 Runner

（1）执行命令：

```text
sudo gitlab-runner register
```

（2）输入 Gitlab URL 和 令牌

URL 和令牌信息在 Gitlab 的 Runner 管理页面获取：

![img](http://dunwu.test.upcdn.net/snap/20190129163100.png!zp)

```text
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
https://gitlab.com

Please enter the gitlab-ci token for this runner
xxx
```

（3）输入 Runner 的描述

```text
 Please enter the gitlab-ci description for this runner
 [hostame] my-runner
```

（4）输入 Runner 相关的标签

```text
 Please enter the gitlab-ci tags for this runner (comma separated):
 my-tag,another-tag
```

（5）输入 Runner 执行器

```text
 Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
 docker
```

如果想选择 Docker 作为执行器，你需要指定默认镜像（ `.gitlab-ci.yml` 中没有此配置）

```text
 Please enter the Docker image (eg. ruby:2.1):
 alpine:latest
```

### Docker 安装 gitlab-ci-multi-runner

拉取镜像

```text
docker pull docker.io/gitlab/gitlab-runner
```

启动

```text
docker run -d --name gitlab-runner --restart always \
   -v /srv/gitlab-runner/config:/etc/gitlab-runner \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest
```

## 自签名证书

首先，创建认证目录

```text
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 700 /etc/gitlab/ssl
```

### 创建证书

#### 创建 Private Key

```text
sudo openssl genrsa -des3 -out /etc/gitlab/ssl/gitlab.domain.com.key 2048
```

会提示输入密码，请记住

#### 生成 Certificate Request

```text
sudo openssl req -new -key /etc/gitlab/ssl/gitlab.domain.com.key -out /etc/gitlab/ssl/gitlab.domain.com.csr
```

根据提示，输入信息

```text
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:JS
Locality Name (eg, city) [Default City]:NJ
Organization Name (eg, company) [Default Company Ltd]:xxxxx
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:gitlab.xxxx.io
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

#### 移除 Private Key 中的密码短语

```text
sudo cp -v /etc/gitlab/ssl/gitlab.domain.com.{key,original}
sudo openssl rsa -in /etc/gitlab/ssl/gitlab.domain.com.original -out /etc/gitlab/ssl/gitlab.domain.com.key
sudo rm -v /etc/gitlab/ssl/gitlab.domain.com.original
```

#### 创建证书

```text
sudo openssl x509 -req -days 1460 -in /etc/gitlab/ssl/gitlab.domain.com.csr -signkey /etc/gitlab/ssl/gitlab.domain.com.key -out /etc/gitlab/ssl/gitlab.domain.com.crt
```

#### 移除证书请求文件

```text
sudo rm -v /etc/gitlab/ssl/gitlab.domain.com.csr
```

#### 设置文件权限

```text
sudo chmod 600 /etc/gitlab/ssl/gitlab.domain.com.*
```

## gitlab 配置

```text
sudo vim /etc/gitlab/gitlab.rb
external_url 'https://gitlab.domain.com'
```

#### gitlab 网站 https：

```text
nginx['redirect_http_to_https'] = true
```

#### gitlab ci 网站 https：

```text
ci_nginx['redirect_http_to_https'] = true
```

#### 复制证书到 gitlab 目录：

```text
sudo cp /etc/gitlab/ssl/gitlab.domain.com.crt /etc/gitlab/trusted-certs/
```

#### gitlab 重新配置+更新：

```text
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```

### 创建你的 SSH key

1. 使用 Gitlab 的第一步是生成你自己的 SSH 密钥对（Github 也类似）。
2. 登录 Gitlab
3. 打开 **Profile settings**.

![img](https://docs.gitlab.com/ce/gitlab-basics/img/profile_settings.png)

1. 跳转到 **SSH keys** tab 页

![img](https://docs.gitlab.com/ce/gitlab-basics/img/profile_settings_ssh_keys.png)

1. 黏贴你的 SSH 公钥内容到 Key 文本框

![img](https://docs.gitlab.com/ce/gitlab-basics/img/profile_settings_ssh_keys_paste_pub.png)

1. 为了便于识别，你可以为其命名

![img](https://docs.gitlab.com/ce/gitlab-basics/img/profile_settings_ssh_keys_title.png)

1. 点击 **Add key** 将 SSH 公钥添加到 GitLab

![img](https://docs.gitlab.com/ce/gitlab-basics/img/profile_settings_ssh_keys_single_key.png)

### 创建项目

![img](http://dunwu.test.upcdn.net/snap/20190131150658.png!zp)

输入项目信息，点击 Create project 按钮，在 Gitlab 创建项目。

![img](http://dunwu.test.upcdn.net/snap/20190131150759.png!zp)

### 克隆项目到本地

可以选择 SSH 或 HTTPS 方式克隆项目到本地（推荐 SSH）

拷贝项目地址，然后在本地执行 `git clone <url>`

### 创建 Issue

依次点击 **Project’s Dashboard** &gt; **Issues** &gt; **New Issue** 可以新建 Issue

![img](https://docs.gitlab.com/ce/user/project/issues/img/new_issue_from_tracker_list.png)

在项目中直接添加 issue

![img](https://docs.gitlab.com/ce/user/project/issues/img/new_issue.png)

在未关闭 issue 中，点击 **New Issue** 添加 issue

![img](https://docs.gitlab.com/ce/user/project/issues/img/new_issue_from_open_issue.png)

通过项目面板添加 issue

![img](https://docs.gitlab.com/ce/user/project/issues/img/new_issue_from_projects_dashboard.png)

通过 issue 面板添加 issue

![img](https://docs.gitlab.com/ce/user/project/issues/img/new_issue_from_issue_board.png)

## 更多内容

* **引申**
  * [操作系统、运维部署总结系列](https://github.com/dunwu/OS)

