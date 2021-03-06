# 自动申请 HTTPS 证书

1.转发 80 端口到 10080 端口（需要 root 权限）

```bash
iptables -t nat -I PREROUTING 1 -p tcp --dport 80 -j REDIRECT --to-ports 10080
```

2.安装 acme.sh

该脚本依赖 `openssl`，否则无法生成证书（大部分系统默认已安装）

```bash
yum install -y openssl
```

推荐安装 `crontab`，可定期续签证书（CentOS 7 为例）

```bash
yum install cronie
service crond start
```

安装 acme.sh（无需 root 权限，在 `jsproxy` 用户下安装）

```bash
su - jsproxy
curl https://raw.githubusercontent.com/Neilpang/acme.sh/master/acme.sh | INSTALLONLINE=1  sh
```

3.申请证书

```bash
# 服务器公网 IP
ip=$(curl -s https://api.ipify.org)
domain="$ip.xip.io"

dist=~/server/cert/$domain
mkdir -p $dist

~/.acme.sh/acme.sh \
  --issue \
  -d $domain \
  --keylength ec-256 \
  --webroot ~/server/acme

~/.acme.sh/acme.sh \
  --install-cert \
  -d $domain \
  --ecc \
  --key-file $dist/ecc.key \
  --fullchain-file $dist/ecc.cer
```

如果申请失败（例如提示 `rate limit exceeded`），尝试将 `xip.io` 换成 `nip.io`、`sslip.io` 等其他类似的域名。


4.生成配置文件：

```conf
echo "
listen                8443 ssl http2;
ssl_certificate       cert/$domain/ecc.cer;
ssl_certificate_key   cert/$domain/ecc.key;
" > ~/server/cert/cert.conf
```

重启服务：`~/server/run.sh reload`

5.验证

访问 `https://服务器IP.xip.io:8443/`，没出现证书错误即成功（404 等错误不影响）。

6.关闭 80 端口转发

```bash
iptables -t nat -D PREROUTING 1
```