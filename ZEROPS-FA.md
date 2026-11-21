# اجرای PasarGuard Node روی app.zerops.io

این نسخه برای اجرای PasarGuard Node در Zerops آماده شده است. در حالت Zerops، سرویس به‌صورت REST روی HTTP داخلی اجرا می‌شود و TLS در بالانسر عمومی Zerops خاتمه پیدا می‌کند.

## تنظیمات سرویس

در Zerops یک Runtime Service با hostname برابر `node` بسازید و Repository را به این مخزن وصل کنید. فایل `zerops.yaml` پورت‌های زیر را تعریف می‌کند:

- `8080/TCP`: API اصلی REST و health check. این پورت را در بخش HTTP Balancer به دامنه یا زیردامنه وصل کنید.
- `62050/TCP`: پورت اختیاری gRPC/TLS برای دسترسی مستقیم. فقط در صورت نیاز آن را Public Port کنید.
- `8081` تا `8090/TCP`: پورت‌های اختیاری Xray inbound. فقط پورت‌هایی را Public کنید که در کانفیگ inbound واقعاً استفاده می‌شوند.

متغیرهای محیطی پیشنهادی:

```env
NODE_HOST=0.0.0.0
SERVICE_PORT=8080
SERVICE_PROTOCOL=rest
TLS_ENABLED=false
API_KEY=<UUID تصادفی و غیرصفر>
GENERATED_CONFIG_PATH=/var/lib/pg-node/generated
XRAY_EXECUTABLE_PATH=/usr/local/bin/xray
XRAY_ASSETS_PATH=/usr/local/share/xray
PG_NODE_WG_HOST_ROUTING=false
```

`API_KEY` را در Secrets/Environment Variables خود Zerops وارد کنید و آن را در Git commit نکنید. برای ساخت UUID:

```bash
python3 -c 'import uuid; print(uuid.uuid4())'
```

## اتصال از طریق URL عمومی

پس از ساخت route برای پورت `8080`، آدرس API به شکل زیر خواهد بود:

```text
https://<service-domain>/info
```

تمام درخواست‌های API باید هدر زیر را داشته باشند:

```text
x-api-key: <API_KEY>
```

Health check مسیر `GET /healthz` است و احراز هویت نمی‌خواهد. مسیر `/info` و سایر APIها احراز هویت می‌خواهند.

## پورت‌های Xray و پنل سنایی

پورت `8080` فقط برای API نود است و نباید به‌عنوان inbound سنایی استفاده شود. برای inboundهای Xray از پورت‌های `8081` تا `8090` استفاده کنید و همان پورت را در Zerops به‌عنوان Public Port به مقصد داخلی همان پورت map کنید. در پنل سنایی، آدرس نود را با دامنه‌ی Zerops و پورت public متناظر وارد کنید.

اگر Zerops در پروژه‌ی شما فقط دسترسی HTTP دارد، inboundهای TCP/UDP و gRPC عمومی مناسب نیستند؛ در آن حالت از transportهای HTTP/WebSocket/XHTTP پشت route پورت `8080` استفاده کنید. برای gRPC/TLS واقعی، از Direct TCP Access روی `62050` یا یک IPv4 اختصاصی استفاده کنید.

## اجرای VPS بدون Zerops

رفتار اجرای عادی حفظ شده است. `docker-compose.yml` همچنان روی `62050` با `grpc` اجرا می‌شود و TLS را نگه می‌دارد. پس از تغییرات، برای اجرای مستقیم یا VPS مقدارهای زیر را صریح تنظیم کنید:

```env
SERVICE_PORT=62050
SERVICE_PROTOCOL=grpc
TLS_ENABLED=true
```

## نکته‌ی ذخیره‌سازی

مسیرهای `/var/lib/pg-node/certs` و `/var/lib/pg-node/generated` در کانتینرهای Zerops بدون Local Storage پایدار نیستند. برای production، یک Local Storage service وصل کنید و آن را روی `/var/lib/pg-node` mount کنید؛ یا برای TLS مستقیم، گواهی را در Secret/volume پایدار قرار دهید.
