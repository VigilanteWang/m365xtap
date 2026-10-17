# 跨租户日历忙闲共享：迁移到 Microsoft 365 Cross-Tenant Access Policy

本项目介绍 Exchange Online 中的跨租户日历忙闲共享，以及将现有共享配置迁移到 Microsoft 365 Cross-Tenant Access Policy 的背景、原理与配置方式。微软的[官方迁移指南](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap)涵盖 Free/Busy、日历共享和 MailTips，本项目重点讲解其中的 Free/Busy（忙闲信息）共享。

三篇文档依次讲解现有跨租户忙闲共享方式及工作原理、新策略的授权模型与创建方式，以及在世纪互联运营的 Microsoft 365（21V）中使用 Postman 进行配置的步骤。配套 Postman Collection 汇集所需的策略与日历能力 API，便于按需读取、创建和更新配置。

## 文件夹树

```text
m365xtap/
├── README.md                                 项目简介与阅读入口
├── docs/
│   ├── 01-exo-freebusy.md                    现有跨租户忙闲共享方式及原理
│   ├── 02-m365-xtap.md                       新策略原理、授权模型与创建方式
│   └── 03-21v-postman.md                     21V Postman 配置指南
├── postman/
│   ├── 21v_M365_XTAP.collection.json         API 请求集合（Read / Write）
│   └── 21v_M365_XTAP.environment.json        中国云端点与配置变量模板
└── LICENSE                                   项目许可证
```

## 如何阅读

建议按 **01 → 02 → 03** 阅读，从现有共享机制过渡到新策略，再进行配置操作。

| 文档                                                                   | 主要内容                                                                      |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [01 · Exchange Online 如何跨租户共享日历忙闲](docs/01-exo-freebusy.md) | 现有 OrgRel 与 AAS 共享方式、适用场景及查询与身份验证原理。                   |
| [02 · Microsoft 365 跨租户日历忙闲策略](docs/02-m365-xtap.md)          | 迁移背景、跨租户策略结构、协作信任与能力授权，以及创建策略的 Graph API 请求。 |
| [03 · 在 21V 中使用 Postman 配置日历忙闲共享](docs/03-21v-postman.md)  | 应用注册与权限、导入文件、填写变量、登录，以及配置写入与读回步骤。            |

已了解现有跨租户忙闲共享方式的读者可从 02 开始；使用配套 API 集合时，按 03 完成 Postman 准备与操作。

## 配套 Postman

将 [Collection](postman/21v_M365_XTAP.collection.json) 和 [Environment](postman/21v_M365_XTAP.environment.json) 导入 Postman 桌面客户端，选择 **21v_M365_XTAP** Environment，并按 [03](docs/03-21v-postman.md) 填写变量和登录。

Collection 按用途分为两组，按需选择请求并手动发送：

| 文件夹    | 请求                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------ |
| **Read**  | `Get Default Policy`<br>`Get Partner Policy`<br>`Get Default Capabilities`<br>`Get Partner Capabilities`           |
| **Write** | `Create Partner Policy`<br>`Update Partner Policy`<br>`Create Calendar Capability`<br>`Update Calendar Capability` |
