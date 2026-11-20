# dce 使用指南

`dce` 是 DaoCloud Enterprise（DCE）的命令行工具，可管理登录凭据、发现 API 命令并执行操作。

本文的 `--insecure`、`--debug` 行为按 `dce v0.2.0-rc.16`（依赖 lathe v0.6.2）核对。本仓库当前源码依赖 lathe v0.4.5，从源码构建的 CLI 可能与该发行版不同。先运行 `dce --version` 确认版本。

## 登录与主机选择

使用平台地址登录，在交互提示中粘贴 Token：

```sh
dce auth login --hostname https://dce.example.com
```

登录会通过 `GET /apis/ghippo.io/v1alpha1/current-user` 校验凭据。不要将真实 Token 写入命令示例、脚本仓库或排障日志。

查看登录状态，或为某次操作显式指定平台地址：

```sh
dce auth status -o json
dce --hostname https://dce.example.com global-management about list-g-product-versions -o json
```

主机地址的优先级为：`--hostname`、`DCE_HOST` 环境变量、`dce auth use <host>` 选择的主机、`http.default_hostname`、`hosts.yml` 中唯一的主机。

## 发现与执行命令

先搜索候选命令，再查看精确参数及请求定义：

```sh
dce search "list clusters" --json
dce commands show container-management cluster list-clusters --json
```

根据命令定义补齐参数后执行；脚本中建议使用 `-o json`。不要仅凭搜索结果猜测参数。

```sh
dce container-management cluster list-clusters -o json
```

执行修改或删除操作前，检查命令定义中的 `mutation` 和 `dry_run`。如果支持 HTTP 预览，使用定义中声明的预览参数检查请求。

## 跳过 TLS 证书校验：`--insecure`

`--insecure` 是全局布尔参数，默认关闭。在 rc.16 中被刻意隐藏，因此 `dce --help`、`dce auth login --help` 不显示它，但仍可使用。

内部 CA 未受信任、自签名证书或证书名称不匹配时，可以在明确接受风险的环境中跳过 TLS 校验：

```sh
dce --insecure auth login --hostname https://dce.example.com
```

**用于登录时，登录成功会将 `insecure: true` 与该主机的凭据一起保存。后续对该主机的 API 请求会沿用此设置，即使不再传 `--insecure`。**

对普通 API 命令传入该参数，只会为本次调用开启跳过校验，不会因此修改已保存的主机设置：

```sh
dce --insecure --hostname https://dce.example.com global-management about list-g-product-versions -o json
```

`--insecure` 不会修复网络连接、Token 过期或权限不足，也不会安装 CA。它关闭了服务端证书验证，应优先通过信任内部 CA、补齐网关中间证书链和使用证书覆盖的域名解决问题。

恢复证书校验时，修复证书或信任配置后，不带 `--insecure` 重新登录同一主机；成功后保存的主机记录会恢复为校验证书。普通 API 命令中的 `--insecure=false` 不会覆盖主机记录中已保存的 `insecure: true`。

### 与 `--skip-validate` 的区别

`dce auth login --skip-validate` 跳过登录时的远端凭据校验，直接保存凭据。它不会关闭后续 API 请求的 TLS 校验，也不代表凭据有效，不能作为证书问题的修复方式。

## HTTP 排障：`--debug`

`--debug` 也是默认关闭的隐藏全局参数。对生成的 API 命令使用它，可在标准错误输出（stderr）中查看请求方法、目标主机、HTTP 状态码及请求耗时：

```sh
dce --debug --hostname https://dce.example.com global-management about list-g-product-versions -o json
```

示意输出：

```text
> GET request host=dce.example.com

< HTTP 200 (120ms)
```

请求未获得 HTTP 响应时，调试输出为 `request failed` 和耗时；它不保证显示底层 TLS 或网络异常详情。rc.16 不会在该调试输出中打印完整 URL 路径、请求头或请求/响应正文。

`--debug` 只影响本次调用，不会保存到主机配置。rc.16 的 `auth login` 校验请求没有接入该开关，因此在登录命令上添加 `--debug` 不会获得上述 HTTP 调试输出。

如需分别保存 JSON 结果和调试信息：

```sh
dce --debug global-management about list-g-product-versions -o json > result.json 2> debug.log
```

分享日志前检查内容；API 结果文件可能包含业务数据，stderr 也可能包含错误信息。

## 排查登录失败

如果只看到：

```text
Error: API request failed
Hint: check the hostname, network connectivity, and TLS settings
```

该提示不能单独证明 Token 无效。rc.16 的错误格式化可能丢失登录代码补充的 TLS 提示。

先从运行 dce 的同一环境检查 HTTPS：

```sh
curl --connect-timeout 5 --max-time 10 -I https://dce.example.com
```

- 连接失败或超时：检查地址、端口、VPN 和网络连通性。
- 证书校验失败：获取管理员提供的 CA 证书，检查信任配置和服务端证书链；必要时按上文使用 `--insecure`。
- HTTPS 可达但登录失败：检查 Token 和登录校验接口的响应。首页返回 200 不代表 API 鉴权成功。

获取内部根 CA 后，可先用它验证连接，无需立即修改系统信任：

```sh
curl --cacert ./dce-root-ca.crt -I https://dce.example.com
```
