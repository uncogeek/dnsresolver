# MasterResolver

**یک DNS resolver scanner با رابط گرافیکی، طراحی‌شده برای [MasterDnsVPN](https://github.com/masterking32/MasterDnsVPN).**

وقتی MasterDnsVPN را اجرا می‌کنید، ترافیک شما از طریق DNS resolver های باز tunnel می‌شود. پیدا کردن resolver هایی که واقعاً با domain سرور شما کار کنند وقت می‌برد. این ابزار هزاران IP را از یک pipeline شش‌مرحله‌ای رد می‌کند و فقط resolver هایی که از همه تست‌ها رد شده‌اند را تحویل می‌دهد — آماده برای قرار دادن در config کلاینت.

![MasterResolver Screenshot](screenshot.png)

---

## پس‌زمینه

ایده از [findns](https://github.com/mosajjal/findns) آمد؛ یک CLI scanner قوی نوشته‌شده با Go. MasterResolver همان مسیر را طی می‌کند اما یک رابط گرافیکی کامل، پشتیبانی از `.exe` ویندوز، و دو مرحله اضافه (تشخیص DNS hijacking و اندازه‌گیری EDNS payload) اضافه کرده که برای DNS tunneling اهمیت ویژه دارند. آخرین تست هم مستقیماً با MTU engine خود MasterDnsVPN اجرا می‌شود — نه تخمین، بلکه تست واقعی مسیر.

---

## Pipeline

هر IP که لود می‌کنید از شش مرحله به‌ترتیب رد می‌شود. IP های رد شده در هر مرحله حذف می‌شوند، پس تست‌های سنگین‌تر فقط روی تعداد کمی از IP های باقی‌مانده اجرا می‌شوند.

```
1. Ping (ICMP)
   Pre-filter. IP هایی که اصلاً پاسخ نمی‌دهند حذف می‌شوند.
   قابل skip شدن است — اکثر DNS resolver های عمومی ICMP را در firewall بلاک می‌کنند.

2. Port 53 Scan
   چک می‌کند که port 53 روی UDP/TCP باز است و هر DNS response ای برمی‌گرداند.

3. Recursion Check
   یک domain واقعی query می‌شود. Resolver باید یک پاسخ معتبر با حداقل
   یک record برگرداند — تأیید می‌کند که واقعاً resolve می‌کند نه فقط echo می‌فرستد.

4. NXDOMAIN / Hijack Detection
   یک subdomain تصادفی که وجود ندارد query می‌شود.
   یک resolver سالم NXDOMAIN (rcode=3) برمی‌گرداند.
   ISP hijacker ها یک IP جعلی برمی‌گردانند — اینجا فیلتر می‌شوند.
   این مرحله برای tunneling حیاتی است: resolver های hijacked packet های شما را خراب می‌کنند.

5. EDNS Payload Size
   یک EDNS0 OPT query ارسال می‌شود. حداکثر UDP payload ای که resolver
   اعلام می‌کند اندازه‌گیری می‌شود. بزرگ‌تر = داده بیشتر در هر DNS query = throughput بالاتر.
   Resolver هایی که زیر آستانه تنظیم‌شده هستند حذف می‌شوند.

6. Tunnel Test (MasterDnsVPN MTU Engine)
   تست واقعی end-to-end. از MTU engine خود کلاینت MasterDnsVPN برای اجرای
   MTU discovery handshake با domain شما استفاده می‌کند. فقط resolver هایی که
   هر دو negotiate upload MTU و download MTU را با موفقیت انجام دهند قبول می‌شوند.
```

Pipeline در چهار حالت اجرا می‌شود: **Full Scan Once**، **Loop**، **Interval**، یا **Refine**. حالت Refine نتایج قبول‌شده را دوباره به pipeline برمی‌گرداند تا کیفیت تثبیت شود.

---

## قابلیت‌ها

- Pipeline کامل شش‌مرحله‌ای با نمایش live progress برای هر مرحله
- هر تست می‌تواند مستقل اجرا شود — هر مرحله را انتخاب کنید و هر لیست IP ای بدهید
- دکمه "Pass on" نتایج یک تست را مستقیم به عنوان input تست بعدی ارسال می‌کند
- Replace-on-rerun: اجرای مجدد یک تست نتایج قدیمی را پاک می‌کند به جای append کردن
- Test Domain برای مراحل 2 تا 5 جدا از Tunnel domain است (مراحل 1 تا 6)
- تا چهار Tunnel domain slot با انتخاب radio — مفید وقتی سرور چند NS record دارد
- تنظیمات scanner (workers، timeout ها، EDNS threshold، domain ها) در `resolver_settings.json` ذخیره می‌شوند
- Config viewer: قبل از اجرای Tunnel Test مقادیر فعال `client_config.toml` در لاگ نمایش داده می‌شود
- Paste متن خام → استخراج IP → بارگذاری خودکار در scanner با یک کلیک
- Export نتایج به ازای هر تست یا کپی به clipboard
- به عنوان `.exe` مستقل ارائه می‌شود — نیازی به نصب Python روی سیستم مقصد نیست

---

## مقایسه با findns

| قابلیت | findns | MasterResolver |
|---|---|---|
| رابط | CLI (TUI) | GUI |
| Ping/ICMP | ✅ | ✅ (قابل skip) |
| Port 53 Scan | ✅ | ✅ |
| Recursion Check | ✅ | ✅ |
| NXDOMAIN Hijack Detection | ✅ | ✅ |
| EDNS Payload Size | ✅ | ✅ |
| DoH (DNS-over-HTTPS) | ✅ | ❌ |
| End-to-End Tunnel Test | dnstt / slipstream | MasterDnsVPN MTU engine |
| Windows `.exe` | ❌ | ✅ |
| تنظیمات persistent | ❌ | ✅ |
| Export نتایج به ازای هر تست | ❌ | ✅ |

---

## پیش‌نیازها (اجرا از سورس)

```
Python 3.10+
PyQt6
loguru
zstd (اختیاری، برای پشتیبانی از compression)
```

نصب dependencies:
```bash
pip install -r requirements.txt
```

اجرا:
```bash
python resolver_gui.pyw
```

---

## ساخت `.exe`

```bash
pip install pyinstaller

C:\Users\...\Python313\Scripts\pyinstaller.exe --onefile --windowed --name MasterResolver ^
  --icon assets/masterdnsvpn.ico ^
  --add-data "assets;assets" ^
  --add-data "dns_utils;dns_utils" ^
  --hidden-import dns_utils.ARQ ^
  --hidden-import dns_utils.DNSBalancer ^
  --hidden-import dns_utils.DNS_ENUMS ^
  --hidden-import dns_utils.DnsPacketParser ^
  --hidden-import dns_utils.DnsResponseCache ^
  --hidden-import dns_utils.PacketQueueMixin ^
  --hidden-import dns_utils.PingManager ^
  --hidden-import dns_utils.PrependReader ^
  --hidden-import dns_utils.compression ^
  --hidden-import dns_utils.config_loader ^
  --hidden-import dns_utils.utils ^
  --hidden-import loguru ^
  resolver_gui.pyw
```

خروجی در `dist/MasterResolver.exe` خواهد بود. این فایل‌ها را کنار آن قرار دهید:

```
dist/
├── MasterResolver.exe
├── client_config.toml
├── build_version.py
├── resolvers/       ← لیست IP های .txt خود را اینجا بگذارید
└── resolved/        ← resolver های تأیید شده اینجا ذخیره می‌شوند
```

---

## لینک‌های مرتبط

- [MasterDnsVPN](https://github.com/masterking32/MasterDnsVPN) — کلاینت tunnel که این ابزار برای آن ساخته شده
- [findns](https://github.com/mosajjal/findns) — CLI scanner که الهام‌بخش طراحی pipeline بود

---

## لایسنس

MIT
