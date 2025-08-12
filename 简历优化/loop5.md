### 会话与 JWT 管理
- 技术栈：Spring Boot 3.2、Spring Security、JJWT(HS512)、Redis、MySQL、JPA、Docker
- 难点&方案：治理令牌生命周期（短Token+刷新），网关无状态鉴权支持横向扩展；Redis分布式会话+黑名单，登出即时失效；PBKDF2密码哈希，签名/过期/用户状态三重校验，登录链路打点与结构化日志。
- 结果：认证链路稳定可扩展、失效及时、可观测性强。

### 空间/团队管理（多租户）
- 技术栈：Spring Boot、JPA、MySQL、Redis
- 难点&方案：以 `Space/SpaceUser` 建模租户边界与成员（Owner/Admin/Member）；资源与审计全量绑定 `spaceId`；空间上下文贯穿读写与鉴权；高频查询分页与索引优化。
- 结果：低成本实现强隔离多租户，支撑团队协作与后续扩展。

### RBAC 权限与多租户
- 技术栈：Spring Security、JPA、MySQL、Redis
- 难点&方案：`Role/Permission/UserRole/RolePermission` 完整建模；按 Action/ResourceType/spaceId 细粒度鉴权；统一支持用户态JWT与API Key；高频校验路径加索引/缓存。
- 结果：达成最小权限策略与空间级隔离，权限治理清晰、可扩展。
