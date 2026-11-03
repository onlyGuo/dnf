# 服务与平台集成

[DNF 服务器管理平台](https://github.com/onlyGuo/dnf-server-public)（简称平台）与本项目的游戏服务（简称服务）是独立项目。`web/dnf-admin.jar` 包含平台后端和管理页面，`web/integration` 包含服务启动适配资源；构建集成镜像不需要平台项目目录或平台镜像。

当前打包的平台源码提交：`2c493a07729957ed48f6a88dc5672935a867311e`。

已发布的集成镜像：

```bash
docker pull guoshengkai/dnf-server:dnf-integrated
```

如需从本项目构建，先按服务原有流程构建游戏镜像，再在项目根目录执行：

```bash
docker build --platform linux/amd64 -f build/Centos7-DNF/Dockerfile -t dnf-game:local .
docker build --platform linux/amd64 -f build/Centos7-DNF/Dockerfile.integrated \
  --build-arg DNF_IMAGE=dnf-game:local -t dnf-with-platform:local .
```

服务使用 x86，集成镜像应构建为 `linux/amd64`。`JAVA_IMAGE` 可指定兼容的 Java 21 镜像。更新平台时，将新构建的 JAR 和对应的集成脚本放入 `web/`，与服务项目一起提交；JAR 不应包含平台私钥。

桥接网络下平台在容器内默认监听 9001，宿主机网络模式下默认监听 8003。数据库、PVF、频道配置和密钥需要持久化。频道、PVF 与密钥管理的适用条件、网络和迁移步骤见 [集成部署说明](web/integration/README.md)。
