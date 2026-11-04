# DNF 服务：集成 DNF 服务器管理平台

本仓库 fork 自 [1995chen/dnf](https://github.com/1995chen/dnf)。原项目将 DNF 服务、数据库及初始化脚本整合进 Docker，支持通过环境变量快速部署；感谢原作者及相关项目为服务容器化所做的工作。本仓库在此基础上集成了 Web 管理平台，提供一套可通过图形界面管理的 DNF 服务端。

这里把游戏服务简称为**服务**，把 DNF 服务器管理平台简称为**平台**。集成镜像包含服务、数据库、Java 运行环境和平台，一个容器即可启动整套服务。

## 相关项目

- [原项目：1995chen/dnf](https://github.com/1995chen/dnf)：本仓库的上游项目。
- [DNF 服务器管理平台：dnf-server-public](https://github.com/onlyGuo/dnf-server-public)：开放的 Web 后台，可独立部署，也可与服务部署在同一宿主机或容器。
- [DNFLogin 登录器](https://github.com/onlyGuo/DNFLogin)：配套登录器，使用平台提供的登录、配置和版本更新接口。
- [PVF/NPK 解析器](https://github.com/onlyGuo/dnf_parser)：平台用于解析游戏资源的项目。

## 相对原项目的主要变化

- **统一管理入口**：平台提供账号、角色、邮件、物品、任务等 GM 运营功能，也能通过页面管理服务；日常操作不再依赖单独的本地 GM 工具。后台按账号角色控制权限，管理员也可以作为玩家登录游戏。
- **角色与背包**：编辑角色属性、浏览和修改背包及快捷栏，向角色发送邮件和物品。
- **频道运维**：新增、编辑、启用、停用和重启频道，无需为单个频道变更重启整个容器或重新运行整套服务。可选择游戏默认类型、地下城、交易、亡者峡谷、普通决斗场、自由决斗场和公平决斗场。页面显示频道进程内存及每频道约 1 GB 的预估。
- **动态地址**：默认通告地址和各频道地址均可配置固定 IPv4 或 DDNS。单频道地址改变时只重启该频道；默认地址改变涉及共用中继和 STUN，还会重新加载相关公共进程和已启用频道。
- **PVF 与资源**：在后台浏览 PVF 中的物品属性，上传自定义 PVF，或上传 `ImagePacks2` ZIP 更新物品图标。更换 PVF 后会重启运行中的频道。
- **登录器和客户端**：后台集中配置登录器标题、背景、整包地址与更新版本，以及客户端的服务器地址、等级上限、画质、自动拾取等选项，具体生效范围取决于配套客户端的实现。
- **数据库与密钥**：超管可浏览数据库、编辑数据并执行 SQL；也可上传匹配的公私钥对或重新生成密钥，并同步至服务。

平台与 DNFLogin 提供统一的管理和登录入口。当前集成镜像仍保留原网关进程，以兼容既有部署；使用平台管理不需要额外运行本地 GM 工具。

## 功能预览

### 角色、背包和邮件

![角色管理](doc/platform-screenshots/role-management.png)

![角色背包与快捷栏](doc/platform-screenshots/inventory-management.png)

![发送邮件与选择物品](doc/platform-screenshots/mail-items.png)

### PVF 和频道

![PVF 管理](doc/platform-screenshots/pvf-management.png)

![频道管理](doc/platform-screenshots/channel-management.png)

### 登录器和客户端

![登录器配置](doc/platform-screenshots/launcher-settings.png)

![版本管理](doc/platform-screenshots/launcher-versions.png)

![客户端配置](doc/platform-screenshots/client-settings.png)

### 数据库管理

![数据库管理](doc/platform-screenshots/database-management.png)

## 镜像

| 镜像 | 内容 | 已发布架构 |
| --- | --- | --- |
| `guoshengkai/dnf-server:dnf-integrated` | 服务、数据库与平台的一体化镜像 | `linux/amd64` |
| `guoshengkai/dnf-server:latest` | 独立平台镜像，需连接已有数据库 | `linux/amd64` |

```bash
docker pull guoshengkai/dnf-server:dnf-integrated
```

以下示例在 Linux 宿主机使用 host 网络；独立部署平台的示例见[平台项目 README](https://github.com/onlyGuo/dnf-server-public)。宿主机网络使容器共享宿主机网络栈，不需要 `-p` 映射；端口直接占用宿主机端口。详见 [Docker host 网络文档](https://docs.docker.com/engine/network/drivers/host/)。

## 部署集成服务

### 1. 安装宿主机守护服务

宿主机网络部署需要支持 systemd、cgroup v2 和 nftables socket cgroupv2 的 Linux。以下命令在本仓库根目录执行，默认容器名为 `dnf`：

```bash
INTEGRATION_DIR=build/Centos7-DNF/web/integration
sudo install -m 755 "$INTEGRATION_DIR/host-guard.sh" /usr/local/sbin/dnf-host-guard
sudo install -m 644 "$INTEGRATION_DIR/dnf-host-guard.service" /etc/systemd/system/dnf-host-guard.service
sudo systemctl daemon-reload
sudo systemctl enable --now dnf-host-guard.service
```

守护服务为该容器安装规则，限制内部端口；Web 登录及游戏端口不按玩家来源区分权限。启动容器时，守护规则未就绪会阻止游戏服务启动。修改容器名、配置数据库访问来源和排查要求，请参阅[集成部署说明](build/Centos7-DNF/web/integration/README.md)。

### 2. 配置并启动

准备持久化目录：

```bash
sudo mkdir -p /srv/dnf/data /srv/dnf/mysql /srv/dnf/log /srv/dnf/platform
```

创建仅供管理员读取的 `/etc/dnf-service.env`，将下面的占位值替换成实际配置。`DNF_DB_GAME_PASSWORD` 需要 8 位；原网关账号和密码不要使用特殊字符。

```dotenv
PUBLIC_IP=服务器可访问的IPv4地址
SERVER_GROUP=3
SERVER_GROUP_DB=cain
OPEN_CHANNEL=11,52,53,54,55,56
DNF_DB_ROOT_PASSWORD=替换数据库root密码
DNF_DB_GAME_PASSWORD=替换为8位密码
WEB_USER=替换Supervisor用户名
WEB_PASS=替换Supervisor密码
GM_ACCOUNT=替换原网关用户名
GM_PASSWORD=替换原网关密码
```

`PUBLIC_IP` 应填写玩家实际能连接的地址。也可使用原服务的 `DDNS_ENABLE=true` 和 `DDNS_DOMAIN` 配置动态域名，此时不要同时指定 `PUBLIC_IP`。启动后还能在平台管理默认通告地址和独立频道地址。

```bash
docker run -d \
  --name dnf \
  --hostname dnf \
  --restart unless-stopped \
  --network host \
  --cap-add NET_ADMIN \
  --shm-size 8g \
  -e DNF_HOST_NETWORK=true \
  --env-file /etc/dnf-service.env \
  -v /srv/dnf/data:/data \
  -v /srv/dnf/mysql:/var/lib/mysql \
  -v /srv/dnf/log:/home/neople/game/log \
  -v /srv/dnf/platform:/data/dnf-admin \
  guoshengkai/dnf-server:dnf-integrated
```

平台默认访问地址为 `http://服务器地址:8003/login`，登录器连接同一 Web 服务。`OPEN_CHANNEL` 用于首次导入频道配置；平台完成初始化后，使用持久化的频道配置，后续请在页面管理。

查看容器启动日志和运行中的进程：

```bash
docker logs -f dnf
docker exec dnf supervisorctl status
```

请确保宿主机内存和磁盘空间充足，并持久化和备份以上数据目录。升级或迁移已有服务时，应保留原有数据库、PVF 和密钥；不要直接执行会覆盖既有数据的初始化脚本。

## 开放端口

host 网络模式下，宿主机防火墙及云安全组或路由器转发都需要放行实际使用的端口。下表以 `SERVER_GROUP=3`、平台默认端口 8003，以及示例启用的频道为例：

| 用途 | 协议和端口 |
| --- | --- |
| 平台与登录器 Web 接口 | TCP 8003 |
| 游戏公共服务 | TCP/UDP 7001、TCP/UDP 7300、UDP 2311–2313 |
| 频道 11 | TCP 30011、UDP 31011 |
| 频道 52–56 | TCP 30052–30056、UDP 31052–31056 |

例如，使用 firewalld 的 `public` 区域时，容器启动并确认守护规则就绪后，可同时添加运行时和永久规则：

```bash
for port in 8003/tcp 7001/tcp 7001/udp 7300/tcp 7300/udp \
  2311-2313/udp 30011/tcp 31011/udp 30052-30056/tcp 31052-31056/udp; do
  sudo firewall-cmd --zone=public --add-port="$port"
  sudo firewall-cmd --permanent --zone=public --add-port="$port"
done
```

请按实际活动区域替换 `public`。以上命令不重载 firewalld；规则同时影响宿主机上使用这些端口的其他服务。[firewall-cmd 文档](https://firewalld.org/documentation/man-pages/firewall-cmd.html)介绍了区域和端口配置。

新增频道时，按大区号和频道号计算端口：TCP 为 `大区号 × 10000 + 频道号`，UDP 为 `大区号 × 10000 + 1000 + 频道号`。例如在 3 区新增 27 号频道，还需开放 TCP 30027 和 UDP 31027：

```bash
for port in 30027/tcp 31027/udp; do
  sudo firewall-cmd --zone=public --add-port="$port"
  sudo firewall-cmd --permanent --zone=public --add-port="$port"
done
```

别忘了同步配置云安全组或上游路由。默认宿主机守护规则覆盖频道 1、6、7、11–39 和 52–56；如果调整游戏版本或使用其他端口，还需相应调整守护规则。

原服务还包含 TCP 881 和 7600 网关端口。当前集成镜像保留原网关，但宿主机守护规则默认不对外放行这两个端口；确需使用旧网关时，需要结合部署方式调整守护规则及防火墙。数据库和 Supervisor 的兼容入口不在上面的公网开放示例中；来源配置见[集成部署说明](build/Centos7-DNF/web/integration/README.md)。

## 构建与其他资料

从服务项目构建集成镜像的操作、替换平台 JAR 和版本同步见[集成构建说明](build/Centos7-DNF/WEB_INTEGRATION.md)。原项目的[环境准备](doc/PrepareLinux.md)、[其他部署方式](doc/OtherDeploy.md)和[服务架构图](doc/ArchitectureDiagram.md)可供参考；其中旧镜像及网关示例不等同于本文的集成镜像。

上游相关社区项目：[XanderYe/dnf](https://github.com/XanderYe/dnf)、[llnut/dnf](https://github.com/llnut/dnf)。

原项目声明该项目仅供学习使用。

