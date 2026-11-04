# DNF 一体化服务

基于 [1995chen/dnf](https://github.com/1995chen/dnf) 构建的 DNF 服务端 Docker 镜像。感谢原项目完成服务、数据库与初始化流程的容器化；本项目在此基础上集成了 [DNF 服务器管理平台](https://github.com/onlyGuo/dnf-server-public)，一个容器即可运行游戏服务、数据库和 Web 后台。

搭配 [DNFLogin 登录器](https://github.com/onlyGuo/DNFLogin)，可集中管理游戏运营、服务配置及客户端更新。

## 功能一览

- **运营管理**：管理账号、角色和属性，发送游戏邮件与物品，完成任务。
- **频道管理**：新增、编辑、启用、停用和重启频道，配置地下城、交易和决斗场等类型，查看内存占用。单频道修改无需重启整个容器。
- **动态地址**：默认通告地址和各频道可选择固定 IP 或 DDNS。
- **资源管理**：浏览 PVF 物品，上传新 PVF 更新服务，上传 `ImagePacks2` ZIP 更新图标。
- **背包管理**：查看和编辑物品、装备、背包及快捷栏。
- **登录器与客户端**：管理登录器背景、下载地址和版本更新，集中配置服务器地址、等级上限、画质和自动拾取等。
- **数据库与密钥**：图形化管理数据库，上传或生成游戏公私钥对。

![频道管理](doc/platform-screenshots/channel-management.png)

## 快速开始

在 Linux 宿主机安装 Docker 后，将以下地址和密码替换为自己的配置，然后运行：

```bash
mkdir -p /srv/dnf/data /srv/dnf/mysql /srv/dnf/log /srv/dnf/platform

docker run -d --name dnf --hostname dnf --restart unless-stopped \
  --network host --cap-add NET_ADMIN --shm-size 8g \
  -e DNF_HOST_NETWORK=true \
  -e PUBLIC_IP=服务器可访问的IPv4 \
  -e SERVER_GROUP=3 \
  -e SERVER_GROUP_DB=cain \
  -e OPEN_CHANNEL=11,52 \
  -e DNF_DB_ROOT_PASSWORD=替换数据库密码 \
  -e DNF_DB_GAME_PASSWORD=替换为8位密码 \
  -e WEB_USER=替换Supervisor用户名 \
  -e WEB_PASS=替换Supervisor密码 \
  -v /srv/dnf/data:/data \
  -v /srv/dnf/mysql:/var/lib/mysql \
  -v /srv/dnf/log:/home/neople/game/log \
  -v /srv/dnf/platform:/data/dnf-admin \
  guoshengkai/dnf-server:dnf-integrated
```

`DNF_DB_GAME_PASSWORD` 需要 8 位。首次启动后访问 `http://服务器地址:8003/login`；没有超管时，已有的第一个账号会被设为超管，空账号库的第一个注册账号会成为超管。完成启动后，可在平台配置频道及 DDNS。

`/srv/dnf` 保存数据库、PVF、频道配置、密钥和日志，请按需备份。`OPEN_CHANNEL` 只在首次导入时使用，后续在平台管理频道。

## 端口

以下为默认的 3 区配置：

| 用途 | 端口 |
| --- | --- |
| 平台 | TCP 8003 |
| 游戏公共服务 | TCP/UDP 7001、7300；UDP 2311–2313 |
| 频道 11 | TCP 30011、UDP 31011 |
| 频道 52 | TCP 30052、UDP 31052 |

新增频道时，3 区的 TCP 端口为 `30000 + 频道号`，UDP 端口为 `31000 + 频道号`。例如 27 号频道使用 TCP 30027 和 UDP 31027；请在宿主机防火墙及云安全组中按需放行。

数据库兼容端口为 TCP 3000，是否对外开放请根据自己的环境配置防火墙。

## 功能预览

<details>
<summary>角色、背包与游戏邮件</summary>

![角色管理](doc/platform-screenshots/role-management.png)
![角色背包与快捷栏](doc/platform-screenshots/inventory-management.png)
![发送邮件与选择物品](doc/platform-screenshots/mail-items.png)

</details>

<details>
<summary>PVF、登录器、客户端和数据库</summary>

![PVF 管理](doc/platform-screenshots/pvf-management.png)
![登录器配置](doc/platform-screenshots/launcher-settings.png)
![版本管理](doc/platform-screenshots/launcher-versions.png)
![客户端配置](doc/platform-screenshots/client-settings.png)
![数据库管理](doc/platform-screenshots/database-management.png)

</details>

## 项目与构建

- [DNF 服务器管理平台](https://github.com/onlyGuo/dnf-server-public)：可单独部署，也可与服务同机运行。
- [DNFLogin 登录器](https://github.com/onlyGuo/DNFLogin)：游戏登录与更新。
- [PVF/NPK 解析器](https://github.com/onlyGuo/dnf_parser)：游戏数据与图标解析。
- [集成构建说明](build/Centos7-DNF/WEB_INTEGRATION.md) · [服务集成说明](build/Centos7-DNF/web/integration/README.md)。

镜像发布架构为 `linux/amd64`。从旧版升级时，如曾安装宿主机守护服务，需停用该服务并清理其遗留规则。

原项目声明该项目仅供学习使用。
