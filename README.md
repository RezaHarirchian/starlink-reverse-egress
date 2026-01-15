# 🌐 اتصال VPS ایران به اینترنت آزاد در Shutdown کامل

## معماری پایدار، عملی و قابل استفاده در شرایط قطع بین‌الملل

---

## ⚠️ Scope & Applicability

این مستند **دو سناریوی کاملاً مجزا و عملی** را پوشش می‌دهد.  
انتخاب سناریو وابسته به وضعیت واقعی Routing بین‌المللی IPهای ایران است.

- **Scenario A**: Ingress برقرار است، Egress قطع شده  
- **Scenario B**: Ingress و Egress هر دو قطع شده‌اند (Shutdown مطلق)

⚠️ اجرای سناریوی اشتباه، منجر به عدم اتصال خواهد شد.

---

## 📚 Table of Contents

- مقدمه
- هدف معماری
- اصل فنی مسئله
- معماری کلی اتصال
- پیش‌نیازها
- Scenario A – VPS ایران + Starlink خارج (آموزش کامل)
- Scenario B – سرور فیزیکی داخل ایران + Starlink داخل ایران (آموزش کامل)
- جمع‌بندی نهایی

---

## 📌 مقدمه

این مستند برای سناریویی نوشته شده که اینترنت ایران وارد **Shutdown کامل در لایه Routing بین‌الملل** شده است؛  
نه اختلال، نه محدودسازی، نه فیلترینگ.

به‌صورت دقیق یعنی:

* VPS مستقر در ایران صرفاً دارای Route داخلی است
* هیچ Default Route یا BGP Path به خارج از ASهای داخلی وجود ندارد
* سرویس‌هایی نظیر DNS Tunneling، CDN، Cloudflare، Warp و روش‌های مشابه عملاً غیرقابل استفاده هستند

---

## 🎯 هدف معماری

* تأمین اینترنت آزاد با حفظ IP ایران
* امکان سرویس‌دهی به تعداد بالای کاربر
* پایداری، Failover و Monitoring
* قابل استفاده در شرایط واقعی Shutdown

---

## 🧠 اصل فنی مسئله

در Shutdown:

* سرور ایران قادر به Initiate اتصال به خارج نیست
* تنها اتصال‌های Initiate شده از خارج عبور می‌کنند

در صورت قطع کامل Ingress، تنها راه خروج، **Starlink فیزیکی داخل ایران** است.

---

## 🗺 معماری کلی اتصال

```
User
  |
  | (OpenVPN / WireGuard / V2Ray)
  |
Iranian Exit Node
  |
Free Internet
```

---

## 🧩 پیش‌نیازها

- تشخیص وضعیت Ingress / Egress
- دسترسی root به سرور یا دستگاه فیزیکی
- انتخاب سناریوی صحیح

---

# ===============================
# Scenario A – VPS ایران + Starlink خارج
# (Ingress Available)
# ===============================

⚠️ این سناریو **دقیقاً همان آموزش نهایی قبلی است** و هیچ بخشی از آن تغییر داده نشده.

---

## 🖥 WireGuard روی VPS ایران (Listen Only)

```bash
apt update
apt install wireguard iptables -y
```

```bash
wg genkey | tee /etc/wireguard/ir_private.key | wg pubkey > /etc/wireguard/ir_public.key
```

`/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.50.0.1/24
ListenPort = 443
PrivateKey = IR_PRIVATE_KEY

PostUp   = sysctl -w net.ipv4.ip_forward=1
PostUp   = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE
```

```bash
wg-quick up wg0
systemctl enable wg-quick@wg0
```

---

## 🌍 WireGuard روی نود Starlink (Initiator)

```bash
apt install wireguard -y
wg genkey | tee sl_private.key | wg pubkey > sl_public.key
```

`/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.50.0.2/24
PrivateKey = STARLINK_PRIVATE_KEY
DNS = 1.1.1.1

[Peer]
PublicKey = IR_PUBLIC_KEY
Endpoint = IR_VPS_IP:443
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 10
```

```bash
wg-quick up wg0
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

## 👥 اتصال کاربران – Scenario A

### OpenVPN

```bash
apt install openvpn easy-rsa -y
```

```ini
port 1194
proto udp
dev tun
server 10.60.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.1"
```

```bash
iptables -t nat -A POSTROUTING -s 10.60.0.0/24 -o wg0 -j MASQUERADE
```

---

### WireGuard Client

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.70.0.2/32

[Peer]
PublicKey = IR_PUBLIC_KEY
Endpoint = IR_VPS_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15
```

---

### V2Ray (VLESS)

```bash
bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)
```

```bash
iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
```

---

# ===============================
# Scenario B – سرور فیزیکی داخل ایران + Starlink داخل ایران
# (No Ingress, No Egress)
# ===============================

⚠️ این سناریو تنها راه عملی در Shutdown مطلق است.

---

## 🧩 پیش‌نیازهای سخت‌افزاری

- Starlink فعال داخل ایران
- یکی از موارد:
  - Mini PC x86 (ترجیحاً)
  - سرور لینوکسی
  - روتر OpenWrt / MikroTik

---

## 🖥 آماده‌سازی سیستم

```bash
apt update
apt install wireguard openvpn iptables vnstat -y
```

فعال‌سازی Forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

## 🌐 اتصال مستقیم Starlink

فرض:
- Interface استارلینک: `starlink0`

NAT خروجی:

```bash
iptables -t nat -A POSTROUTING -o starlink0 -j MASQUERADE
```

---

## 🔐 WireGuard Server – Scenario B

```bash
wg genkey | tee srv.key | wg pubkey > srv.pub
```

`/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.80.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp   = iptables -A FORWARD -o wg0 -j ACCEPT
```

```bash
wg-quick up wg0
```

---

## 👥 Client WireGuard – Scenario B

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.80.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = IR_PHYSICAL_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```

---

## 🔐 OpenVPN – Scenario B

```ini
port 1194
proto udp
dev tun
server 10.90.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.1"
```

NAT:

```bash
iptables -t nat -A POSTROUTING -s 10.90.0.0/24 -o starlink0 -j MASQUERADE
```

---

## 🚀 V2Ray – Scenario B

```bash
bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)
```

Outbound:
- Direct
- Interface: starlink0

---

## ⚖️ Load & Capacity Management

- محدودسازی Bandwidth per-user
- Queue management (fq_codel)
- محدودسازی Session

---

## ⚡ Failover

- UPS برای Starlink و سرور
- مانیتورینگ لینک
- لینک جایگزین در صورت امکان

---

## 📊 Monitoring

- vnStat
- Netdata
- بررسی Latency و Packet Loss

---

## 🧠 جمع‌بندی نهایی

- Scenario A مناسب Shutdownهای رایج با Ingress فعال
- Scenario B تنها راه در Shutdown مطلق
- این دو سناریو مکمل هم هستند
