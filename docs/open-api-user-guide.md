# OpenAPI 用户接入与调用操作手册

本文面向需要调用平台对外开放接口的业务用户。按照本文步骤，可以完成 AK/SK 凭证准备、请求签名、接口调用、分页处理、失败重试和常见错误排查。

> 当前开放网关仅支持 **GET** 请求，采用 **AK/SK + HMAC-SHA256** 鉴权。生产环境必须使用 HTTPS。

---

## 1. 接入前需要准备的内容

请先向平台管理员获取以下信息：

| 项目 | 说明 | 示例 |
|---|---|---|
| 接口基础地址 | 平台分配的 HTTPS 域名，不包含具体接口路径 | `https://api.example.com` |
| 开放接口路径 | 必须以 `/openapi/v1/` 开头 | `/openapi/v1/institutions/top-100-by-heat` |
| 接口参数说明 | 参数名称、是否必填、允许值和业务含义 | 以管理员提供的接口清单为准 |
| Access Key（AK） | 请求身份标识，可以放在请求头中 | `ak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| Secret Key（SK） | 签名密钥，仅在创建或轮换时返回一次 | `sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| 签名版本 | 当前固定为 `v1` | `v1` |

### 1.1 获取凭证

登录平台后，在“开放接口 → AK/SK 管理”页面自助创建凭证：

1. 点击“创建凭证”，填写便于识别的凭证名称，可选择过期时间。
2. 创建成功后立即保存本次返回的 AK、SK 和签名版本 `v1`。
3. SK 仅在创建或轮换时返回一次，凭证列表不会再次返回 SK。
4. 确认管理员已经为对应用户等级配置接口调用策略，并将目标接口设置为“已开放”。

用户只能查看和管理自己的凭证。管理员仍可以统一管理所有用户的凭证，必要时也可以由管理员代为创建、轮换或禁用。

如果调用时返回“接口不存在或未开放”或“当前等级未配置接口权限”，应联系管理员检查配置，而不是反复重试。

### 1.2 凭证安全要求

- AK 可以标识身份，但仍不应随意公开。
- SK 等同于调用密码，不得提交到 Git、写入前端页面、放入 App 安装包或发送到群聊。
- 建议将 SK 保存在服务端密钥管理系统或环境变量中。
- 签名必须由用户的服务端完成，不能由浏览器或移动端直接持有 SK 后签名。
- 怀疑 SK 泄露时，应立即在 AK/SK 管理页面轮换或禁用凭证，也可以联系管理员处理。轮换后旧 SK 立即失效。

---

## 2. 一次完整调用的流程

每次请求都按以下顺序处理：

1. 确定请求路径和查询参数。
2. 获取当前 Unix 秒级时间戳。
3. 为本次请求生成全新的随机 nonce。
4. 按本文规则生成规范查询字符串。
5. 拼接待签名串。
6. 使用 SK 计算 HMAC-SHA256，并转成 64 位十六进制字符串。
7. 将 AK、时间戳、nonce、签名版本和签名放入请求头。
8. 发送不带请求体的 HTTPS GET 请求。
9. 检查 HTTP 状态码、业务 `code` 和响应数据。
10. 失败时记录 `X-OpenApi-Request-Id`，再按错误类型处理。

---

## 3. 请求规范

### 3.1 请求方法和请求体

- 仅支持 `GET`。
- 不要发送请求体。
- 不要设置 `Transfer-Encoding`。
- 参数通过 URL 查询字符串传递。
- 路径必须使用管理员分配的 `/openapi/v1/**` 对外路径，不能替换成平台内部路径。

### 3.2 必填请求头

| 请求头 | 必填 | 规则 |
|---|---|---|
| `X-OpenApi-Access-Key` | 是 | 管理员分配的 AK |
| `X-OpenApi-Timestamp` | 是 | 10 位 Unix 秒级时间戳，不是毫秒；与服务器时间偏差不能超过 300 秒 |
| `X-OpenApi-Nonce` | 是 | 每次请求唯一，长度 16～64，只允许字母、数字、下划线和短横线 |
| `X-OpenApi-Sign-Version` | 是 | 当前固定为 `v1` |
| `X-OpenApi-Signature` | 是 | 使用 SK 计算出的 64 位 HMAC-SHA256 十六进制签名 |

nonce 在平台保留期内不能重复。建议每次请求都重新生成，不要基于固定值递增，也不要在重试时复用。

### 3.3 建议读取的响应头

| 响应头 | 说明 |
|---|---|
| `X-OpenApi-Request-Id` | 本次请求的唯一标识。排查问题时应提供给管理员 |
| `Retry-After` | 用户或系统处于临时封禁状态时返回，单位为秒；应等待指定时间后再试 |

---

## 4. 签名算法详细步骤

### 4.1 第一步：生成时间戳和 nonce

时间戳使用当前 Unix 时间，单位为秒。例如：

```text
1735689600
```

nonce 示例：

```text
9f3ab86214c44d02b7f92e51836ad0a1
```

每次发送请求，包括失败后的重新发送，都要重新生成时间戳和 nonce，并重新计算签名。

### 4.2 第二步：生成规范查询字符串 CanonicalQuery

如果没有查询参数，`CanonicalQuery` 是空字符串，但在待签名串中仍必须保留这一行。

如果有查询参数，按以下规则处理：

1. 从实际准备发送的 URL 中取得原始查询字符串，不包含开头的 `?`。
2. 按 `&` 拆分参数，不允许出现空参数，例如 `a=1&&b=2`。
3. 每个参数按第一个 `=` 拆分名称和值；没有 `=` 时，值按空字符串处理。
4. 对名称和值先进行 URL 解码，其中 `+` 按空格处理。
5. 再按 RFC 3986 使用 UTF-8 编码：
   - 空格编码为 `%20`，不能编码为 `+`。
   - 波浪号保持为 `~`。
   - 其他需要编码的字符使用大写十六进制百分号编码。
6. 先按编码后的参数名升序排序；参数名相同时，再按编码后的参数值升序排序。
7. 使用 `名称=值` 格式，通过 `&` 连接。

例如实际发送的原始查询字符串为：

```text
b=2&a=hello+world&a=%7E
```

规范化结果为：

```text
a=hello%20world&a=~&b=2
```

**重要：签名必须使用实际发送的原始查询参数计算。签名完成后，不要再修改、重排或重新编码 URL。**

### 4.3 第三步：计算空请求体摘要

当前只允许无请求体的 GET 请求，因此固定计算空字符串的 SHA-256，并使用小写十六进制表示：

```text
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### 4.4 第四步：拼接待签名串

待签名串由以下 8 行组成，每行之间使用一个换行符 `\n`：

```text
OPENAPI-HMAC-SHA256
v1
GET
请求路径
CanonicalQuery
时间戳
nonce
空字符串SHA256
```

规则说明：

- 请求方法使用大写 `GET`。
- 请求路径只包含路径，不包含协议、域名和查询字符串。
- 无查询参数时，`CanonicalQuery` 所在行为空行。
- 不要在待签名串末尾额外增加换行符。
- 所有内容按 UTF-8 编码。

假设：

```text
请求路径：/openapi/v1/institutions/top-100-by-heat
查询参数：无
时间戳：1735689600
nonce：9f3ab86214c44d02b7f92e51836ad0a1
```

待签名串为：

```text
OPENAPI-HMAC-SHA256
v1
GET
/openapi/v1/institutions/top-100-by-heat

1735689600
9f3ab86214c44d02b7f92e51836ad0a1
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### 4.5 第五步：计算签名

使用以下参数计算：

```text
算法：HMAC-SHA256
密钥：SK 原文的 UTF-8 字节
内容：待签名串的 UTF-8 字节
输出：64 位十六进制字符串
```

注意：

- 不需要先对 SK 做 Base64 解码。
- 最终签名使用十六进制，不使用 Base64。
- 建议输出小写十六进制。
- SK 只用于本地计算，绝不能放入 URL、请求体或响应日志。

---

## 5. 完整 Python 调用示例

以下示例只使用 Python 标准库，不需要安装第三方依赖。建议使用 Python 3.9 或更高版本。

将代码保存为 `openapi_client.py`：

```python
import hashlib
import hmac
import os
import secrets
import time
import urllib.error
import urllib.parse
import urllib.request


BASE_URL = os.environ["OPENAPI_BASE_URL"].rstrip("/")
ACCESS_KEY = os.environ["OPENAPI_ACCESS_KEY"]
SECRET_KEY = os.environ["OPENAPI_SECRET_KEY"]

# 修改为管理员分配的接口路径。
PATH = os.environ.get(
    "OPENAPI_PATH",
    "/openapi/v1/institutions/top-100-by-heat",
)

# 不含开头的 ?；没有查询参数时保持为空。
RAW_QUERY = os.environ.get("OPENAPI_QUERY", "")


def rfc3986_encode(raw_value: str) -> str:
    """先 URL 解码，再按 RFC 3986 使用 UTF-8 编码。"""
    decoded = urllib.parse.unquote_plus(raw_value, encoding="utf-8")
    return urllib.parse.quote(
        decoded,
        safe="-._~",
        encoding="utf-8",
        errors="strict",
    )


def canonical_query(raw_query: str) -> str:
    if not raw_query:
        return ""

    parameters = []
    for pair in raw_query.split("&"):
        if not pair:
            raise ValueError("查询参数中不能出现空参数")

        raw_name, separator, raw_value = pair.partition("=")
        if not separator:
            raw_value = ""

        parameters.append((
            rfc3986_encode(raw_name),
            rfc3986_encode(raw_value),
        ))

    parameters.sort(key=lambda item: (item[0], item[1]))
    return "&".join(f"{name}={value}" for name, value in parameters)


def build_signature(path: str, raw_query: str, timestamp: str, nonce: str) -> str:
    empty_body_sha256 = hashlib.sha256(b"").hexdigest()
    canonical_request = "\n".join([
        "OPENAPI-HMAC-SHA256",
        "v1",
        "GET",
        path,
        canonical_query(raw_query),
        timestamp,
        nonce,
        empty_body_sha256,
    ])

    return hmac.new(
        SECRET_KEY.encode("utf-8"),
        canonical_request.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()


def call_open_api():
    timestamp = str(int(time.time()))
    nonce = secrets.token_hex(16)  # 32 位随机十六进制字符串
    signature = build_signature(PATH, RAW_QUERY, timestamp, nonce)

    # 签名完成后，URL 中使用的仍然是同一份原始查询字符串。
    url = BASE_URL + PATH
    if RAW_QUERY:
        url += "?" + RAW_QUERY

    request = urllib.request.Request(url, method="GET")
    request.add_header("X-OpenApi-Access-Key", ACCESS_KEY)
    request.add_header("X-OpenApi-Timestamp", timestamp)
    request.add_header("X-OpenApi-Nonce", nonce)
    request.add_header("X-OpenApi-Sign-Version", "v1")
    request.add_header("X-OpenApi-Signature", signature)

    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            body = response.read().decode("utf-8")
            print("HTTP 状态:", response.status)
            print("请求 ID:", response.headers.get("X-OpenApi-Request-Id"))
            print("响应内容:", body)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8")
        print("HTTP 状态:", error.code)
        print("请求 ID:", error.headers.get("X-OpenApi-Request-Id"))
        print("Retry-After:", error.headers.get("Retry-After"))
        print("错误内容:", body)


if __name__ == "__main__":
    call_open_api()
```

### 5.1 配置凭证并调用

将域名和凭证替换成实际值：

```bash
export OPENAPI_BASE_URL="https://api.example.com"
export OPENAPI_ACCESS_KEY="ak_替换为实际AK"
export OPENAPI_SECRET_KEY="sk_替换为实际SK"

python3 openapi_client.py
```

如果需要调用其他接口或带查询参数：

```bash
export OPENAPI_PATH="/openapi/v1/替换为实际接口路径"
export OPENAPI_QUERY="pageNum=1&pageSize=10"

python3 openapi_client.py
```

`OPENAPI_QUERY` 不包含 `?`，并且必须与实际发送的查询字符串完全一致。

---

## 6. 如何判断调用成功

### 6.1 普通响应

响应通常采用以下结构：

```json
{
  "code": 200,
  "message": "操作成功",
  "data": {}
}
```

调用方应同时检查：

1. HTTP 状态码是否成功。
2. 业务 `code` 是否表示成功。
3. `data` 是否满足当前接口的数据结构。

不同业务接口的 `data` 内容不同，以管理员提供的接口说明为准。

### 6.2 分页响应

分页接口的分页对象通常包含：

| 字段 | 说明 |
|---|---|
| `data` | 当前页数据列表 |
| `total` | 总记录数 |
| `pageNum` | 当前页码 |
| `pageSize` | 实际每页条数 |
| `pages` | 总页数 |
| `hasNext` | 是否存在下一页 |
| `hasPrev` | 是否存在上一页 |

完整响应中的分页对象位于业务响应的 `data` 字段内，因此列表通常通过 `响应.data.data` 获取。

### 6.3 OpenAPI 路由的 pageSize 固定规则

通过 `/openapi/v1/**` 调用分页接口时，服务端会将实际执行的 `pageSize` 固定为 10：

- 不传 `pageSize`：按 10 条处理。
- 传 `pageSize=10`：按 10 条处理。
- 传 `pageSize=100`：签名仍按原始参数 `pageSize=100` 计算，但业务执行时仍按 10 条处理。

建议调用方统一传 `pageSize=10` 或不传，并使用 `hasNext`、`pages` 逐页获取数据，不要假设可以通过调大 `pageSize` 一次取得全部数据。

---

## 7. 限流、额度、并发和临时封禁

平台可能同时配置以下限制：

| 限制 | 含义 | 调用方处理方式 |
|---|---|---|
| IP/AK 安全限流 | 防止来源 IP 或凭证短时大量请求 | 降低请求速度，检查是否存在重试风暴 |
| 业务频率 | 同一用户调用同一接口的频率限制 | 按接口限额控制请求间隔 |
| 并发限制 | 同一用户调用同一接口的并行请求数限制 | 降低客户端线程数或并发协程数 |
| 周期额度 | 按天、月或累计周期限制单个用户可调用次数 | 停止无效请求，联系管理员确认额度 |
| 接口每日总额度 | 所有用户、等级和 AK/SK 共享同一接口的每日总额度 | 停止重复调用，次日自动恢复或联系管理员 |
| 用户临时封禁 | 当前用户的开放接口访问被暂时停止 | 根据 `Retry-After` 等待，或联系管理员 |
| 系统临时封禁 | 开放网关暂时停止对外服务 | 根据 `Retry-After` 等待，并联系平台确认 |

这些限制按平台配置生效，调用方应在客户端实现限速，避免短时间集中请求和无限重试。

---

## 8. 常见错误及处理方法

错误响应通常包含：

```json
{
  "code": 401,
  "message": "AKSK 认证失败",
  "data": null
}
```

请同时参考 HTTP 状态码和 `message`：

| HTTP 状态 | 常见提示 | 可能原因 | 建议处理 |
|---|---|---|---|
| 400 | 请求格式不合法 | 请求体、路径或传输格式不符合要求 | 使用无请求体 GET，检查对外路径 |
| 400 | 查询参数格式不合法 | 查询字符串包含空参数或编码异常 | 检查 `&`、`=` 和 URL 编码 |
| 401 | AKSK 认证失败 | AK/SK 错误、请求头缺失、签名版本错误或签名不一致 | 按第 9 节逐项排查 |
| 401 | 请求时间戳已失效 | 使用了毫秒时间戳、时间戳过旧或服务器时钟不准 | 使用 10 位秒级时间戳并校准 NTP |
| 403 | 用户无开放接口权限 | 用户状态或等级不允许调用 | 联系管理员 |
| 403 | 当前等级未配置接口权限 | 当前用户等级没有目标接口策略 | 联系管理员配置接口和等级限额 |
| 404 | 接口不存在或未开放 | 路径错误、接口未同步或未启用 | 核对 `/openapi/v1/**` 路径并联系管理员 |
| 405 | 仅支持 GET 接口 | 使用了 POST、PUT 等方法 | 改为无请求体 GET |
| 409 | 重复请求已拒绝 | nonce 被重复使用 | 每次请求重新生成 nonce 并重新签名 |
| 429 | 系统访问频率超限，请等待约 N 秒后重试 | IP 或 AK 触发系统安全限流 | 按提示秒数等待后重试，避免并发重试 |
| 429 | 接口调用频率超限，请等待约 N 秒后重试 | 当前用户超过等级策略频率 | 按提示秒数等待后重试 |
| 429 | 接口并发数超限 | 同时执行的请求过多 | 降低客户端并发数 |
| 429 | 接口调用额度已用尽 | 当前用户周期额度耗尽 | 停止重复调用，联系管理员 |
| 429 | 接口每日总额度已用尽 | 所有用户共享的接口当日总额度耗尽 | 停止重复调用，次日自动恢复或联系管理员 |
| 429 | 当前用户已被暂时封禁，请等待约 X 后自动解除 | 当前用户触发临时封禁 | 按提示时间或 `Retry-After` 等待，到期自动解除 |
| 429 | OpenAPI 系统已被暂时封禁，请等待约 X 后自动恢复 | 系统级临时封禁 | 按提示时间或 `Retry-After` 等待，到期自动恢复 |
| 500 | 开放接口调用失败 | 服务端处理异常 | 记录请求 ID，有限次退避重试并联系平台 |
| 503 | 开放接口暂时不可用 | 限流、额度等保护依赖暂时异常 | 暂停密集请求，退避重试并联系平台 |

---

## 9. 签名失败排查清单

出现 `AKSK 认证失败` 时，按以下顺序检查：

1. **AK 是否完整**：不要包含空格、引号或换行。
2. **SK 是否完整**：确认使用的是当前有效 SK，而不是已轮换前的旧 SK。
3. **签名版本是否为 `v1`**。
4. **时间戳是否为 10 位秒级时间戳**：不能传 13 位毫秒时间戳。
5. **本机时间是否准确**：与服务器偏差不能超过 300 秒，建议启用 NTP 自动校时。
6. **nonce 是否符合规则**：16～64 位，只含字母、数字、`_`、`-`，并且每次请求唯一。
7. **请求方法是否为大写 `GET`**。
8. **签名路径是否与实际路径一致**：不要包含域名，不要误用内部接口路径。
9. **CanonicalQuery 是否正确**：检查 URL 解码、RFC 3986 再编码和排序。
10. **无查询参数时是否保留空行**。
11. **是否使用空字符串 SHA-256**，而不是其他请求体摘要。
12. **待签名串是否恰好 8 行**，行之间是否只有一个 `\n`，末尾是否多加了换行。
13. **HMAC 密钥是否为 SK 原文字节**，不要提前 Base64 解码。
14. **输出是否为 64 位十六进制**，不要误用 Base64。
15. **签名完成后 URL 是否又被框架重排或重新编码**。

不要通过连续发送大量错误签名来测试。安全错误持续发生时，平台可能触发告警或临时封禁。

---

## 10. 正确的重试方式

### 10.1 不建议重试的情况

以下错误通常需要修改请求或联系管理员，直接重试没有意义：

- 400：请求或参数格式错误。
- 401：凭证、时间戳或签名错误。
- 403：权限或等级策略未配置。
- 404：接口不存在或未开放。
- 405：请求方法错误。
- 409：nonce 重复。

### 10.2 可以有限重试的情况

- 429：限流、并发超限或临时封禁。
- 500：服务端临时异常。
- 网络超时或连接中断。

建议策略：

1. 如果存在 `Retry-After`，至少等待指定秒数。
2. 没有 `Retry-After` 时，采用指数退避，例如等待 1、2、4、8 秒，并增加少量随机抖动。
3. 设置最大重试次数，禁止无限重试。
4. 每次重试都重新生成时间戳和 nonce，并重新签名。
5. 不要并发重试同一批失败请求，避免进一步扩大限流。
6. 业务超时后不要盲目反复查询，必要时联系平台确认原请求是否成功。

---

## 11. 凭证轮换和禁用

### 11.1 轮换 SK

需要更换凭证时，联系管理员执行“轮换 SK”：

- AK 保持不变。
- 平台生成新的 SK，并且只在轮换成功时返回一次。
- 旧 SK 立即失效。
- 已有调用额度和历史用量不会因轮换而清零。
- 用户需要及时更新服务端配置，之后使用新 SK 签名。

建议轮换流程：

1. 用户准备凭证配置变更。
2. 管理员执行轮换并将新 SK 安全交给用户。
3. 用户更新服务端 SK。
4. 发起一次低频测试请求，确认新凭证可用。
5. 确认旧 SK 已不再使用。

### 11.2 禁用凭证

发生泄露、项目停用或不再需要调用时，应联系管理员禁用凭证。禁用后，该凭证不能继续调用开放接口；如后续仍需使用，应由管理员明确重新启用。

---

## 12. 上线前自检清单

正式接入前，建议逐项确认：

- [ ] 已获得正确的生产环境 HTTPS 基础地址。
- [ ] 已获得目标 `/openapi/v1/**` 接口路径和参数说明。
- [ ] AK、SK 已妥善保存，SK 未写入前端或代码仓库。
- [ ] 使用 10 位秒级时间戳，服务器已自动校时。
- [ ] 每次请求都会生成新的合规 nonce。
- [ ] CanonicalQuery 的解码、编码和排序与本文一致。
- [ ] 待签名串为 8 行，空查询参数和空请求体处理正确。
- [ ] HMAC-SHA256 输出为 64 位十六进制。
- [ ] 已成功完成一次真实接口调用。
- [ ] 已验证查询参数和中文参数的签名。
- [ ] 分页调用按每页 10 条处理，并能根据 `hasNext` 翻页。
- [ ] 客户端已设置连接和读取超时。
- [ ] 已实现限速、有限重试和指数退避。
- [ ] 已记录 `X-OpenApi-Request-Id`，但日志中不记录 SK。
- [ ] 已明确凭证泄露、轮换和管理员联系流程。

完成以上检查后，即可开始使用平台开放接口。
