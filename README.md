# استقرار Traefik، WhoAmI و cAdvisor با GitLab CI/CD

## معرفی

این پروژه یک زیرساخت ساده مبتنی بر Docker Compose را پیاده‌سازی می‌کند که شامل موارد زیر است:

* **Traefik** به عنوان Reverse Proxy و Load Balancer
* **سه سرویس WhoAmI** به عنوان Backend
* **cAdvisor** برای مانیتورینگ کانتینرها
* **GitLab CI/CD** برای استقرار خودکار

## معماری

```text
                Internet
                    │
                    ▼
              ┌─────────┐
              │ Traefik │
              └────┬────┘
                   │
         ┌─────────┴─────────┐
         │                   │
         ▼                   ▼
   whoami-service      cadvisor-service

         ┌───────┬───────┬───────┐
         ▼       ▼       ▼
      whoami1 whoami2 whoami3
```

### ساختار شبکه

* شبکه `web` برای ارتباط با اینترنت
* شبکه `app` به صورت داخلی (`internal: true`)
* تنها سرویس Traefik به هر دو شبکه متصل است
* سرویس‌های Backend به صورت مستقیم به اینترنت دسترسی ندارند

---

## سرویس‌ها

### Traefik

وظایف:

* دریافت درخواست‌های HTTP و HTTPS
* توزیع درخواست‌ها بین سرویس‌های WhoAmI
* انتشار رابط مانیتورینگ cAdvisor

پورت‌های منتشرشده:

* HTTP: 80
* HTTPS: 443

---

### WhoAmI

سه نمونه از سرویس WhoAmI اجرا می‌شوند:

* whoami1
* whoami2
* whoami3

Traefik درخواست‌ها را به صورت Load Balance بین آن‌ها توزیع می‌کند.

نمونه پاسخ:

```text
Hostname: whoami1
```

---

### cAdvisor

امکانات:

* نمایش مصرف CPU
* نمایش مصرف حافظه
* نمایش مصرف دیسک
* نمایش آمار شبکه
* مانیتورینگ تمامی کانتینرهای Docker

سرویس cAdvisor مستقیماً در اینترنت منتشر نشده و فقط از طریق Traefik در دسترس است.

---

## اجرای پروژه

برای راه‌اندازی سرویس‌ها:

```bash
docker compose up -d
```

بررسی وضعیت کانتینرها:

```bash
docker ps
```

مشاهده لاگ‌ها:

```bash
docker logs traefik
docker logs cadvisor
```

---

## مسیریابی (Routing)

### دسترسی به WhoAmI

```text
http://whoami.local
```

### دسترسی به cAdvisor

```text
http://cadvisor.local
```

### تنظیم فایل Hosts

در لینوکس و macOS:

```text
127.0.0.1 whoami.local
127.0.0.1 cadvisor.local
```

در ویندوز:

```text
C:\Windows\System32\drivers\etc\hosts
```

محتوا:

```text
127.0.0.1 whoami.local
127.0.0.1 cadvisor.local
```

---

## استقرار خودکار با GitLab CI/CD

### هدف

با هر Push روی شاخه اصلی (`main`)، سرویس‌ها به صورت خودکار به‌روزرسانی و مجدداً اجرا شوند.

---

## پیش‌نیازهای GitLab Runner

Runner باید دارای موارد زیر باشد:

* دسترسی به سرور به وسیله ssh با جفت کلید
* در فایل .env متغیرهای سرور را قرار میدهیم(بهتر است متغیرها را در gitlab ست کنیم)
* Docker
* Docker Compose
* دسترسی اجرای دستورات Docker

بررسی نصب:

```bash
docker version
docker compose version
```

---

## نمونه فایل `.gitlab-ci.yml`

```yaml
stages:
  - deploy

deploy:
  stage: deploy
  image: alpine:latest

  before_script:
    - apk add --no-cache openssh-client git

    - mkdir -p ~/.ssh

    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa

    - ssh-keyscan -H $SSH_HOST >> ~/.ssh/known_hosts

  script:
    - |
      ssh $SSH_USER@$SSH_HOST "
        set -e

        if [ ! -d '$DEPLOY_PATH/.git' ]; then
          mkdir -p '$DEPLOY_PATH'
          git clone '$CI_REPOSITORY_URL' '$DEPLOY_PATH'
        fi

        cd '$DEPLOY_PATH'

        git fetch origin
        git reset --hard origin/main

        docker compose pull
        docker compose up -d

        docker image prune -f
      "

  only:
    - main
```

---

## فرآیند استقرار

1. تغییرات در مخزن ثبت می‌شوند:

```bash
git add .
git commit -m "Update configuration"
git push origin main
```

2. GitLab به صورت خودکار Pipeline را اجرا می‌کند.

3. GitLab Runner دستورات زیر را اجرا می‌کند:

```bash
docker compose pull
docker compose up -d
```

4. سرویس‌های جدید بدون دخالت دستی در دسترس قرار می‌گیرند.

---

## اعتبارسنجی

بررسی سرویس WhoAmI:

```bash
curl http://whoami.local
```

نمونه خروجی:

```text
Hostname: whoami1
```

در درخواست‌های متوالی باید پاسخ بین سرویس‌ها جابجا شود:

```text
Hostname: whoami1
Hostname: whoami2
Hostname: whoami3
```

بررسی cAdvisor:

```text
http://cadvisor.local
```

---

## عیب‌یابی

### بررسی وضعیت کانتینرها

```bash
docker ps
```

### مشاهده لاگ Traefik

```bash
docker logs traefik
```

### مشاهده لاگ cAdvisor

```bash
docker logs cadvisor
```

### بررسی عضویت کانتینرها در شبکه

```bash
docker network inspect app
```

### تست مستقیم cAdvisor

```bash
docker exec cadvisor wget -qO- http://localhost:8080
```

---

## نکات امنیتی

* تمامی سرویس‌های Backend روی شبکه داخلی اجرا می‌شوند.
* فقط Traefik از بیرون قابل دسترس است.
* cAdvisor صرفاً از طریق Traefik منتشر می‌شود.
* هیچ‌یک از سرویس‌های داخلی پورت مستقیمی روی میزبان (Host) باز نمی‌کنند.

این سناریو برای یادگیری مفاهیم Reverse Proxy، Load Balancing، شبکه‌بندی Docker، مانیتورینگ کانتینرها و استقرار خودکار با GitLab CI/CD مناسب است.
