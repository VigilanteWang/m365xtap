# Microsoft 365 跨租户日历忙闲策略

本文介绍 Microsoft 365 Cross-Tenant Access Policy（Microsoft 365 跨租户访问策略）下的 Free/Busy（忙闲信息）共享。

## 1. 从 Exchange 里的共享配置，走向身份与资源一起管理

上一篇讲到，Exchange 2007 的 Availability Service 开始按需查询忙闲，Exchange 2010 的 Federation 又帮助独立组织验证跨组织请求。邮箱搬到云端后，OrgRel 与 AAS 继续承担组织间共享的工作。

随着 Microsoft 365 的协作场景增多，微软推荐通过 **Microsoft 365 Cross-Tenant Access Policy**，将日历等资源的共享授权与 Entra 的跨租户访问设置一起管理。

这次迁移也与 EWS 有关。OrgRel 和 **Sharing Policy（共享策略）** 的相关共享依赖 **Exchange Web Services（EWS）**；微软正在移除 EXO 中的 EWS 依赖，并更新高权限访问模式。EXO 租户间 AAS 的 `OrgWideFBToken` 路径独立于 EWS，虽然可以继续提供基本忙闲，但微软仍推荐迁移获得更多功能。

## 2. Cross-tenant、B2B 与 Cross-Tenant Access Policy

一家公司使用 Microsoft 365 时，会有自己的 **tenant（租户）**，用来管理员工账号、应用和资源。两家公司各有一个租户，员工访问对方租户中的资源，就是 **cross-tenant（跨租户）** 访问。

- **B2B Collaboration（企业间协作）** 是 Microsoft Entra ID 提供的协作方式（其中一种）：合作公司的员工可以用自己公司的账号，访问其他公司授予他的应用或资源。自己的公司继续管理账号和登录，拥有资源的公司决定给他什么权限。[B2B Collaboration 概述](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)

- **Entra Cross-Tenant Access Policy（跨租户访问策略）** 则是双方管理员制定的访问规则：本公司哪些员工可以访问外部组织，哪些外部员工可以进入本公司，允许访问哪些应用，以及是否可以跨云访问。

- 策略分为 **Default settings（默认设置）** 和 **Organizational settings（组织设置）**：Default settings 提供通用规则，Organizational settings 为指定 **Organization（组织）** 设置专用规则，覆盖默认设置，没设置的则继承。[cross-tenant access settings](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-settings-b2b-collaboration)

- 设置有访问方向：**Inbound（入站）** 控制外部用户访问本组织资源，**Outbound（出站）** 控制本组织用户访问外部资源。来源租户的 outbound 和资源租户的 inbound 规则共同决定是否允许这次跨租户访问。

<p align="center">
  <img src="https://learn.microsoft.com/en-us/entra/external-id/media/cross-tenant-access-overview/cross-tenant-access-settings-overview.png" alt="Microsoft Entra 跨租户访问设置示意图：来源租户出站与资源租户入站规则" width="720" style="max-width: 100%; height: auto;">
</p>

这些设置大多可在 Azure Portal 访问，而在Graph API 里，由以下结构表示：

- crossTenantAccessPolicy 作为 tenant 唯一全局策略，跨云设置存放在此
- 然后关联 `default` 默认配置，以及用 `partners` 保存各 Organization 的配置

```text
crossTenantAccessPolicy                    tenant 唯一的 root 策略
├─ displayName                             root 策略显示名称
├─ allowedCloudEndpoints                   允许协作的 Microsoft 云，如commerical, 21v
├─ default                                 默认配置
│  ├─ 协作设置                              各种 inbound / outbound 策略及其它设置，如 trust MFA
│  └─ m365Capabilities                     默认开放的 M365 能力授权集合
└─ partners                                Organization 配置集合
   └─ {tenantId}                           某个 Organization 的配置
      ├─ tenantId                          Organization 的 Tenant ID
      ├─ 协作设置                           主要字段与 default 相同
      └─ m365Capabilities                  向该组织开放的 M365 能力授权集合
```
Microsoft 365 Cross-Tenant Access Policy 扩展了上述 Entra Cross-Tenant Access Policy，将 Exchange 的日历忙闲等功能，纳入 Cross-tenant B2B 体系。[M365XTAP策略概述](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)

## 3. Microsoft 365 Cross-Tenant Access Policy

### 3.1 先建立协作信任，再配置能力授权

Microsoft 365 Cross-Tenant Access Policy 通过两层设置管理跨组织访问：

**第一层是 `m365CollaborationInbound`**：与其他 Organization 建立 Microsoft 365 collaboration trust（Microsoft 365 协作信任），通过 `users` 数组，指定允许参与协作的**外部用户**。[协作信任前提](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta#microsoft-entra-cross-tenant-trust-prerequisite)

> 实际上文档中还写了 `appServiceConnectInbound` 建立应用连接信任，配合 `crossTenantMigration` capability， 专门给邮箱迁移用

**第二层是 `m365Capabilities`**：在协作信任建立的基础上，具体允许或拒绝日历忙闲、日历共享、MailTips 等能力，并可限定访问邮箱的范围。[Capability 定义](https://learn.microsoft.com/en-us/graph/api/resources/m365capabilitybase?view=graph-rest-beta)

这些设置由拥有资源的租户配置，且仅有 **inbound** 策略，双方可按需配置，决定向对方开放什么。[官方迁移流程](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#part-2--configure-your-new-microsoft-365-cross-tenant-access-policies)

### 3.2 两层怎样共同决定启用？

以 A 的 `alice@a.com` 查看 B 的 `bob@b.com` 为例。在 B 的 Organizational settings 中为 Organization A 配置规则：`m365CollaborationInbound` 允许 Alice，日历 capability 的资源范围覆盖 Bob。

```text
partners/{A-Tenant-ID}
├─ m365CollaborationInbound                   第一层：允许谁参与协作
│  └─ users                                  A 的 Alice 或允许的外部用户范围
└─ m365Capabilities                          第二层：开放什么资源能力
   └─ crossTenantCalendarAvailabilityBasic   能力名称
      └─ inboundAccess
         ├─ isAllowed: true                  开放功能；false 表示禁止
         └─ resourceScopes                   选择 B 的 Bob、Bob 所在组或全部用户
```

**两层授权需要同时满足。** 微软明确说明：缺少有效 m365CollaborationInbound，capability 即使已创建，运行时也不会生效。M365 capabilities 的系统默认是禁用。

### 3.3 上述例子的 Graph API 请求

下面在 B 租户中执行两个请求，实现上述日历忙闲查询：先允许 Organization A 的用户参与协作，再开放 Bob 所在组的基础忙闲。这里将请求者范围设为 A 的全部用户。

**1. 更新针对 Organization A 的 Partner 配置，建立协作信任，假设已有 B2B collaboration 配置**

```http
PATCH /beta/policies/crossTenantAccessPolicy/partners/{A-Tenant-ID}
Content-Type: application/json
```

```json
{
  "m365CollaborationInbound": {
    "users": {
      "accessType": "allowed",
      "targets": [
        { "target": "AllUsers", "targetType": "user" }
      ]
    }
  }
}
```

此请求修改 Graph API 中已存在的 Partner 配置；如果尚未创建，应先向 `/partners` 集合创建以 A Tenant ID 标识的配置。[Inbound 设置定义](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationinboundsetting?view=graph-rest-beta)、[更新 Partner 示例](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-update?view=graph-rest-beta)

**2. 向同一个 Partner 配置添加基础忙闲能力**

将 `resourceId` 替换为 B 租户中 Bob 所在组的 Object ID。

```http
POST /beta/policies/crossTenantAccessPolicy/partners/{A-Tenant-ID}/m365Capabilities
Content-Type: application/json
```

```json
{
  "@odata.type": "#microsoft.graph.crossTenantCalendarAvailabilityBasic",
  "inboundAccess": {
    "isAllowed": true,
    "resourceScopes": {
      "included": [
        {
          "resourceId": "<id of a group that Bob is memberof>",
          "resourceType": "group"
        }
      ],
      "excluded": []
    }
  }
}
```

创建时用 `@odata.type` 选择能力；后续更新时用返回对象的 `name` 定位 capability。[创建 capability](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-beta)

Basic 允许查询忙闲状态和时间，不返回会议详情。需要主题、地点时选择 `crossTenantCalendarAvailabilityLimitedDetails`。其余参考 [capability 列表](https://learn.microsoft.com/en-us/graph/api/resources/m365capabilitybase?view=graph-rest-beta)

## 4. 验证配置

功能部署到双方环境后，按以下步骤验证：

1. 与对方管理员约定测试时间，备份双方需要迁移的旧配置。
2. 禁用对应的 Organization Relationship（OrgRel）和 Sharing Policy。
3. Availability Address Space（AAS）无法临时禁用，需先导出备份，再移除对应配置，避免旧配置优先处理请求。
4. 用 Outlook 验证范围内用户的忙闲是否可见、范围外用户是否按预期受限；需要双向共享时，双方分别测试。
5. 若测试失败，重新启用旧策略，并从备份恢复 AAS。

具体操作参考[官方测试与回滚步骤](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#part-3--test-functionality)。

已确认 21V 环境尚未部署此功能，待功能部署后再迁移和验证；后续发布进度请关注 Exchange Team Blog：[Cross-tenant Free/Busy, MailTips, and Calendar Sharing are moving to Cross-Tenant Access Policy](https://techcommunity.microsoft.com/blog/exchange/cross-tenant-freebusy-mailtips-and-calendar-sharing-are-moving-to-cross-tenant-a/4545169)。

## 5. 文档一些待验证事项

相关文档仍有一些表述含混、衔接不足的地方；本文解释了配置模型，以下运行细节仍需微软补充说明或通过租户测试确认。

- **命名容易混淆。** Entra 文档主要使用 Cross-tenant access settings，Graph 使用 `crossTenantAccessPolicy`，新功能又称 Microsoft 365 Cross-Tenant Access Policy；同一组词跨越管理功能、API 资源和 M365 扩展，读者需要自行厘清各自指代。[Entra 概览](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)、[M365 概览](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)

- **Scope 的措辞存在出入。** 迁移指南说启用本组织全部用户的 collaboration trust，但 `m365CollaborationInbound.users` 的定义指向外部用户；指南还提到可缩小 trust scope，却未给出完整示例，容易与 capability 中的本地资源范围混淆。[迁移指南](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#check-your-entra-cross-tenant-access-policy-trust-relationship)、[Inbound 字段定义](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationinboundsetting?view=graph-rest-beta)

- **与 B2B Collaboration 的运行依赖未说透。** `m365CollaborationInbound` 与 `b2bCollaborationInbound` 在配置模型中并列，迁移步骤也只配置前者；但示例未设置 B2B 可能仍继承默认允许，不能据此确认“显式阻止 B2B、只允许 M365”时的实际结果。[迁移步骤](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#check-your-entra-cross-tenant-access-policy-trust-relationship)

- **Outbound 的实际作用缺少衔接说明。** 概览将 M365 capability 描述为 inbound-only，日历迁移步骤也只配置资源侧入站授权；Graph 却同时提供 `m365CollaborationOutbound`，没有清楚解释它与这些日历场景的关系，以及设为 blocked 是否会阻止查询。[M365 概览](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)、[Outbound 字段定义](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationoutboundsetting?view=graph-rest-beta)

- **邮箱迁移的信任前提存在歧义。** 概览先说所有 M365 capability 都需要 collaboration trust，随后又列出邮箱迁移需要 App service connect，却没有明确后者是额外前提还是例外路径；应用连接设置与 `crossTenantMigration` 的完整配合流程也缺少说明。[信任前提](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta#microsoft-entra-cross-tenant-trust-prerequisite)

- **配置方法已有，运行原理与排障依据仍不完整。** 已查阅的公开资料没有完整说明 Outlook 查询经过哪些服务、使用什么令牌、如何执行两层策略检查，以及如何确认请求实际走了新路径；配置保存成功、业务查询成功和新策略确实生效，仍需分别验证，仅凭现有指南难以定位所有失败原因。[官方配置与测试指南](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap)

下一篇展开管理请求和验收步骤：[21V raw HTTP 配置](03-21v-http-guide.md)。其中采用 Bob 查看 Alice 的方向，资源租户为 A；按第 3.2 节交换双方角色即可。上一篇：[Exchange Online 如何跨租户共享日历忙闲](01-exchange-current.md)。
