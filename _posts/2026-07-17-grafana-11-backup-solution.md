---
title: Grafana 11 备份方案
date: 2026-07-17 17:00:00 +0800
categories: [DevOps, 可观测]
tags: [grafana, backup, gdg, grafana-backup-tool, docker, postgres]
---

星图平台的生产 Grafana（11.6.7）需要一套可靠的备份方案。思路很简单：用社区工具通过 API 导出仪表盘、数据源、告警规则等资源，再配合数据库 dump 兜底。

踩了一圈发现，社区最知名的 Python 工具 `grafana-backup-tool` 在 Grafana 11 上多个 API 直接炸了。最终用 Go 写的 GDG 搞定。

## grafana-backup-tool：三连翻车

安装很简单，Docker 直接跑：

```bash
docker run --rm \
  -e GRAFANA_URL=https://your-grafana \
  -e GRAFANA_TOKEN=glsa_xxx \
  -v $(pwd)/backup:/_OUTPUT_ \
  ysde/docker-grafana-backup-tool
```

PyPI 上 1.4.2 就是这个工具的最新版，已经很久没更新了。跑起来之后，三个组件接连崩溃：

**alert-channels → 404 Not Found**

```
query alert channels failed, status: 404, msg: {'message': 'Not found'}
```

`/api/alert-notifications` 是 Grafana v8 时代旧告警的端点。v9 统一告警之后这个 API 已经不再暴露。工具没有适配。

**alert-rules → KeyError**

```
KeyError: 'Unable to get version, returned respone: <bound method Response.json of <Response [200]>>'
```

工具解析 Grafana 版本号的逻辑跟 v11 返回格式不匹配，一个内部异常导致整个流程退出。

**dashboard-versions → 类型断言错误**

```
TypeError: string indices must be integers, not 'str'
```

解析 `/api/dashboards/uid/xxx/versions` 返回值时把对象当数组用了，又是版本兼容性问题。

基础的三件套（folders / dashboards / datasources）倒还正常，16 个 Dashboard 和 6 个 Datasource 都导出了。

## GDG：除了文件名有点丑，都能用

GDG（Grafana Dash-n-Grab）是 ESnet 用 Go 写的工具，最新版 0.9.3，对 Grafana 11 兼容得很好。

macOS 上安装：

```bash
# brew 没有收录 cask，用 go install
go install github.com/esnet/gdg/cmd/gdg@latest
```

配置文件 `gdg.yaml`：

```yaml
contexts:
  production:
    url: https://your-grafana
    token: glsa_xxx
    output_path: backup
    dashboard_settings:
      ignore_filters: true
    watched: []
global:
  ignore_ssl_errors: false
```

备份命令：

```bash
# 按组件逐个导出
gdg -c gdg.yaml --context production backup dashboards download
gdg -c gdg.yaml --context production backup connections download
gdg -c gdg.yaml --context production backup folders download
gdg -c gdg.yaml --context production backup alerting rules download
```

结构按 Grafana 文件夹分类存放，比 grafana-backup-tool 的 UID 平铺直观：

```
backup/org_unknown/
├── connections/
│   ├── prometheus.json
│   ├── elasticsearch.json
│   └── ...
├── dashboards/
│   ├── 星图 - 统一大盘/
│   │   ├── f0c9268.json
│   │   └── ...
│   ├── 星图 - 独立看板/
│   │   ├── ai-网关-业务监控.json
│   │   └── ...
│   └── ...
└── folders/
    ├── 星图 - 统一大盘.json
    └── ...
```

注意事项：

- GDG 把 datasources 重命名成了 connections（v0.8 的 breaking change），配置文件里叫 `connections` 就行
- Alert rules 的 auth 路径有个坑：工具先找 `backup/secure/auth_production` 文件，找不到才 fallback 到 config 里的 token。warning 可以忽略
- 文件名是 URL 编码的中文，不够可读，但不影响恢复
- Dashboard 文件名用 hash 命名（`f0c9268.json`），同样不影响功能

## 数据库层备份

两个工具都只覆盖了 API 资源，用户、团队、组织关系、API Key 这些都在 Postgres 里。加上数据库备份才算完整：

```bash
# 每日备份
pg_dump -U grafana -h your-pg-host grafana | gzip > grafana_db_$(date +%Y%m%d).sql.gz

# 恢复
gunzip -c grafana_db_20260717.sql.gz | psql -U grafana -h your-pg-host -d grafana
```

## 方案结论

| | grafana-backup-tool 1.4.2 | GDG 0.9.3 |
|---|---|---|
| Grafana 11 兼容 | 三个模块崩溃 | 全部正常 |
| Dashboards | 正常 | 正常 |
| Alert Rules | 崩溃 | 正常 |
| Alert Contacts | 404 | 正常 |
| Dashboard Versions | 崩溃 | 不支持 |
| 文件结构 | UID 平铺 | 按文件夹分类 |

生产环境建议用 GDG 导出 API 资源，配合 pg_dump 做数据库全量备份，两边互补。API 备份解决细粒度恢复（只恢复某个被误删的仪表盘），数据库备份解决整实例灾难恢复。
