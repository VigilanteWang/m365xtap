# 在 21V 中使用 Postman 配置日历忙闲共享

本文使用配套 Postman Collection 配置基础日历忙闲共享，管理请求统一使用中国云端点和 Microsoft Graph `/beta` API。

## 1. 注册应用并配置权限

### 1.1 注册 public client 应用

在拥有待共享日历的 21V 租户中操作：

1. 打开 [Azure Portal](https://portal.azure.cn)，进入 **Microsoft Entra ID → App registrations → New registration**。
2. Name 填写 `Postman Calendar Basic`。
3. Supported account types 选择 **Accounts in this organizational directory only**，点击 **Register**。
4. 在 **Overview** 中记录 **Application (client) ID** 和 **Directory (tenant) ID**。
5. 进入 **Authentication → Add a platform → Mobile and desktop applications**，添加 Custom redirect URI：`https://oauth.pstmn.io/v1/browser-callback`。
6. 将 **Allow public client flows** 设为 **Yes**，保存。

应用使用 public client 和 Authorization Code with PKCE，无需创建 Client Secret。

### 1.2 添加权限

1. 进入 **API permissions → Add a permission → Microsoft Graph → Delegated permissions**。
2. 添加下表中的三项权限。
3. 由有权授予这些权限的管理员点击 **Grant admin consent**，确认已同意。

本流程使用 **Delegated permissions（委派权限）** 用户需具备对应的管理员角色。下表列出本流程的最低所需管理员角色

| 操作 | Graph permission | 最低所需管理员角色 |
| --- | --- | --- |
| 读取策略及日历 capabilities | `Policy.Read.All` | 按读取资源确定；日历 capabilities：Exchange Administrator |
| 创建或更新 Partner、配置协作信任 | `Policy.ReadWrite.CrossTenantAccess` | Security Administrator |
| 创建或更新日历 capability | `Policy.ReadWrite.CrossTenantCapability` | Exchange Administrator |

- **Security Administrator（安全管理员）**：负责 Partner 和 `m365CollaborationInbound` 协作信任。
- **Exchange Administrator（Exchange 管理员）**：负责日历 capability。
- 同一管理员完成全流程时，需同时具备上述两种角色，或使用 Global Administrator；分工操作时，分别使用对应管理员登录。

## 2. 导入 Collection 和 Environment

1. 下载两个文件：
   - [21v_M365_XTAP Collection](../postman/21v_M365_XTAP.collection.json)
   - [21v_M365_XTAP Environment](../postman/21v_M365_XTAP.environment.json)
2. 打开 Postman 桌面客户端，点击 **Import → Files**，导入这两个 JSON 文件。
3. 在右上角 Environment 选择器中选择 **21v_M365_XTAP**。

Collection 包含两个文件夹：

- **Read**：读取默认策略、Partner 配置及 capabilities。
- **Write**：创建或更新 Partner、协作信任及日历 capability。
- 按需选择请求、编辑 URL 或 Body，点击 **Send**；Collection 不包含自动执行或校验脚本。

## 3. 填写 Environment

打开 **Environments → 21v_M365_XTAP**，填写以下变量（* 表示需要填写或调整）：

| 变量 | 用途与填写内容 |
| --- | --- |
| `graph_base_url` | 中国云 Microsoft Graph API 的基础地址。 |
| `login_base_url` | 中国云 OAuth 登录服务的基础地址。 |
| `resource_tenant_id` * | 指定拥有待共享日历的租户。填写该租户的 Directory (tenant) ID。 |
| `partner_tenant_id` * | 指定获准访问日历的合作租户。填写合作租户的 Tenant ID。 |
| `client_id` * | 指定 Postman 登录使用的应用。填写第 1 节应用的 Application (client) ID。 |
| `capability_name` | 指定更新的日历 capability 名称。 |
| `calendar_is_allowed` | 控制是否启用日历忙闲授权。 |
| `resource_scope_type` * | 指定共享资源的类型。本文按组共享，填写 `group`。 |
| `resource_scope_id` * | 指定共享哪些用户的日历。填写资源租户中目标安全组的 Object ID。 |
| `oauth_callback_url` | 管理员登录完成后返回 Postman 的回调地址。 |
| `delegated_scopes` | 指定登录时申请的 Graph 委派权限及刷新令牌权限。 |
| `m365_inbound_access_type` | 控制是否允许入站 Microsoft 365 协作。 |
| `m365_inbound_target` | 指定允许参与协作的外部用户范围。 |
| `m365_inbound_target_type` | 指定外部协作对象的类型。 |

## 4. 在 Postman 中登录

1. 点击 Collection **21v_M365_XTAP → Authorization → Get New Access Token**。
2. 在浏览器中使用资源租户的管理员账号登录，完成 MFA。
3. 返回 Postman，点击 **Use Token**。

切换管理员时，重新获取并选择对应管理员的 Token。

## 5. 按上一篇的例子配置

沿用 [02 的例子](02-m365-xtap.md#33-上述例子的-graph-api-请求)：**Alice 查看 Bob 的基础忙闲**。本例假设双方均为 21V 租户，Bob 所在租户已有针对 Alice 所在租户的 Partner Policy，接下来更新协作信任并添加 Basic capability。

在 Bob 所在租户中操作，Environment 填写如下：

| 变量 | 填写内容 |
| --- | --- |
| `resource_tenant_id` | Bob 所在租户的 Tenant ID |
| `partner_tenant_id` | Alice 所在租户的 Tenant ID |
| `client_id` | 在 Bob 所在租户注册的应用 Client ID |
| `resource_scope_type` | `group` |
| `resource_scope_id` | Bob 所在安全组的 Object ID |

### 5.1 更新协作信任

由 Security Administrator 操作：

1. 打开 **Write → Update Partner Policy**。
2. 在 **Body** 中使用已预填的入站协作设置，允许 Alice 所在租户的全部用户参与协作。
3. 点击 **Send**，成功返回 `204 No Content`。
4. 执行 **Read → Get Partner Policy**，确认 `m365CollaborationInbound.users`：
   - `accessType` 为 `allowed`。
   - `targets` 中 `target` 为 `AllUsers`、`targetType` 为 `user`。

### 5.2 添加基础日历忙闲授权

由 Exchange Administrator 登录并选择 Token：

1. 打开 **Write → Create Calendar Capability**。
2. 在 **Body** 中确认：
   - `@odata.type` 为 `#microsoft.graph.crossTenantCalendarAvailabilityBasic`。
   - `isAllowed` 引用 `{{calendar_is_allowed}}`，值为 `true`。
   - `resourceId` 引用 `{{resource_scope_id}}`，值为 Bob 所在组的 Object ID。
   - `resourceType` 引用 `{{resource_scope_type}}`，值为 `group`。
3. 点击 **Send**，成功返回 `201 Created`。

### 5.3 读回并验证

1. 执行 **Read → Get Partner Capabilities**，确认 Basic capability 的 `isAllowed` 为 `true`，资源范围包含 Bob 所在组。
2. 功能部署到租户后，按 [02 的验证步骤](02-m365-xtap.md#4-验证配置)处理旧共享路径。
3. 让 Alice 在 Outlook 的 **Scheduling Assistant** 中查询 Bob，并测试组内、组外用户。

注意：

- API 保存成功不代表业务查询已生效；21V 租户仍需确认功能部署情况。
- Basic 只返回忙闲时间，不包含会议主题和地点。
- 本文使用 `/beta`，接口可能变化，微软不支持将其用于生产应用。

迁移、测试与回滚参考：[Migrate to Microsoft 365 Cross-Tenant Access Policy](https://learn.microsoft.com/en-us/exchange/sharing/migrate-to-m365-xtap)。
