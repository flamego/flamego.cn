# flamego.cn

[![Better Uptime Badge](https://betteruptime.com/status-badges/v1/monitor/dg05.svg)](https://betteruptime.com/?utm_source=status_badge)

本仓库为中文文档站点 https://flamego.cn 的源码，站点使用 [VuePress](https://v2.vuepress.vuejs.org/) 作为生成器，并基于 [GitHub Actions](.github/workflows/sync.yml) 实时同步到 [CODING](https://coding.net/) 仓库以通过 Webhook 触发[云开发 Webify](https://webify.cloudbase.net/) 进行部署。

运行以下命令启动本地预览实例：

```
yarn add -D vuepress@next
yarn add -D @vuepress/plugin-docsearch@next
yarn docs:dev
```

## 获取帮助

- 请直接在 [flamego/flamego](https://github.com/flamego/flamego) 仓库[提交工单](https://github.com/flamego/flamego/issues)或[发起讨论](https://github.com/flamego/flamego/discussions/87)

## 授权许可

本仓库内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.zh) 授权。
