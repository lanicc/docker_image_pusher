# Docker Images Pusher

使用Github Action将国外的Docker镜像转存到阿里云私有仓库，供国内服务器使用，免费易用<br>
- 支持 Docker Hub、gcr.io、registry.k8s.io、ghcr.io 等镜像仓库
- 通过 Crane 流式复制镜像，无需 Docker 守护进程或在本地展开镜像
- 默认保留所有可运行架构，自动排除 ACR 不兼容的 unknown/unknown 构建证明清单
- 支持 images.txt 批量同步，以及手动指定单个镜像、目标标签和架构

视频教程：https://www.bilibili.com/video/BV1Zn4y19743/

作者：**[技术爬爬虾](https://github.com/tech-shrimp/me)**<br>
B站，抖音，Youtube全网同名，转载请注明作者<br>

## 使用方式


### 配置阿里云
登录阿里云容器镜像服务<br>
https://cr.console.aliyun.com/<br>
创建个人版或企业版实例，并创建一个命名空间（**ALIYUN_NAME_SPACE**）。目标镜像仓库需提前创建，或在命名空间中启用自动创建仓库。

**ALIYUN_REGISTRY** 使用控制台提供的公网域名，不带 `https://`、命名空间或仓库路径。企业版需要开启公网入口，并通过访问控制允许 Runner 的出口地址；仅开放 VPC 地址时，需要使用能访问该 VPC 的自托管 Runner。
![](/doc/命名空间.png)

访问凭证–>获取环境变量<br>
用户名（**ALIYUN_REGISTRY_USER**)<br>
密码（**ALIYUN_REGISTRY_PASSWORD**)<br>
仓库地址（**ALIYUN_REGISTRY**）<br>

![](/doc/用户名密码.png)


### Fork本项目
Fork本项目<br>
#### 启动Action
进入您自己的项目，点击Action，启用Github Action功能<br>
#### 配置环境变量
进入Settings->Secret and variables->Actions->New Repository secret
![](doc/配置环境变量.png)
将上一步的**四个值**<br>
ALIYUN_NAME_SPACE,ALIYUN_REGISTRY_USER，ALIYUN_REGISTRY_PASSWORD，ALIYUN_REGISTRY<br>
配置成 Repository Secrets。原有四个 Secret 名称保持兼容，无需改名。

如果源镜像仓库需要登录，再添加以下三个可选 Repository Secrets（需要一起配置）：

| Secret | 说明 |
| --- | --- |
| `SOURCE_REGISTRY` | 源仓库域名，例如 `ghcr.io` 或 `docker.io` |
| `SOURCE_USERNAME` | 源仓库用户名 |
| `SOURCE_PASSWORD` | 源仓库密码或具有镜像读取权限的访问令牌 |

公开镜像可以不配置源仓库 Secrets；Docker Hub 遇到匿名拉取限流时，也可以配置登录信息。

### 添加镜像

编辑 `images.txt`，每行填写一个镜像。支持空行、`#` 注释和行尾注释；未指定 tag 时使用 `latest`。

```text
samanhappy/mcphub:1.2.4
nginx:1.28.0
--platform=linux/amd64 ghcr.io/owner/image:tag
--platform linux/arm64/v8 owner/image:tag
# 支持用 digest 锁定源镜像，不会退回到 latest
# owner/image@sha256:<完整的64位摘要>
```

提交到 `main` 后自动同步。仅修改 README 等无关文件不会触发镜像复制。

### 手动同步

进入 **Actions → Docker → Run workflow**，可填写：

| 参数 | 说明 |
| --- | --- |
| `source_image` | 源镜像；留空则批量同步 `images.txt` |
| `target_image` | 可选的目标 `仓库:标签`，例如 `mcphub:1.2.4`；不含 ACR 域名和命名空间 |
| `platform` | 可选的 `os/arch[/variant]`，例如 `linux/amd64`；留空保留所有可运行架构 |

`target_image` 和 `platform` 只用于填写了 `source_image` 的单镜像同步。目标标签未填写时，根据源镜像名称和标签自动生成。仅使用 digest 的源镜像会生成 `sha256-<完整摘要>` 标签。

现有 `images.txt` 中的 `samanhappy/mcphub:1.2.4` 会同步到：

```text
<ALIYUN_REGISTRY>/<ALIYUN_NAME_SPACE>/mcphub:1.2.4
```

同步结果及保留的架构会显示在 Actions 运行摘要中。网络操作失败后最多重试两次；登录凭证仅通过标准输入传递，并在本次任务结束时删除临时认证文件。

### 使用镜像
回到阿里云，镜像仓库，点击任意镜像，可查看镜像状态。(可以改成公开，拉取镜像免登录)
![](doc/开始使用.png)

在国内服务器pull镜像, 例如：<br>
```
docker pull registry.cn-hangzhou.aliyuncs.com/shrimp-images/alpine
```
registry.cn-hangzhou.aliyuncs.com 即 ALIYUN_REGISTRY(阿里云仓库地址)<br>
shrimp-images 即 ALIYUN_NAME_SPACE(阿里云命名空间)<br>
alpine 即 阿里云中显示的镜像名<br>

### 多架构

未指定 `--platform` 时，将源镜像中所有可运行平台复制到同一个目标 tag，拉取时由 Docker 自动选择匹配的架构。

指定 `--platform=linux/amd64` 或 `--platform linux/arm64/v8` 时，仅复制指定架构，默认在目标仓库名称前添加 `linux_amd64_` 或 `linux_arm64_v8_` 前缀。手动同步时，如果设置了 `target_image`，则使用显式指定的目标名称。

源镜像中的 `unknown/unknown` 清单通常是 SBOM 或 provenance 构建证明。部分 ACR 实例拒绝这类制品并报 `unknown manifest class for application/vnd.oci.empty.v1+json`。流水线会重建只包含可运行平台的目标清单，因此目标索引的 digest 可能与源索引不同；构建证明不会随镜像复制。

### 镜像重名
当镜像名称相同但源仓库路径不同时，会把源命名空间作为前缀加在目标仓库名称前；多层命名空间使用 `_` 连接。如果仍有多个不同来源映射到同一个目标，流水线会在复制前报错，避免互相覆盖。
例如:
```
xhofe/alist
xiaoyaliu/alist
```

分别生成 `xhofe_alist:latest` 和 `xiaoyaliu_alist:latest`。
![](doc/镜像重名.png)

### 定时执行
修改/.github/workflows/docker.yaml文件
添加 schedule即可定时执行(此处cron使用UTC时区)
![](doc/定时执行.png)
