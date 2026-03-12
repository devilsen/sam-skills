---
name: vps-proxy-deploy
description: 在全新 VPS 上一键部署翻墙服务。包含：SSH 公钥配置、3X-UI 面板安装、VLESS+XTLS-Reality 节点、Hysteria2 节点、Clash YAML 订阅生成、BBR 加速。使用时提供 VPS 的 IP、端口、用户名、密码即可。
---

用户想在一台新的 VPS 上部署翻墙服务。按照以下步骤依次执行，全程自动化，最终输出可直接导入 Clash Verge Rev 的订阅链接。

## 用户需要提供的信息

开始前，收集以下信息（如果用户没有提供，主动询问）：
- VPS IP 地址
- SSH 端口（默认 22）
- SSH 用户名（通常是 root）
- SSH 密码

将这些信息记为变量：`$VPS_IP`、`$VPS_PORT`、`$VPS_USER`、`$VPS_PASS`

---

## Step 1：配置 SSH 公钥免密登录

```bash
# 检查本地是否有 SSH 密钥
ls ~/.ssh/id_*.pub

# 没有则生成
ssh-keygen -t ed25519 -C "vps-deploy"

# 安装 sshpass（如果没有）
brew install sshpass

# 上传公钥
sshpass -p '$VPS_PASS' ssh-copy-id -p $VPS_PORT -o StrictHostKeyChecking=no $VPS_USER@$VPS_IP

# 检查并开启服务端公钥认证（如果默认关闭）
sshpass -p '$VPS_PASS' ssh -p $VPS_PORT -o StrictHostKeyChecking=no $VPS_USER@$VPS_IP \
  "grep -q 'PubkeyAuthentication no' /etc/ssh/sshd_config && \
   sed -i 's/^PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
   systemctl restart sshd && echo 'fixed' || echo 'already ok'"

# 验证免密登录
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "echo 'SSH key auth OK'"
```

---

## Step 2：查看服务器基本信息

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "
uname -a
cat /etc/os-release | head -5
free -h
df -h /
"
```

输出系统信息供用户参考。

---

## Step 3：安装 3X-UI 面板

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP \
  "bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)"
```

安装过程中会自动：
- 生成随机面板端口、用户名、密码
- 申请 Let's Encrypt IP 证书（6天有效期，自动续签）

安装完成后记录输出中的：
- `Username`
- `Password`
- `Port`
- `WebBasePath`
- `Access URL`

---

## Step 4：安装 sqlite3

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "apt-get install -y sqlite3"
```

---

## Step 5：生成 VLESS+XTLS-Reality 节点

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "
# 生成密钥对
python3 -c \"
import subprocess, base64

priv_pem = subprocess.run(['openssl', 'genpkey', '-algorithm', 'X25519'], capture_output=True).stdout
priv_der = subprocess.run(['openssl', 'pkey', '-outform', 'DER'], input=priv_pem, capture_output=True).stdout
pub_der = subprocess.run(['openssl', 'pkey', '-pubout', '-outform', 'DER'], input=priv_pem, capture_output=True).stdout
def b64url(b): return base64.urlsafe_b64encode(b).rstrip(b'=').decode()
print('PRIVATE_KEY=' + b64url(priv_der[-32:]))
print('PUBLIC_KEY=' + b64url(pub_der[-32:]))
\"

UUID=\$(cat /proc/sys/kernel/random/uuid)
SHORT_ID=\$(openssl rand -hex 8)
# 用上面生成的 PRIVATE_KEY / PUBLIC_KEY 替换下方变量
PRIVATE_KEY=<从上方输出获取>
PUBLIC_KEY=<从上方输出获取>

SETTINGS='{\"clients\":[{\"id\":\"'\$UUID'\",\"flow\":\"xtls-rprx-vision\",\"email\":\"user1\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"\"}],\"decryption\":\"none\",\"fallbacks\":[]}'
STREAM='{\"network\":\"tcp\",\"security\":\"reality\",\"externalProxy\":[],\"realitySettings\":{\"show\":false,\"xver\":0,\"dest\":\"www.apple.com:443\",\"serverNames\":[\"www.apple.com\"],\"privateKey\":\"'\$PRIVATE_KEY'\",\"minClient\":\"\",\"maxClient\":\"\",\"maxTimediff\":0,\"shortIds\":[\"'\$SHORT_ID'\"],\"settings\":{\"publicKey\":\"'\$PUBLIC_KEY'\",\"fingerprint\":\"chrome\",\"serverName\":\"\",\"spiderX\":\"/\"}},\"tcpSettings\":{\"acceptProxyProtocol\":false,\"header\":{\"type\":\"none\"}}}'
SNIFFING='{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\",\"fakedns\"],\"metadataOnly\":false,\"routeOnly\":false}'

sqlite3 /etc/x-ui/x-ui.db \"INSERT INTO inbounds (user_id,up,down,total,remark,enable,expiry_time,listen,port,protocol,settings,stream_settings,tag,sniffing) VALUES (1,0,0,0,'VLESS-Reality',1,0,'',443,'vless','\$SETTINGS','\$STREAM','inbound-443','\$SNIFFING');\"

systemctl restart x-ui
echo \"UUID: \$UUID\"
echo \"PUBLIC_KEY: \$PUBLIC_KEY\"
echo \"SHORT_ID: \$SHORT_ID\"
"
```

记录输出的 UUID、PUBLIC_KEY、SHORT_ID。

---

## Step 6：开启 BBR 加速

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules-load.d/bbr.conf

cat > /etc/sysctl.d/99-bbr.conf << 'EOF'
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
EOF

sysctl -p /etc/sysctl.d/99-bbr.conf
sysctl net.ipv4.tcp_congestion_control
"
```

---

## Step 7：安装 nginx 并生成 Clash 订阅文件

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "apt-get install -y nginx -q"

# 配置 nginx
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "
cat > /etc/nginx/sites-available/clash-sub << 'EOF'
server {
    listen 8080;
    root /var/www/clash;
    location / {
        add_header Content-Type 'application/yaml; charset=utf-8';
    }
}
EOF
mkdir -p /var/www/clash
ln -sf /etc/nginx/sites-available/clash-sub /etc/nginx/sites-enabled/clash-sub
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl restart nginx
"
```

生成 Clash YAML（将下方占位符替换为实际值）：

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "cat > /var/www/clash/clash.yaml << 'YAML'
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
external-controller: 127.0.0.1:9090

dns:
  enable: true
  ipv6: false
  default-nameserver: [223.5.5.5, 119.29.29.29]
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver: [223.5.5.5, 119.29.29.29]
  fallback: [tls://8.8.8.8:853, tls://1.1.1.1:853]
  fallback-filter:
    geoip: true
    geoip-code: CN

proxies:
  - name: VLESS-Reality
    type: vless
    server: $VPS_IP
    port: 443
    uuid: <UUID>
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: <PUBLIC_KEY>
      short-id: <SHORT_ID>
    servername: www.apple.com
    client-fingerprint: chrome

proxy-groups:
  - name: Proxy
    type: select
    proxies: [VLESS-Reality, DIRECT]
  - name: Auto
    type: url-test
    proxies: [VLESS-Reality]
    url: http://www.gstatic.com/generate_204
    interval: 300
  - name: Media
    type: select
    proxies: [VLESS-Reality, DIRECT]
  - name: Apple
    type: select
    proxies: [DIRECT, VLESS-Reality]
  - name: Microsoft
    type: select
    proxies: [DIRECT, VLESS-Reality]

rules:
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - DOMAIN-SUFFIX,apple.com,Apple
  - DOMAIN-SUFFIX,icloud.com,Apple
  - DOMAIN-SUFFIX,microsoft.com,Microsoft
  - DOMAIN-SUFFIX,office.com,Microsoft
  - DOMAIN-SUFFIX,netflix.com,Media
  - DOMAIN-SUFFIX,youtube.com,Media
  - DOMAIN-SUFFIX,spotify.com,Media
  - DOMAIN-SUFFIX,twitter.com,Media
  - DOMAIN-SUFFIX,x.com,Media
  - DOMAIN-SUFFIX,instagram.com,Media
  - DOMAIN-SUFFIX,facebook.com,Media
  - DOMAIN-SUFFIX,telegram.org,Media
  - DOMAIN-SUFFIX,discord.com,Media
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,googleapis.com,Proxy
  - DOMAIN-SUFFIX,github.com,Proxy
  - DOMAIN-SUFFIX,githubusercontent.com,Proxy
  - DOMAIN-SUFFIX,openai.com,Proxy
  - DOMAIN-SUFFIX,claude.ai,Proxy
  - DOMAIN-SUFFIX,anthropic.com,Proxy
  - DOMAIN-SUFFIX,cn,DIRECT
  - DOMAIN-SUFFIX,baidu.com,DIRECT
  - DOMAIN-SUFFIX,taobao.com,DIRECT
  - DOMAIN-SUFFIX,jd.com,DIRECT
  - DOMAIN-SUFFIX,qq.com,DIRECT
  - DOMAIN-SUFFIX,wechat.com,DIRECT
  - DOMAIN-SUFFIX,alipay.com,DIRECT
  - DOMAIN-SUFFIX,bilibili.com,DIRECT
  - DOMAIN-SUFFIX,douyin.com,DIRECT
  - DOMAIN-SUFFIX,zhihu.com,DIRECT
  - DOMAIN-SUFFIX,weibo.com,DIRECT
  - DOMAIN-SUFFIX,bytedance.com,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
YAML"
```

---

## Step 8：验证并输出结果

```bash
ssh -p $VPS_PORT $VPS_USER@$VPS_IP "
echo '=== 服务状态 ==='
systemctl is-active x-ui
ss -tlnp | grep -E '443|8080'
echo '=== 订阅验证 ==='
curl -s http://localhost:8080/clash.yaml | grep 'name:'
"
```

---

## 最终输出给用户

部署完成后，向用户输出：

1. **3X-UI 面板**
   - 地址：`https://$VPS_IP:<面板端口>/<WebBasePath>`
   - 用户名 / 密码

2. **Clash 订阅链接**
   ```
   http://$VPS_IP:8080/clash.yaml
   ```

3. **使用说明**
   - 客户端推荐：Clash Verge Rev（Mac/Windows）、v2rayNG（Android）、Shadowrocket（iOS）
   - Clash Verge Rev：Profiles → + → Remote → 粘贴订阅链接

---

## 注意事项

- **云防火墙**：需在 VPS 服务商控制台开放以下端口（TCP）：443、8080、面板端口
- **Hysteria2**：国内部分运营商对 UDP 封锁严格，如无法使用则跳过，VLESS+Reality 已足够稳定
- **证书续签**：Let's Encrypt IP 证书 6 天有效，acme.sh 会自动续签，无需手动操作
- **BBR**：已通过 sysctl 持久化，重启后自动生效
