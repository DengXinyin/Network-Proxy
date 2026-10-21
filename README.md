# Network-Proxy

本仓库记录服务器网络代理（Mihomo / Clash Verge / Codex / OpenAI）的排查思路、可复现命令和已验证的解决办法。

## 文档索引

- [服务器代理排查指南_合并最终版.md](服务器代理排查指南_合并最终版.md)：主文档，推荐优先阅读。
- [服务器代理排查指南_完整实践版_v4.md](服务器代理排查指南_完整实践版_v4.md)：完整实践版 v4。
- [服务器代理排查指南_完整实践版_v3.md](服务器代理排查指南_完整实践版_v3.md)：完整实践版 v3。
- [服务器代理排查指南_完整实践版_v2.md](服务器代理排查指南_完整实践版_v2.md)：完整实践版 v2。

## 订阅链接失效导致代理全部不可用

> 记录时间：2026-10-21。适用环境：node4 服务器，用户 `xydeng`，mihomo 监听 `127.0.0.1:5964`。

### 现象

- codex 无法使用，OpenAI / ChatGPT 访问失败；
- 通过代理执行 `curl` 报 `SSL_ERROR_SYSCALL`；
- 但本机 mihomo 服务正常运行，`127.0.0.1:5964` 正常监听；
- 订阅 `updatedAt` 长时间不变，节点健康检查全部失败。

### 原因

这类故障通常由三层原因叠加造成，只看其中一层容易误判：

1. **机场更换域名，旧订阅入口失效**
   - 旧订阅域名返回 `NXDOMAIN`；
   - 旧节点域名返回 `SERVFAIL`；
   - 此时账号本身可能并未到期，只是服务商更换了域名和入口。

2. **订阅更新死循环（最关键）**
   - `proxy-providers` 未设置 `proxy: DIRECT` 时，mihomo 拉取订阅的请求会按 `MATCH,PERSONAL_PROXY` 规则进入代理组；
   - 代理组里全是已经失效的节点，于是形成"要更新订阅必须先有可用节点，但节点已经失效"的死循环；
   - 即使换成新订阅地址，订阅仍然拉不下来。

3. **新订阅地址对默认 UA 返回 403**
   - 用 curl 默认 UA 请求订阅会得到 `403 订阅访问被拒绝`；
   - 使用 `mihomo/...`、`clash-verge/...` 等客户端 UA 才返回 200 和 Clash YAML。

### 排查命令

```bash
# 1. 确认本机代理服务和端口正常
systemctl --user is-active mihomo
ss -tlnp | grep 5964

# 2. 通过代理测试出网
#    SSL_ERROR_SYSCALL：本地代理连上了，但上游隧道失败
curl -sS -m 15 -o /dev/null -w 'HTTP %{http_code}\n' \
  -x http://127.0.0.1:5964 https://api.openai.com/v1/models

# 3. 查看订阅更新时间和节点数量
curl -s --unix-socket /home/xydeng/.config/mihomo/api.sock \
  http://localhost/providers/proxies | python3 -c "
import json, sys
p = json.load(sys.stdin)['providers']['personal_subscription']
print('updatedAt:', p['updatedAt'])
print('nodes:', len(p['proxies']))
"

# 4. 检查订阅域名是否还能解析
getent hosts <订阅域名>

# 5. 用权威 DoH 交叉验证
#    Status 3 = NXDOMAIN，Status 2 = SERVFAIL
curl -sS --noproxy '*' 'https://dns.alidns.com/resolve?name=<订阅域名>&type=A'
curl -sS --noproxy '*' -H 'accept: application/dns-json' \
  'https://doh.pub/dns-query?name=<订阅域名>&type=A'

# 6. 用客户端 UA 验证新订阅地址
curl -sS --noproxy '*' -m 20 -A 'mihomo/1.19.24' \
  -o /tmp/newsub.yaml -w 'HTTP %{http_code} size %{size_download}\n' \
  '<新的订阅地址>'
head -3 /tmp/newsub.yaml
```

判断原则：

- 端口在监听、`ipinfo` 能返回但 OpenAI 失败：多半是节点或订阅问题；
- 端口在监听、所有请求都报 `SSL_ERROR_SYSCALL`：上游节点全挂或订阅失效；
- 端口不在监听：才是 mihomo 服务或配置问题。

### 解决办法

#### 1. 换成新的订阅地址

从机场官网公告、TG 群或客服处获取新订阅地址。注意不要在 README、GitHub 或公开位置写入真实订阅 URL、UUID、Token。

#### 2. 修改 `config.yaml`

三个关键配置缺一不可：

```yaml
proxy-providers:
  personal_subscription:
    type: http
    url: "<YOUR_SUBSCRIPTION_URL>"
    path: ./providers/personal_subscription.yaml

    # 让订阅通过直连拉取，避免进入已失效的代理组
    proxy: DIRECT

    # 机场对默认 UA 返回 403，必须显式带客户端 UA
    header:
      User-Agent:
        - "mihomo/1.19.24"

    interval: 3600
```

#### 3. 备份旧缓存并重启

mihomo 启动时会优先加载本地订阅缓存，缓存存在时不一定立即重新拉取。换 URL 后应先备份并移走旧缓存：

```bash
cd /home/xydeng/.config/mihomo

# 备份配置
cp -a config.yaml "config.yaml.bak.before-newsub-$(date +%Y%m%d-%H%M%S)"

# 备份并移走旧订阅缓存
mv providers/personal_subscription.yaml \
   "providers/personal_subscription.yaml.bak.old-$(date +%Y%m%d-%H%M%S)"

# 重启服务
systemctl --user restart mihomo
```

#### 4. 验证

```bash
# 等待新订阅缓存写入
timeout 60 bash -c 'until grep -q "<新节点域名关键词>" \
  /home/xydeng/.config/mihomo/providers/personal_subscription.yaml 2>/dev/null; do
  sleep 0.5; done'

# 组内节点测速（url 参数需要 URL 编码）
curl -s -m 20 --unix-socket /home/xydeng/.config/mihomo/api.sock \
  'http://localhost/proxies/ChatGPT-Auto/delay?url=https%3A%2F%2Fapi.openai.com%2Fv1%2Fmodels&timeout=8000'

# 通过代理访问
curl -sS -m 20 -o /dev/null -w 'api.openai.com HTTP %{http_code} %{time_total}s\n' \
  -x http://127.0.0.1:5964 https://api.openai.com/v1/models

curl -sS -m 20 -x http://127.0.0.1:5964 https://ipinfo.io/json

# codex 端到端验证
timeout 120 codex exec --skip-git-repo-check -s read-only "Reply with exactly: OK"
```

成功标志：

| 检查项 | 正常表现 |
|---|---|
| `systemctl --user is-active mihomo` | `active` |
| `ss -tlnp \| grep 5964` | `127.0.0.1:5964` 处于 LISTEN |
| 订阅 `updatedAt` | 为最近时间，节点数几十个 |
| `ChatGPT-Auto/delay` | 返回 `{"delay": 几百毫秒}` |
| `api.openai.com` | HTTP 401（未带 API key，说明链路正常） |
| codex 实测 | 正常返回结果 |

### 注意事项

1. **`chatgpt.com` 用 curl 返回 403 是正常的**：这是 Cloudflare 对非浏览器请求的 bot 拦截，不代表代理失效。判断 OpenAI 连通性应看 `api.openai.com` 是否返回 401/200，以及 codex 是否能实际返回内容。
2. **`api.openai.com` 返回 401 是正常现象**：说明 TCP / TLS / 代理链路已经通，只是请求没有携带 API key。
3. **订阅更新必须直连**：只要 `proxy-providers` 没有设置 `proxy: DIRECT`，节点失效时就会形成"要更新订阅必须先有可用节点"的死循环。
4. **换订阅地址后要移走旧缓存**：否则 mihomo 可能继续使用旧节点文件，直到下一次定时更新。
5. **机场再次更换域名时**：只需替换 `url`，保留 `proxy: DIRECT` 和 `header.User-Agent`，备份并移走订阅缓存后重启服务，再按上面的检验步骤确认。
6. **凭据安全**：真实订阅 URL 含账号令牌，不要提交到本仓库或任何公开位置；如已泄露，应尽快在服务商处轮换。
