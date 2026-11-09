# fanout（fork: lovegongqi/fanout）

上游 https://github.com/byJoey/fanout 的 fork，线上跑在自建 VPS 上（部署细节见本机记忆，不写进这个公开仓库）。
上游版本 v1.3.1，本 fork 从 v1.4.0 开始有改动。

## 这个项目是干什么的

把 VPN Gate 的公共节点变成母机上的 SOCKS5 端口：一个端口一个出口 IP。
每个出口跑在独立 netns 里，netns 内跑 openvpn，母机上开 SOCKS5 监听。
面板（`/var/lib/fanout/`）同时管理挂在这些出口上的 VLESS/VMess/Trojan 入站。

## 本 fork 相对上游的改动

### v1.4.0 — SOCKS5 出口订阅

- 新端点 `/socks`，与节点链接订阅 `/sub` 共用同一串口令
  - 默认 base64；`&target=links` 明文；`&target=clash` 出 Clash/Mihomo YAML
  - 只含 `status == "up"` 的出口；每次请求现生成，出口增删自动跟上
- `GET /api/sub` 多返回 `socks_path` / `socks_url` / `socks_count`
- 订阅弹窗多一条「SOCKS5 出口订阅」地址和复制按钮
- 新文件 `sub_socks.go` / `sub_socks_test.go`

### v1.4.1 — 修复访问路径重启后被换掉

上游 `LoadBasePath` 把「文件存在但内容为空」当成「还没配过」，重新随机了一个路径。
用户主动去掉路径前缀后，重启一次面板地址就变了，已分发出去的订阅地址跟着失效。

现在文件在就按它来的，空串保持为空；只有文件缺失才生成随机路径。

### 自更新指向 fork

`updateRepo` 从 `const` 改成 `var`，CI 注入 `-X main.updateRepo=${GITHUB_REPOSITORY}`，
所以 `f update` 拉的是 fork 自己的 release，不会把本地改动换成上游的。
`install.sh` / `f.sh` 里的 `REPO` 默认值也改成了 `lovegongqi/fanout`。

## 构建与测试

本机没装 Go，用 Docker 编译测试：

```sh
docker run --rm --privileged -v "$PWD":/src -w /src golang:1.24-alpine \
  sh -c "go vet ./... && go test ./..."
```

`--privileged` 必须加：`netnsguard_test.go` 要 `setns` 权限，普通容器里会假失败。

交叉编译给 VPS 用：

```sh
docker run --rm -v "$PWD":/src -w /src golang:1.24-alpine sh -c \
  "CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath \
   -ldflags '-s -w -X main.version=dev -X main.updateRepo=lovegongqi/fanout' -o fanout ."
```

## 发版流程

1. 改代码 → `go test` 通过
2. commit 推到 `fork`（`lovegongqi/fanout`）
3. 打 tag：`git tag -a v1.4.2 -m "..." && git push fork v1.4.2`
4. GitHub Actions 的 `release.yml` 自动出包（amd64 + arm64 + checksums.txt）
5. VPS 上换二进制 → 重启

## 部署到 VPS

发布后把 release 里的二进制换到线上 `/usr/local/bin/fanout`，`f.sh` 一起换，
然后 `systemctl restart fanout`。

```sh
curl -fsSLO https://github.com/lovegongqi/fanout/releases/download/v1.4.2/fanout-linux-amd64.tar.gz
curl -fsSLO https://github.com/lovegongqi/fanout/releases/download/v1.4.2/checksums.txt
grep "fanout-linux-amd64.tar.gz" checksums.txt | sha256sum -c -   # 文件名要一致
tar xzf fanout-linux-amd64.tar.gz
systemctl stop fanout
install -m 755 fanout /usr/local/bin/fanout && install -m 755 f.sh /usr/local/bin/f
systemctl start fanout
```

数据目录 `/var/lib/fanout/`（state.json 存隧道、settings.json 存端口和订阅口令、basepath 存访问路径）。
重启会短暂中断隧道，之后自动从 state.json 恢复；VPN Gate 节点满员时会反复换节点重连。

## 运维要点

- 面板只认随机路径 + 口令，没有 HTTPS
- 访问口令在 `/var/lib/fanout/password`，订阅口令在 `settings.json` 的 `sub_token`
- `f` 是管理菜单（状态/日志/隧道列表/改端口口令路径/更新/卸载）
- `f update` 走 fork 的 release
- 备份：`tar czf /root/fanout-backup-$(date +%Y%m%d-%H%M%S).tgz -C /var/lib fanout`
