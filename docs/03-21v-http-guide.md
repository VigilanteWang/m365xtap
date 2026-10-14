# 在 21V 中使用 raw HTTP 配置 Calendar Availability Basic

核查日期：2026-10-13。上一篇：[原理与架构](02-cross-tenant-architecture.md)。

## 1. 本文要完成的配置

**目标：让 B 的用户查看 A 中指定用户的基础日历忙闲。A 是 21V 租户，所有主流程请求都使用 A 的管理员身份。**

本文主例 B 也是 21V 租户。若 B 是全球商业云，先执行第 9 节的跨云准备，再返回主流程。双向共享见第 8 节。

流程：准备令牌 → 读取现状 → 创建 Partner → 配置 Microsoft 365 collaboration trust → 创建 Basic capability → 读回 → 用实际排程验证。

本文只提供请求模板，未连接实际租户。所有配置请求采用 `/beta`：创建 capability 的 beta 支持矩阵包含 21V，而其 v1.0 页面不包含 21V。Beta 接口存在变动和生产支持限制；API 支持矩阵也不能替代租户 rollout 验证。[beta 创建操作](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-beta)、[v1.0 创建操作](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-1.0)。

### 1.1 变量表：每个 ID 属于谁？

| 占位符 | 含义 | 所属租户 |
| --- | --- | --- |
| `{A_TENANT_ID}` | 拥有待共享日历的 Tenant ID | A |
| `{B_TENANT_ID}` | 允许访问的伙伴 Tenant ID，也是 Partner 的 URL 键 | B |
| `{CLIENT_ID}` | 用来获取管理令牌的应用 Client ID | 在 A 的中国云注册 |
| `{A_PROJECT_GROUP_ID}` | A 的协作项目安全组 Object ID；包含 Alice | A |
| `{A_TOKEN}` | A 租户签发、面向中国云 Graph 的管理 access token | A |

占位符要替换为真实值，包括 URL 和 JSON 中的值；花括号不属于实际请求。Tenant ID 是 Entra 租户 GUID，不是邮箱域、应用 Client ID 或用户 Object ID。组必须使用 Object ID，不使用显示名称。邮箱示例使用虚构的 `.example` 域。

### 1.2 中国云端点

| 用途 | 21V 端点 |
| --- | --- |
| Microsoft Graph | `https://microsoftgraph.chinacloudapi.cn` |
| 获取 OAuth 令牌 | `https://login.chinacloudapi.cn/{A_TENANT_ID}/oauth2/v2.0/...` |
| 本文策略 API 基路径 | `https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy` |

官方 Graph national cloud 文档说明中国云不支持 Graph Explorer，且不同云签发的 access token 不可互换。不要将全球云 `graph.microsoft.com` 的 URL 与中国云 token 混用。[National cloud deployments](https://learn.microsoft.com/en-us/graph/deployments)。

## 2. 权限与获取管理令牌

### 2.1 权限和角色是两道检查

本指南使用 delegated（管理员交互登录）路径。应用需获得下列 Microsoft Graph delegated permissions 的管理员同意：

| 操作 | Graph permission | 登录用户的角色 |
| --- | --- | --- |
| 读取政策现状 | `Policy.Read.All` | 按读取接口与资源要求分配角色；本流程由配置管理员读取 |
| 创建/更新 Partner、collaboration trust、cloud settings | `Policy.ReadWrite.CrossTenantAccess` | 使用 Security Administrator；Global Administrator 也可执行 |
| 创建/更新/关闭 Basic capability | `Policy.ReadWrite.CrossTenantCapability` | Exchange Administrator 或 Global Administrator |

可以由 Security Administrator 完成身份策略部分，再由 Exchange Administrator 完成日历授权。若单个管理员做完整演示，需要具备覆盖两部分的角色；不必为了日常分工长期使用 Global Administrator。

相关 API reference 还支持 application permissions，但那是无人值守应用路径，需另外配置并同意 application permissions，不能把 delegated 令牌与 app-only 令牌混为一谈。本指南不需要 `Calendars.Read`/`Calendars.ReadWrite`，因为这里管理的是策略，不是读取/修改管理员自己的日历。[Partner 创建权限](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicy-post-partners?view=graph-rest-beta)、[capability 创建权限](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-beta)、[Microsoft 365 角色说明](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)。

**资料表述差异：**Exchange 迁移指南使用 Global Administrator 作为 trust 配置前提，而当前 Graph/概览列出了 Security Administrator 等可支持角色。本指南按具体 API 的权限及角色要求说明；若受租户自定义角色或 rollout 影响，保留完整错误并核查对应操作，不要以 Exchange Administrator 的 capability 权限推断其可修改 Entra trust。

### 2.2 令牌准备前提

使用 A 中国云中已注册的管理用 public client 应用，启用适合 device code 的公共客户端身份验证，并完成上述 delegated permissions 的管理员同意。下列 HTTP 用于协议演示；应用注册和首次管理员同意应由组织既有身份管理流程完成。

若组织 Conditional Access 禁止 device code，应使用组织批准的交互认证流程，例如 authorization code + PKCE。Device code 不是绕过 MFA 或 Conditional Access 的方法。[Device code 协议](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)。

### 2.3 请求 device code

```http
POST https://login.chinacloudapi.cn/{A_TENANT_ID}/oauth2/v2.0/devicecode
Content-Type: application/x-www-form-urlencoded

client_id={CLIENT_ID}&scope=https%3A%2F%2Fmicrosoftgraph.chinacloudapi.cn%2FPolicy.Read.All%20https%3A%2F%2Fmicrosoftgraph.chinacloudapi.cn%2FPolicy.ReadWrite.CrossTenantAccess%20https%3A%2F%2Fmicrosoftgraph.chinacloudapi.cn%2FPolicy.ReadWrite.CrossTenantCapability
```

成功响应包含 `device_code`、`user_code`、`verification_uri`、`expires_in`、`interval`。打开响应给出的验证地址，输入 `user_code`，使用 **A 的管理员账号**完成登录及适用的 MFA。验证地址以实际响应为准。

请求体是表单，不是 JSON；实际工具应正确 URL-encode 参数。这里已编码 `scope` 中的 URL 与空格。[Device authorization request](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code#device-authorization-request)、[中国云身份端点](https://learn.microsoft.com/en-us/graph/deployments)。

### 2.4 轮询获取 token

```http
POST https://login.chinacloudapi.cn/{A_TENANT_ID}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adevice_code&client_id={CLIENT_ID}&device_code={URL_ENCODED_DEVICE_CODE}
```

至少等待返回的 `interval` 再轮询。`authorization_pending` 表示登录尚未完成；拒绝或过期时停止，并按协议重新开始。成功后，将响应中的 **`access_token`** 用作 `{A_TOKEN}`；不要把 `id_token` 当作 Graph access token。[Token 请求与响应](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code#authenticating-the-user)。

验证令牌请求针对 A、面向中国云 Graph，且返回 `scope` 包含当前操作所需权限。不在文档、工单或请求日志中粘贴完整 token。管理 token 与最终用户忙闲查询的身份上下文是不同用途。

## 3. 读取现状，保存变更前配置

### 3.1 读取根策略和默认策略

```http
GET https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy
Authorization: Bearer {A_TOKEN}
Accept: application/json
```

```http
GET https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/default
Authorization: Bearer {A_TOKEN}
Accept: application/json
```

```http
GET https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/default/m365Capabilities
Authorization: Bearer {A_TOKEN}
Accept: application/json
```

保存响应，特别是 `allowedCloudEndpoints`、`m365CollaborationInbound`、`m365CollaborationOutbound` 和默认 capabilities。Partner 中空属性可能继承这些默认值。本文不修改默认 capability，不将所有外部租户一并开放。[策略资源](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicy?view=graph-rest-beta)、[Default 资源及方法](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicyconfigurationdefault?view=graph-rest-beta)。

### 3.2 查询 B 的 Partner 是否已有配置

```http
GET https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}
Authorization: Bearer {A_TOKEN}
Accept: application/json
```

- `200 OK`：Partner 已存在，保存完整响应，跳过创建，执行第 4.2 节的定向 PATCH。
- `404`：先确认 cloud endpoint、版本、Tenant ID 和具体错误；若确为这个 Partner 不存在，再执行第 4.1 节。
- `401/403`：解决认证/授权问题，不应继续创建。

不要将任何 `404` 都解释为“租户不支持功能”；路径拼写、键不存在和 rollout 都需分别排查。[Partner 资源与 Get 方法链接](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicyconfigurationpartner?view=graph-rest-beta)。

## 4. 从新建 Partner 开始建立 collaboration trust

### 4.1 Partner 不存在：创建

这是按官方迁移指南的 payload 转为 raw HTTP 的最小基线：

```http
POST https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners
Authorization: Bearer {A_TOKEN}
Content-Type: application/json
Accept: application/json

{
  "tenantId": "{B_TENANT_ID}",
  "m365CollaborationInbound": {
    "users": {
      "accessType": "allowed",
      "targets": [
        {
          "target": "AllUsers",
          "targetType": "user"
        }
      ]
    }
  }
}
```

预期 `201 Created`，返回 Partner 对象。这里的 `tenantId` 是 **B**，但请求 token 属于 **A**。虽然 Partner 属性表将 tenantId 标为 read-only/key，创建接口和官方迁移示例仍通过请求体提供该值；创建后不能用 PATCH 将同一个 Partner 改成另一租户。

此配置允许 B 的全部用户作为 Microsoft 365 协作请求者；本请求没有创建 Basic capability。若默认策略已开放某些 capability，trust 生效后可能使继承的资源授权生效，所以应先检查第 3 节的默认基线。未在此 payload 中明确覆盖的属性继承默认设置。不要为本次忙闲配置加入 `AllApplications`、打开 B2B direct connect 或修改 MFA 信任。[创建 Partner](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicy-post-partners?view=graph-rest-beta)、[官方 trust 配置示例](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#check-your-entra-cross-tenant-access-policy-trust-relationship)。

### 4.2 Partner 已存在：更新 collaboration trust

**仅在确认这次要将已有 trust 的请求者范围改为 B 全部用户时执行。**如果已有 trust 只允许特定用户，应保留原范围或按获准范围修改，不要无意扩权。

```http
PATCH https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}
Authorization: Bearer {A_TOKEN}
Content-Type: application/json

{
  "m365CollaborationInbound": {
    "users": {
      "accessType": "allowed",
      "targets": [
        {
          "target": "AllUsers",
          "targetType": "user"
        }
      ]
    }
  }
}
```

预期 `204 No Content`。读取第 3.2 节端点，确认 `m365CollaborationInbound` 返回目标配置。PATCH 的外层仅包含本次需更新的属性，其他 Partner 属性保持原有行为。[更新 Partner](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-update?view=graph-rest-beta)。

**字段注意：**Inbound 使用 `users`，普通 B2B 设置使用 `usersAndGroups`，新的 M365 outbound 也使用 `usersAndGroups`。不能凭相似名称将这些 payload 相互替换。[Inbound 类型](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationinboundsetting?view=graph-rest-beta)、[Outbound 类型](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationoutboundsetting?view=graph-rest-beta)。

## 5. 为 Partner 启用 Basic Calendar Availability

### 5.1 先列出已有 capabilities

```http
GET https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}/m365Capabilities
Authorization: Bearer {A_TOKEN}
Accept: application/json
```

预期 `200 OK` 与 `value` 集合。保存完整结果，包括分页结果（如有 `@odata.nextLink`，跟随该链接）。检查是否已经有 `crossTenantCalendarAvailabilityBasic`；同时检查是否有 LimitedDetails 或其他会扩大暴露范围的 capability。已有 Basic 时使用第 5.3 节 PATCH，不重复 POST。[列出 capabilities](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-list-m365capabilities?view=graph-rest-beta)。

### 5.2 创建 Basic：只共享 A 项目组的忙闲

```http
POST https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}/m365Capabilities
Authorization: Bearer {A_TOKEN}
Content-Type: application/json
Accept: application/json

{
  "@odata.type": "#microsoft.graph.crossTenantCalendarAvailabilityBasic",
  "inboundAccess": {
    "isAllowed": true,
    "resourceScopes": {
      "included": [
        {
          "resourceId": "{A_PROJECT_GROUP_ID}",
          "resourceType": "group"
        }
      ],
      "excluded": []
    }
  }
}
```

预期 `201 Created`，返回创建后的 capability。`@odata.type` 必须提供；`isAllowed` 是 Boolean，不是字符串。`excluded` 使用合法空数组 `[]`，不是 `[{}]`。`resourceId` 是 **A 的组 Object ID**，表示该组成员的日历资源可向 B 共享。

返回对象中的 `name` 预期为 `crossTenantCalendarAvailabilityBasic`，并包含 `inboundAccess` 和 `lastModifiedDateTime`。保存实际返回值；时间戳由服务生成，不在创建请求中填写。本文根据 Basic 定义、通用创建操作及官方迁移映射组合请求，并未声称这是某个实际租户的响应。[创建操作](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-post-m365capabilities?view=graph-rest-beta)、[Basic 定义](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantcalendaravailabilitybasic?view=graph-rest-beta)。

### 5.3 Basic 已存在：更新同一 capability

```http
PATCH https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}/m365Capabilities/crossTenantCalendarAvailabilityBasic
Authorization: Bearer {A_TOKEN}
Content-Type: application/json

{
  "inboundAccess": {
    "isAllowed": true,
    "resourceScopes": {
      "included": [
        {
          "resourceId": "{A_PROJECT_GROUP_ID}",
          "resourceType": "group"
        }
      ],
      "excluded": []
    }
  }
}
```

预期 `204 No Content`。URL 最后一级使用服务返回的 capability `name`；本例按官方 Basic 类型名称给出。提交的是**本次完整期望的 inboundAccess**：应用前检查原 included/excluded，以免丢失既有排除名单。[更新 capability](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-update-m365capabilities?view=graph-rest-beta)。

### 5.4 经授权需要共享全部 A 用户时

保持上述 POST/PATCH 结构，只把 `resourceScopes` 换成：

```json
{
  "included": [
    {
      "resourceId": "All",
      "resourceType": "user"
    }
  ],
  "excluded": []
}
```

这是整个 A 租户的用户资源范围；`All` 区分大小写。不要写成 `AllUsers`，也不要将 `All` 理解为“只共享管理员本人”。若要排除 A 的保密组，将它加入 `excluded`；排除优先。[资源范围规则](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)。

## 6. 读回与业务验收

### 6.1 管理面读回

重发第 3.2 节 Partner GET、第 5.1 节 capability collection GET，以及 default 相关 GET，确认：

1. 请求发往中国云 Graph，令牌来自 A，Partner 键是 B。
2. 有效 `m365CollaborationInbound` 允许测试请求者。
3. Basic 的 `isAllowed` 为 `true`；included/excluded 与获准范围一致。
4. 组 ID 确属 A，测试资源所有者 Alice 是该组的有效成员。
5. 没有把默认策略或其他 capability 当作这次单一 Partner 的配置。

`201/204` 和读回正确说明管理面接受了配置；不能单凭这些响应宣布用户查询已经生效。没有公开可靠的统一生效时间承诺，本指南不指定等待若干分钟就必定成功。[rollout 提醒](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap)。

### 6.2 旧配置存在时，先隔离旧路径再验收

官方迁移指南说明旧 OrgRel、AAS、Sharing Policy 优先。对已有共享的环境，按以下顺序安排变更窗口：

1. 双方管理员盘点并备份对应旧配置，记录权限等级、范围、端点和恢复方式。
2. 只对本次迁移关系协调停用旧路径。OrgRel 可禁用；AAS 不能临时禁用，必须备份后移除；Sharing Policy 若涉及别的用户/功能，应先评估影响。
3. 使用新策略做正向、负向和精度测试。
4. 如失败，恢复旧配置并保留 API 响应、请求 ID、时间和测试条件。
5. 新路径验收通过后，再决定清理旧配置。

本指南的策略创建/更新全部使用 raw HTTP。**旧 Exchange 配置的盘点、停用和恢复由官方迁移指南的 Exchange 管理步骤处理**；这些步骤不是 `m365Capabilities` HTTP 的功能，本文不编造替代 Graph 端点。AAS 本身不依赖 EWS，不能把它当作必须立即删除的 EWS 路径。[测试与回滚](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap#part-3--test-functionality)。

### 6.3 用户面验收矩阵

使用双方现有 Exchange Online 邮箱，并在实际支持的 Outlook 排程体验中测试；记下客户端版本、云组合、UTC 时间及显示结果。

| 用例 | 配置/准备 | 预期与判断 |
| --- | --- | --- |
| B 的 Bob 查询 A 的 Alice | Alice 在 A 项目组中，测试时段有日历事件 | 可以看到基础忙闲时间 |
| Bob 查询 Alice 的会议细节 | Alice 创建有明显主题/地点的测试事件 | Basic 路径不应显示主题、地点、正文；若显示，排查旧路径或其他 capability |
| Bob 查询 A 的 Charlie | Charlie 不在 included，且不受其他有效开放配置覆盖 | 不应取得本策略范围外的忙闲 |
| A 的 Alice 查询 B 的 Bob | B 尚未向 A 配置对应开放，且无其他旧路径 | 不应因为 A 单向开放就自动获得反向访问 |
| 未被 trust 允许的外部请求者查询 | 使用另一个明确未获准的组织/身份，确认默认设置也未开放 | 不应通过本 Partner 获得忙闲 |

不能把“完全无会议的日历显示为空闲”当作唯一正向测试；也不能把客户端“无信息”的某一种显示文字作为唯一负向证据。测试应有已知忙碌事件，明确数据所有者与查询者，并排除旧共享路径。

管理接口的 GET `/m365Capabilities` 返回的是**策略**，不会返回 Alice 的忙闲数据。本文没有用 `/users/{Alice}/calendar` 或任意 `getSchedule` 请求充当新策略必然支持的端到端测试；普通 Graph 日历 API 的权限与跨租户策略是不同问题。

## 7. 关闭、删除与恢复

### 7.1 显式关闭 Basic

通常优先通过 `isAllowed: false` 关闭这一 capability，保留原范围。下面用主例完整范围：

```http
PATCH https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}/m365Capabilities/crossTenantCalendarAvailabilityBasic
Authorization: Bearer {A_TOKEN}
Content-Type: application/json

{
  "inboundAccess": {
    "isAllowed": false,
    "resourceScopes": {
      "included": [
        {
          "resourceId": "{A_PROJECT_GROUP_ID}",
          "resourceType": "group"
        }
      ],
      "excluded": []
    }
  }
}
```

预期 `204 No Content`；再列出 capabilities 读回。已有范围不同则填写原范围。只关闭 Basic 不代表其他 capability 或旧路径一并关闭。[更新操作](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-update-m365capabilities?view=graph-rest-beta)。

### 7.2 删除这个 capability

确认确实要移除对象时：

```http
DELETE https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy/partners/{B_TENANT_ID}/m365Capabilities/crossTenantCalendarAvailabilityBasic
Authorization: Bearer {A_TOKEN}
```

预期 `204 No Content`。**删除对象不应被当作可靠的禁止访问手段**：应重新核查默认 capability 的基线和其他有效配置。通常需要明确阻止时，保留 Partner 的显式关闭配置更容易审计。[删除操作](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-delete-m365capabilities?view=graph-rest-beta)、[默认/Partner 模型](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)。

### 7.3 恢复原则

若对象原先存在，通过 PATCH 恢复变更前的完整 inboundAccess；若本次新建且要撤销，按变更记录移除本次对象并核查默认基线。恢复 collaboration trust 前确认它是否还被 MailTips、Calendar Sharing 等其他能力使用。不要为撤销 Basic 直接删除整个 Partner，也不要顺带重置全租户默认策略。

## 8. 双向共享：双方重复同一流程

A 中允许 B 看 A 的配置完成后，要让 A 看 B，B 管理员用 **B 的 token** 执行相同流程：

| 项目 | A 中的配置 | B 中的配置 |
| --- | --- | --- |
| 当前 token | A_TOKEN | B_TOKEN |
| Partner 键 | B_TENANT_ID | A_TENANT_ID |
| capability | Basic | Basic |
| resourceScopes | A 的用户/组 | B 的用户/组 |
| Graph host | 中国云 | B=21V 使用中国云；B=全球商业云使用全球 Graph |

两侧不能共用 token。B 的设置不会替 A 决定 A 日历的公开范围，A 也不能用自己的管理令牌替 B 授权。[Inbound-only 语义](https://learn.microsoft.com/en-us/graph/api/resources/m365-cross-tenant-access-policy-overview?view=graph-rest-beta)。

## 9. 附加流程：A=21V、B=全球商业云

本节仅在**跨云**时执行。Entra 官方文档要求双方启用对方 Microsoft cloud，并以 Tenant ID 添加对方 Partner；这不要求双方都公开自己的日历。云级协作配置与 inbound 日历资源授权是两个层次。[跨云设置](https://learn.microsoft.com/en-us/entra/external-id/cross-cloud-settings)。

### 9.1 先读取双方现有 allowedCloudEndpoints

A 用中国云端点读取根策略；B 用 B 的全球云管理令牌读取：

```http
GET https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy
Authorization: Bearer {B_TOKEN}
Accept: application/json
```

`allowedCloudEndpoints` 是字符串数组。修改前将新值与已有数组合并；PATCH 不会替你执行集合追加。以下两个示例假定当前数组为空，**若已有值，必须保留获准的既有值**。[策略资源定义](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicy?view=graph-rest-beta)。

### 9.2 A 中国云允许全球商业云

```http
PATCH https://microsoftgraph.chinacloudapi.cn/beta/policies/crossTenantAccessPolicy
Authorization: Bearer {A_TOKEN}
Content-Type: application/json

{
  "allowedCloudEndpoints": ["microsoftonline.com"]
}
```

### 9.3 B 全球商业云允许中国云

```http
PATCH https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy
Authorization: Bearer {B_TOKEN}
Content-Type: application/json

{
  "allowedCloudEndpoints": ["partner.microsoftonline.cn"]
}
```

预期均为 `204 No Content`，并 GET 读回。这里的中国云枚举值是 `partner.microsoftonline.cn`；它不是中国云 Graph URL，也不是前面 token 请求的 `login.chinacloudapi.cn`。[更新策略请求示例](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicy-update?view=graph-rest-beta)。

### 9.4 添加双方 Partner，核查相应身份边界

A 返回第 4 节添加 B、配置 inbound collaboration trust。B 若尚无 A 的 Partner，创建一个继承默认设置的 Partner：

```http
POST https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners
Authorization: Bearer {B_TOKEN}
Content-Type: application/json
Accept: application/json

{
  "tenantId": "{A_TENANT_ID}"
}
```

预期 `201 Created`；B 已有 Partner 则只读取，不重复创建。双方检查有效 inbound/outbound 设置与组织允许的身份范围。普通 B2B guest 工作负载若也在范围内，按该工作负载的 B2B 设置配置；不要为了 Basic 日历能力统一放开全部 B2B 应用。

若 B 的有效 `m365CollaborationOutbound` 阻止本次参与者，应在 B 上按批准范围定向修改。以下仅演示允许 **B 的全部本地用户**对 A 进行 Microsoft 365 协作的字段形状，已有允许范围足够时无需执行：

```http
PATCH https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners/{A_TENANT_ID}
Authorization: Bearer {B_TOKEN}
Content-Type: application/json

{
  "m365CollaborationOutbound": {
    "usersAndGroups": {
      "accessType": "allowed",
      "targets": [
        {
          "target": "AllUsers",
          "targetType": "user"
        }
      ]
    }
  }
}
```

这不是向 A 公开 B 日历的 capability。若只要 B 看 A，到此不在 B 创建反向 Basic；若双向，再按第 8 节配置 B 的 inbound trust 和 Basic。[Outbound 类型](https://learn.microsoft.com/en-us/graph/api/resources/crosstenantaccesspolicym365collaborationoutboundsetting?view=graph-rest-beta)、[Partner 更新](https://learn.microsoft.com/en-us/graph/api/crosstenantaccesspolicyconfigurationpartner-update?view=graph-rest-beta)。

跨云组、目录搜索、客户端和工作负载限制需单独验证。本节建立 Entra cloud/Partner 配置的先决条件，不能单凭这些请求断言某个具体跨云忙闲场景已开放。优先向 21V 租户 Message Center/支持渠道核实 rollout，再做第 6 节的端到端测试。

## 10. 排错顺序

| 现象 | 优先检查 | 判断边界 |
| --- | --- | --- |
| token 获取失败 | Client ID 所属云、public client 配置、同意、登录角色、Conditional Access | 不是 calendar capability JSON 问题 |
| `401` | token 过期、Graph audience、云端点是否匹配 | 换正确令牌，不重建 Partner |
| `403` | permission、管理员同意、角色是否覆盖此操作、租户限制 | Exchange Admin 能配忙闲不代表能改 trust |
| `400` | 字段名、JSON、`@odata.type`、GUID、枚举值、云设置 | 保存服务返回的 error.code/message；不要按 HTTP 状态猜定原因 |
| 重复创建报错 | Partner/Basic 是否已存在 | GET 保存后改用 PATCH |
| GET/POST `404` | 完整 URL、beta、键、云支持与 rollout | 不能将任意 404 都认定为“租户未开放” |
| 配置读回正确，用户查不到 | trust、资源组、excluded、来源 outbound、客户端、rollout、跨云准备 | capability 对象存在不等于 runtime 已启用 |
| 仍能看到比 Basic 更详细信息 | 旧 OrgRel/AAS/Sharing Policy、LimitedDetails/其他能力、原有授权 | 不把旧路径结果当成 Basic 的效果 |
| 关闭 Basic 后仍能查询 | 旧路径、其他能力、默认基线、实际生效状态 | 关闭本能力不等于全面撤销所有共享 |
| 姓名搜不到但已知邮箱可查询 | GAL、Guest 可见性、联系人/目录同步 | Basic 不负责创建或同步通讯录 |

保留 HTTP 状态、error.code、error.message、服务返回的 request-id 和 Date，以及不含 token 的请求体。由于本次没有租户实测，不列出“必然出现”的精确错误代码，也不承诺统一缓存清除方法。

## 11. 交接记录模板

```text
变更日期（注明时区）：
A Tenant ID / 云：
B Tenant ID / 云：
目标方向：B → A / A → B / 双向
API 版本及当日 21V 支持矩阵：
租户 rollout 确认依据：
管理令牌类型、权限与角色（不记录 token）：
变更前 Partner/default/旧 Exchange 配置的保存位置：
trust 请求者范围：
capability / isAllowed：
本地 included / excluded：
管理面创建/更新状态、request-id、读回结果：
旧路径是否隔离，以及恢复方案：
用户面正向/负向/精度测试结果：
待核查事项：
```
