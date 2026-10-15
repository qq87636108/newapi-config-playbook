# PAT（面板令牌）程序化创建配方——三步全自动

> new-api 官方只写了「在面板个人设置里手工生成」。
> 本文为**逆向前端 + 实测验证**所得（2026-10-15，v1.0.0-rc.44），全网首份公开。
> 适用场景：自动化脚本需要管理凭据，用户只需提供一次管理员账密。

## 为什么值得折腾

- 账号密码会话：15 分钟过期 + 每次登录留一条永不回收的记录（50 条上限，约 17h 塞满后所有自动化 409 静默失败）；
- PAT：长效、权限可收窄、可吊销，是自动化正路。

## 前置

管理员账号登录拿临时会话（仅用于本次创建流程）：

```
POST /api/user/login  {"username":"...","password":"..."}
→ data.access_token
```

## 第 1 步：查验证方式（可选）

```
GET /api/verify/methods?scope=access_token.generate
→ {"data":{"methods":[{"method":"password","available":true}]}}
```

## 第 2 步：安全验证，换一次性 proof_token

```
POST /api/verify
Authorization: Bearer <access_token>
{
  "method": "password",
  "scope": "access_token.generate",
  "password": "<管理员密码>",
  "context": {
    "scopes": ["channel:read","channel:write","channel:operate",
               "channel:sensitive_write","option:read","option:write",
               "log:read","model:read"],
    "expires_at": null
  }
}
→ data.proof_token
```

⚠️ **最大的坑**：`context.scopes` 必须与第 3 步 token 的 scopes **完全一致**，不一致第 3 步直接被拒。

## 第 3 步：创建 PAT

```
POST /api/user/access_tokens
Authorization: Bearer <access_token>
X-Security-Proof: <proof_token>
{"name":"my-automation","scopes":[...同上],"expires_at":null}
→ data.token   // nap_ 开头，47 字符
```

之后所有管理接口都用 `Authorization: Bearer nap_...`。
缺权限时接口会明确报 `lacks channel:xxx`，按需补 scope 重建即可。

## scope 速查（GET /api/user/access_tokens/catalog）

| 组 | scope | 用途 |
|---|---|---|
| admin | channel: read / operate / write / sensitive_write | 渠道查/启停/改/建删（含 key 需 sensitive_write） |
| admin | log: read | 看门狗读错误日志 |
| admin | model: read | 模型对账 |
| system | option: read / write | 读写网关配置 |
| personal | profile/api_key/... | 个人域，自动化一般不需要 |

## 免责

非官方文档；接口行为以实测版本为准（rc.44），官方变更请用第 1 步的 methods 探测自行复核。
