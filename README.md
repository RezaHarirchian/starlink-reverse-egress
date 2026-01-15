# Starlink Reverse Egress
A reverse egress architecture built around Starlink, enabling outbound internet connectivity for isolated or shutdown environments through outside-initiated connections, with support for scaling, load distribution, failover, and high user capacity.
🌐 اتصال VPS ایران به اینترنت آزاد در Shutdown کامل
معماری پایدار و قابل Scale برای بار کاربری بالا
📌 مقدمه

این مستند برای سناریویی نوشته شده که اینترنت وارد Shutdown کامل در لایه Routing بین‌الملل شده است؛
نه اختلال، نه محدودسازی، نه فیلترینگ.

به‌صورت دقیق یعنی:

VPS مستقر در ایران صرفاً دارای Route داخلی است

هیچ Default Route یا BGP Path به خارج از ASهای داخلی وجود ندارد

سرویس‌هایی نظیر DNS Tunneling، CDN، Cloudflare، Warp و روش‌های مشابه عملاً غیرقابل استفاده هستند

در این شرایط، فقط یک معماری اتصال صحیح و قابل اتکا وجود دارد.
هر راهکار دیگری یا مبتنی بر سوءبرداشت فنی است یا تبلیغات غیرواقعی.

🎯 هدف معماری

تأمین اینترنت آزاد برای VPS داخل ایران

خروجی ترافیک از طریق استارلینک

اتصال کاربران صرفاً به IP ایران

پشتیبانی از:

حالت Single Exit

حالت Load Balance چندخروجی

قابلیت مدیریت و سرویس‌دهی به تعداد بالای کاربر

برخورداری از Failover و Monitoring واقعی

🧠 اصل فنی مسئله (Core Networking Principle)

در شرایط Shutdown کامل:

❌ VPS ایران قادر به Initiate اتصال به خارج نیست

✅ نود خارج از کشور می‌تواند اتصال را به داخل Initiate کند

بنابراین:

Initiator اتصال: نود متصل به استارلینک

Receiver اتصال: VPS ایران

این الگو در مهندسی شبکه با عنوان
Reverse / Outside-Initiated Tunnel Architecture
شناخته می‌شود و تنها مدل علمی قابل اجرا در این سناریو است.

🗺 معماری کلی اتصال
User
  |
  |  (OpenVPN / WireGuard / V2Ray)
  |
VPS (Iran)
  |
  |  Reverse Tunnel (WireGuard)
  |
Starlink Node(s)
  |
Free Internet

🧩 پیش‌نیازها
VPS ایران

IP عمومی ایران

دسترسی صرفاً به اینترنت داخلی

Ubuntu 20.04 یا 22.04

دسترسی کامل root

نود استارلینک (الزامی)

یکی از موارد زیر:

سیستم لینوکسی پشت لینک استارلینک

روتر مبتنی بر Linux

VPS خارج از کشور که خروجی نهایی آن استارلینک باشد

بدون استارلینک، این معماری اساساً غیرقابل پیاده‌سازی است.

🔑 مرحله ۱ – هسته معماری: تانل معکوس با WireGuard

این تانل، لایه‌ی پایه کل سیستم است و تمام سرویس‌های کاربر روی آن سوار می‌شوند.

🖥 پیکربندی WireGuard روی VPS ایران
apt update
apt install wireguard iptables -y

wg genkey | tee /etc/wireguard/ir_private.key | wg pubkey > /etc/wireguard/ir_public.key


/etc/wireguard/wg0.conf

[Interface]
Address = 10.50.0.1/24
ListenPort = 443
PrivateKey = IR_PRIVATE_KEY

PostUp   = sysctl -w net.ipv4.ip_forward=1
PostUp   = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE

wg-quick up wg0
systemctl enable wg-quick@wg0


🔎 نکته کلیدی:
VPS ایران هیچ اتصال خروجی ایجاد نمی‌کند و صرفاً در حالت Listen قرار دارد.

🌍 پیکربندی WireGuard روی نود استارلینک (Initiator)
apt install wireguard -y
wg genkey | tee sl_private.key | wg pubkey > sl_public.key


/etc/wireguard/wg0.conf

[Interface]
Address = 10.50.0.2/24
PrivateKey = STARLINK_PRIVATE_KEY
DNS = 1.1.1.1

[Peer]
PublicKey = IR_PUBLIC_KEY
Endpoint = IR_VPS_IP:443
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 10

wg-quick up wg0


فعال‌سازی NAT خروجی:

sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE


✅ در این مرحله، VPS ایران دارای دسترسی کامل به اینترنت آزاد است.

👥 مرحله ۲ – روش‌های اتصال کاربر (Client Access)

تمام کاربران صرفاً به IP ایران متصل می‌شوند.
تفاوت فقط در نوع پروتکل دسترسی است.

🔐 روش اول: OpenVPN (عمومی و سازگار)
apt install openvpn easy-rsa -y


نمونه کانفیگ:

port 1194
proto udp
dev tun
server 10.60.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.1"
keepalive 10 60
persist-key
persist-tun


NAT خروجی:

iptables -t nat -A POSTROUTING -s 10.60.0.0/24 -o wg0 -j MASQUERADE

🔗 روش دوم: WireGuard Client (کارایی بالا)
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.70.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = IR_PUBLIC_KEY
Endpoint = IR_VPS_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15


NAT:

iptables -t nat -A POSTROUTING -s 10.70.0.0/24 -o wg0 -j MASQUERADE

🚀 روش سوم: V2Ray (VLESS – مناسب مقیاس بالا)
bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)


Inbound پیشنهادی:

Protocol: VLESS

Transport: TCP یا WebSocket

Port: 443

NAT خروجی:

iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE

⚖️ مرحله ۳ – Load Balance و Scale-Out Architecture

برای شرایطی که:

بار کاربری بالا می‌رود

یک لینک استارلینک کافی نیست

نیاز به Redundancy وجود دارد

🧠 منطق Load Balance

چند تانل معکوس مستقل

هر تانل متصل به یک نود استارلینک

تقسیم کاربران با Policy-Based Routing

کاملاً Session-Based (بدون ECMP تصادفی)

نمونه تقسیم کاربران
ip rule add from 10.60.0.0/25 table 1
ip rule add from 10.60.0.128/25 table 2

ip route add default dev wg1 table 1
ip route add default dev wg2 table 2


هر کاربر مسیر ثابت دارد و سشن‌ها پایدار می‌مانند.

⚡ مرحله ۴ – Failover

در صورت قطع یک لینک یا تانل، سرویس باید بدون مداخله انسانی بازیابی شود.

نمونه اسکریپت ساده:

#!/bin/bash
ping -c 3 10.50.0.2 > /dev/null
if [ $? -ne 0 ]; then
  wg-quick down wg0
  wg-quick up wg0
fi


Cron:

*/1 * * * * /usr/local/bin/check_tunnel.sh

📊 مرحله ۵ – Monitoring

حداقل مانیتورینگ ضروری:

WireGuard:

wg show


OpenVPN:

بررسی status log

V2Ray:

X-UI Dashboard

برای مقیاس بزرگ:

Netdata

Prometheus + Grafana

📈 ملاحظات تجربی مهم

UDP 443 پایدارترین انتخاب در شرایط اختلال است

در صورت Packet Loss، MTU = 1280

Load Balance تصادفی توصیه نمی‌شود

هر تانل باید مانیتور مستقل داشته باشد

بدون استارلینک، این معماری غیرممکن است

🧠 جمع‌بندی نهایی

این معماری:

در شرایط Shutdown قابل اجراست

قابل Scale و پایدار است

برای بار کاربری بالا طراحی شده است.
