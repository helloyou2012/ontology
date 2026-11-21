# Palantir Foundry Ontology(本体)系列笔记

基于 Palantir 官方文档(palantir.com/docs/foundry)整理,整理时间:2026-11-21。

## 目录

| 文件 | 内容 |
|---|---|
| [01-概念与定位.md](01-概念与定位.md) | Ontology 是什么、核心要素、与数据集的类比、类型体系 |
| [02-架构与后端.md](02-架构与后端.md) | OMS / OSS / Funnel / OSv1 与 OSv2、索引、写回、对象集、Ontology SQL、Peering |
| [03-建模.md](03-建模.md) | 对象类型、属性、链接类型、接口、共享属性、值类型、状态生命周期 |
| [04-动能层.md](04-动能层.md) | 动作、函数、Automate 自动化、Scenarios 情景 |
| [05-治理与交付.md](05-治理与交付.md) | 权限与安全策略、Global Branching 与提案、Marketplace / DevOps |
| [06-应用与集成.md](06-应用与集成.md) | Object Explorer、Workshop、Object Views、Vertex、OSDK、OMCP、AIP |
| [07-限制速查表.md](07-限制速查表.md) | 各类上限、不支持项、注意事项汇总 |

## 阅读建议

- 第一次接触:按 01 → 03 → 04 顺序读。
- 做架构或选型:重点读 02、05、07。
- 做应用开发:重点读 04、06。

## 说明与局限

- 本系列是对官方文档的**转述性整理**,不是原文翻译;具体配置步骤、界面和最新数值以官方文档为准。
- 标注 **(Beta)** 的功能可能需要向 Palantir 申请开通,行为可能变化。
- 标注 **(社区)** 的内容来自 Palantir 社区或第三方文章,不属于官方文档的正式承诺。
- 部分数值在不同文档页面间存在新旧差异(例如 Search Around 的默认上限),已在 07 中并列说明。
- 官方文档更新频繁,使用前建议核对对应页面。

## 官方入口

- Ontology 总览:`palantir.com/docs/foundry/ontology/overview/`
- Ontology 架构:`palantir.com/docs/foundry/object-backend/overview/`
- 对象与链接类型:`palantir.com/docs/foundry/object-link-types/`
- 动作:`palantir.com/docs/foundry/action-types/`
- 函数:`palantir.com/docs/foundry/functions/overview`
- 分支:`palantir.com/docs/foundry/foundry-branching/`
- OSDK:`palantir.com/docs/foundry/ontology-sdk/overview`
