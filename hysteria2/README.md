# Hysteria2 节点部署流程总结

## 一、部署目标

基于 `Docker` 在 Linux 服务器上部署 `Hysteria2` 服务端，使用已有证书文件，开放 `UDP` 端口，对外提供可供 `Mihomo / Clash.Meta / OpenClash` 等客户端连接的节点。

---

## 二、已知部署参数示例

- 域名：`trojan-go1.jason002.top`
- 备用域名：`trojan-go2.jason002.top`
- 服务端口：`7654/udp`
- 证书文件：`/etc/hysteria/server.pem`
- 私钥文件：`/etc/hysteria/server.key`
- 配置文件：`/etc/hysteria/config.yaml`
- Docker Compose 文件：`/etc/hysteria/docker-compose.yaml`
- 认证密码：`123456`

---

## 三、前置条件

部署前需要确认以下条件已经满足：

- 有一台 Linux 服务器
- 已安装 `Docker` 和 `Docker Compose`
- 域名已解析到服务器公网 IP
- 已准备好可用证书与私钥
- 系统防火墙允许 `7654/udp`
- 云服务器安全组允许 `7654/udp`

---

## 四、目录准备

建议统一使用以下目录：

```bash
/etc/hysteria
```

目录中包含：

- `/etc/hysteria/docker-compose.yaml`
- `/etc/hysteria/config.yaml`
- `/etc/hysteria/server.pem`
- `/etc/hysteria/server.key`

---

## 五、docker-compose.yaml

```yaml
services:
  hysteria:
    image: tobyxdd/hysteria:latest
    container_name: hysteria
    restart: unless-stopped
    network_mode: host
    volumes:
      - /etc/hysteria/config.yaml:/etc/hysteria/config.yaml:ro
      - /etc/hysteria/server.pem:/etc/hysteria/server.pem:ro
      - /etc/hysteria/server.key:/etc/hysteria/server.key:ro
    command: ["server", "-c", "/etc/hysteria/config.yaml"]
```

### 说明

- `network_mode: host` 直接使用宿主机网络，避免端口映射问题
- 容器启动命令为：
  - `server -c /etc/hysteria/config.yaml`

---

## 六、config.yaml

```yaml
listen: :7654

tls:
  cert: /etc/hysteria/server.pem
  key: /etc/hysteria/server.key
  sniGuard: dns-san

auth:
  type: password
  password: "123456"

bandwidth:
  up: 900 mbps
  down: 900 mbps

ignoreClientBandwidth: false

quic:
  initStreamReceiveWindow: 8388608
  maxStreamReceiveWindow: 8388608
  initConnReceiveWindow: 20971520
  maxConnReceiveWindow: 20971520
  maxIdleTimeout: 60s
  maxIncomingStreams: 1024
  disablePathMTUDiscovery: false

udpIdleTimeout: 90s
```

### 参数说明

- `listen: :7654`
  - 服务监听 `7654` 端口
- `tls.cert / tls.key`
  - 指向证书和私钥文件
- `auth.password`
  - 客户端连接认证密码
- `bandwidth`
  - 服务端带宽提示，建议按真实出口能力填写，不要盲目写太大
- `ignoreClientBandwidth: false`
  - 允许客户端使用自己的带宽提示
- `quic`
  - QUIC 相关参数，建议保持稳妥值
- `udpIdleTimeout`
  - UDP 会话空闲保留时间

---

## 七、启动服务

进入目录：

```bash
cd /etc/hysteria
```

启动：

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f --tail=100
```

---

## 八、常见问题处理

### 1. 容器名冲突

如果提示：

```text
The container name "/hysteria" is already in use
```

说明系统中已存在旧容器。

处理方式：

```bash
docker rm -f hysteria
```

然后重新启动：

```bash
cd /etc/hysteria
docker compose up -d
```

---

### 2. 端口被占用

检查监听情况：

```bash
ss -lunp | grep 7654
```

如果输出类似：

```text
*:7654
users:(("hysteria",pid=xxxx,fd=x))
```

说明 `Hysteria2` 已经监听 `7654/udp`。

---

### 3. ufw 只放行了 TCP

Hysteria2 主要走 `UDP`，需要单独放行：

```bash
ufw allow 7654/udp
```

如果需要同时开放 TCP：

```bash
ufw allow 7654/tcp
```

查看规则：

```bash
ufw status numbered
```

---

### 4. 端口和证书是否冲突

不会冲突。

- TLS 证书主要校验域名
- 端口只是监听入口
- `7654` 与证书本身没有冲突

只要证书包含对应域名即可。

---

## 九、如何判断服务端是否正常

### 1. 进程正常监听

```bash
ss -lunp | grep 7654
```

### 2. 容器状态正常

```bash
docker ps --filter name=^/hysteria$
```

### 3. 日志无明显报错

```bash
docker logs --tail=100 hysteria
```

重点看：

- 是否正常启动
- 是否有证书加载失败
- 是否有配置解析错误
- 是否有端口占用报错

### 4. 防火墙和安全组已放行

- 系统防火墙放行 `7654/udp`
- 云厂商安全组放行 `7654/udp`

### 5. 客户端可成功连接

这是最终判断标准。

---

## 十、Hysteria2 客户端节点示例

### 1. 域名连接方式

```yaml
- name: hysteria(东京)
  type: hysteria2
  server: trojan-go1.jason002.top
  port: 7654
  password: 123456
  sni: trojan-go1.jason002.top
```

### 2. 第二个节点示例

```yaml
- name: hysteria(甲骨文AMD-1G)
  type: hysteria2
  server: trojan-go2.jason002.top
  port: 7654
  password: 123456
  sni: trojan-go2.jason002.top
```

---

## 十一、是否必须使用域名

不是必须。

Hysteria2 可以直接连接 `IP`，但通常仍建议：

- 使用 `IP` 连接
- 使用域名作为 `SNI`

例如：

```yaml
- name: hysteria(IP直连)
  type: hysteria2
  server: 1.2.3.4
  port: 7654
  password: 123456
  sni: trojan-go1.jason002.top
```

这样更容易兼容现有证书。

---

## 十二、速度调优说明

### 1. bandwidth 不是越大越好

如果把 `bandwidth` 写得过大，可能会出现：

- 网卡监控流量很高
- 实际看视频仍然卡顿
- 链路抖动、丢包、重传增加

### 2. 推荐做法

- `bandwidth` 按真实可稳定跑到的带宽填写
- 建议填写真实测速结果的 `70%~90%`
- 客户端和服务端都不要盲目写超大值

### 3. 稳定优先建议

如果更在意看视频稳定性，不要一味追求峰值吞吐。

---

## 十三、Mihomo / Clash.Meta 单节点配置思路

本次使用的是：

- 单本地 `yaml` 文件
- 非订阅方式
- `rule-providers` 使用 `Loyalsoldier` 规则集
- 国内常见域名和中国大陆 IP 直连
- 国外站点走代理节点

---

## 十四、常用命令汇总

```bash
cd /etc/hysteria
docker compose up -d
docker compose ps
docker compose logs -f --tail=100
docker rm -f hysteria
ss -lunp | grep 7654
ufw allow 7654/udp
ufw status numbered
```

---

## 十五、总结

Hysteria2 的核心部署流程可以归纳为：

1. 准备服务器、域名、证书和 Docker 环境
2. 编写 `docker-compose.yaml`
3. 编写 `config.yaml`
4. 放行 `7654/udp`
5. 启动容器并查看日志
6. 确认端口监听正常
7. 使用客户端节点连接测试
8. 根据实际体验调优 `bandwidth` 等参数

只要 `端口监听正常 + 日志正常 + 客户端能连接`，基本可以判断节点部署成功。