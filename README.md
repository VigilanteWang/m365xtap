# Microsoft 365 跨租户日历忙闲：原理与 21V 配置指南

核查日期：2026-10-13。面向具有基本 Microsoft Entra ID 知识的初级管理员。

本文档解释传统 Exchange 组织共享如何工作、Entra 跨租户策略怎样控制身份访问，以及如何在世纪互联运营的 Microsoft 365（21V）中，通过 raw HTTP 配置 `crossTenantCalendarAvailabilityBasic`。

## 阅读顺序

1. [传统 Exchange 组织共享与 Federation](docs/01-exchange-federation.md)：OrgRel、信任代理、SAML 委派、EWS，以及 A/B 组织的查询示例。
2. [从 Entra B2B 到 Microsoft 365 日历忙闲策略](docs/02-cross-tenant-architecture.md)：身份层与资源授权层、架构图、方向与范围、21V 支持边界。
3. [21V raw HTTP 配置与验证](docs/03-21v-http-guide.md)：令牌准备、创建 Partner、建立 Microsoft 365 collaboration trust、启用忙闲 capability、读回、验收、关闭与排错。

## 先明确结论

- **官方 API 已标注 21V 支持。** 创建 Partner 和创建、列出、更新、删除 Partner 的 Microsoft 365 capability 的 beta API 页面均将 China operated by 21Vianet 标为支持。本文采用用户指定的 `/beta` 文档版本；这不等于已经在实际租户完成部署。[创建 Partner](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicy-post-partners?view=graph-rest-beta)、[创建 capability](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-beta)。
- **只打开普通 B2B Collaboration 并不等于打开日历忙闲。** 新路径需要 Microsoft 365 collaboration trust，以及 `m365Capabilities` 中对应的日历授权。[Microsoft 365 API 概览](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)。
- **谁拥有日历，谁配置 inbound capability。** 想让 B 看 A，就在 A 中配置以 B Tenant ID 为键的 Partner；双向共享需要双方配置。[迁移指南](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#part-2--configure-your-new-microsoft-365-cross-tenant-access-policies)。
- **需核查租户实际 rollout。** 官方迁移指南仍有分批推出提示；Exchange 团队博客的 2026-09-29 更新也提醒功能尚未完全可用。不能仅根据支持矩阵承诺特定租户立即生效。[迁移指南](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap)、[Exchange 团队公告](https://techcommunity.microsoft.com/blog/exchange/cross-tenant-freebusy-mailtips-and-calendar-sharing-are-moving-to-cross-tenant-a/4545169)。
- **21V 的版本矩阵有差异。** 同一“创建 capability”操作的 v1.0 页面将 21V 标为不支持，beta 页面标为支持；不能直接把请求改成 `/v1.0`。Beta 文档提示接口可能变化且生产应用使用不受支持，正式变更需确认版本与 rollout。[v1.0 创建 capability](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-1.0)。

所有租户 ID、组 ID、邮箱地址均为示例或占位符。未对任何租户发送配置请求。配置示例经过官方接口、字段和 JSON 语法核查；运行时、跨云组合及客户端行为需由管理员按第三篇的验收步骤确认。
