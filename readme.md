# Docker Image Mirror

该仓库使用 GitHub Actions 定时将公开的第三方容器镜像同步到 Docker Hub，
方便只能访问 Docker Hub 的设备拉取。

## 免责声明

- 本仓库仅对上游公开镜像进行自动同步。
- 镜像软件版权、商标及许可证归原作者或原项目所有。
- 本仓库与上游项目不存在官方隶属关系。
- 请优先参考上游项目文档，并遵守相应软件许可证。
- 同步镜像可能与上游存在时间差，不保证实时性与可用性。
- 镜像未经功能修改，仅增加来源和同步信息标签。

## 同步时间

默认每天北京时间凌晨 03:00 自动执行一次，也可在 GitHub Actions 页面手动运行。

## 镜像列表

镜像映射关系见 [`images.json`](./images.json)。

## 使用方式

```bash
docker pull DOCKERHUB_USERNAME/IMAGE_NAME:TAG
