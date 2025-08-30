## **۱. پیش‌نیازهای سرور**

* سیستم عامل لینوکس (اوبونتو / دبیان)
* حداقل ۲ گیگابایت رم و ۲ هسته CPU پیشنهاد می‌شود
* PostgreSQL ≥ ۱۳
* Ruby 3.2.x
* Node.js ≥ ۱۶
* Nginx
* Bundler و سایر Gems

---

## **۲. نصب پیش‌نیازهای سیستم**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 dirmngr sudo wget git build-essential \
  libssl-dev libreadline-dev zlib1g-dev postgresql postgresql-contrib \
  nginx nodejs yarn
```

---

## **۳. راه‌اندازی PostgreSQL**

1. ساخت کاربر PostgreSQL برای زماد:

```bash
sudo -u postgres psql
CREATE ROLE zammad WITH LOGIN PASSWORD 'securepassword';
ALTER ROLE zammad CREATEDB;
\q
```

2. ساخت پایگاه داده:

```bash
sudo -u postgres createdb zammad
sudo -u postgres psql -c "ALTER DATABASE zammad OWNER TO zammad;"
```

> **رفع مشکلات رایج:**
>
> * خطای `PG::InsufficientPrivilege` --> مطمئن شوید کاربر PostgreSQL اجازه `CREATEDB` دارد.
> * خطای `permission denied` --> قبل از ایجاد دیتابیس به کاربر `postgres` سوییچ کنید.

---

## **۴. نصب Ruby با rbenv**

```bash
sudo -u zammad -i
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
rbenv install 3.2.8
rbenv global 3.2.8
ruby -v
exit
```

> **مشکل رایج:** اگر Bundler با نسخه Ruby اشتباه نصب شده باشد، systemd نمی‌تواند آن را پیدا کند.

---

## **۵. نصب Zammad**

```bash
cd /opt
sudo git clone https://github.com/zammad/zammad.git
sudo chown -R zammad:zammad zammad
cd zammad
sudo -u zammad -i
bundle install --deployment --without development test
exit
```

---

## **۶. پیکربندی پایگاه داده زماد**

```bash
cd /opt/zammad
sudo -u zammad -i
RAILS_ENV=production bundle exec rails db:create
RAILS_ENV=production bundle exec rails db:migrate
RAILS_ENV=production bundle exec rails db:seed  # اختیاری
exit
```

> **مشکلات رایج:**
>
> * خطای `Could not locate Gemfile` --> مطمئن شوید در مسیر `/opt/zammad` هستید.
> * خطای `PG::InsufficientPrivilege` --> مالکیت دیتابیس و دسترسی‌های PostgreSQL را بررسی کنید.

---

## **۷. پیکربندی Puma**

فایل `/opt/zammad/config/puma.rb` را در صورت نیاز ویرایش کنید.

برای تست دستی:

```bash
sudo -u zammad -i
cd /opt/zammad
RAILS_ENV=production bundle exec puma -C config/puma.rb
```

مطمئن شوید Puma روی پورت مناسب (به طور پیش‌فرض ۹۲۹۲) در حال گوش دادن است.

---

## **۸. پیکربندی systemd برای Zammad**

ساخت فایل `/etc/systemd/system/zammad.service`:

```ini
[Unit]
Description=Zammad Rails Server
After=network.target

[Service]
Type=simple
User=zammad
WorkingDirectory=/opt/zammad
Environment=RAILS_ENV=production
Environment="RBENV_ROOT=/home/zammad/.rbenv"
Environment="PATH=/home/zammad/.rbenv/shims:/home/zammad/.rbenv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
ExecStart=/home/zammad/.rbenv/shims/bundle exec puma -C /opt/zammad/config/puma.rb
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

بارگذاری و فعال‌سازی سرویس:

```bash
sudo systemctl daemon-reload
sudo systemctl enable zammad
sudo systemctl start zammad
sudo systemctl status zammad
```

> **مشکل رایج:**
>
> * خطاای `bundler: command not found: puma` --> مسیر کامل bundle را در systemd مشخص کنید و محیط RBENV را تنظیم کنید.

---

## **۹. پیکربندی Nginx**

ساخت فایل `/etc/nginx/sites-available/zammad.conf`:

```nginx
upstream zammad-railsserver { server 127.0.0.1:9292; }
upstream zammad-websocket { server 127.0.0.1:6042; }

server {
    listen 80;
    server_name sub.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name sub.example.com;

    ssl_certificate /etc/letsencrypt/live/sub.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sub.example.com/privkey.pem;

    root /opt/zammad/public;
    client_max_body_size 50M;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://zammad-railsserver;
    }

    location /cable {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass http://zammad-railsserver;
    }
}
```

فعال‌سازی سایت و ریستارت Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/zammad.conf /etc/nginx/sites-enabled/zammad.conf
sudo nginx -t
sudo systemctl restart nginx
```

> **مشکلات رایج:**
>
> * خطای 502 Bad Gateway <-- Puma در حال اجرا نیست یا پورت اشتباه است
> * `خطای ERR_CONNECTION_REFUSED` --> فایروال، پورت اشتباه، یا Puma در حال اجرا نیست

---

## **۱۰. فایروال و پورت‌ها**

اطمینان از باز بودن پورت‌های ۸۰ و ۴۴۳:

```bash
sudo ufw allow 80,443/tcp
sudo ufw reload
```

---

## **۱۱. بررسی نصب**

```bash
curl -I https://127.0.0.1 -H "Host: sub.example.com"
```

نتیجه مورد انتظار: `HTTP/1.1 200 OK` یا ریدایرکت به صفحه ورود.

---

## **۱۲. مشکلات رایج و راه‌حل‌ها**

| مشکل                               | علت                                  | راه‌حل                                                               |
| ---------------------------------- | ------------------------------------ | -------------------------------------------------------------------- |
| `PG::InsufficientPrivilege`        | دسترسی کاربر دیتابیس کم است          | اجازه CREATEDB بدهید و مالکیت دیتابیس را بررسی کنید                  |
| `Could not locate Gemfile`         | مسیر اشتباه                          | `cd /opt/zammad`                                                     |
| `bundler: command not found: puma` | محیط systemd مسیر را نمی‌شناسد       | مسیر کامل bundle + محیط RBENV در systemd                             |
| 502 Bad Gateway                    | Puma اجرا نمی‌شود یا پورت اشتباه است | وضعیت سرویس Zammad و پورت‌ها را بررسی کنید                           |
| Nginx صفحه پیش‌فرض را نشان می‌دهد  | کانفیگ اشتباه                        | `/etc/nginx/sites-enabled/default` را حذف و zammad.conf را فعال کنید |
| خطای SSL در curl                   | mismatch بین هاست و گواهی            | از `-H "Host: your.domain"` و `curl -k` استفاده کنید                 |

---

✅ پس از طی این مراحل، **زماد به طور کامل نصب شده و از طریق دامنه شما روی HTTPS قابل دسترسی است**.

---

اگر بخواهید، می‌توانم یک **نمودار معماری کامل** هم برایتان بکشم که مسیر Nginx --> Puma --> Rails --> PostgreSQL با پورت‌ها و نقاط مهم خطا را نشان دهد. این نمودار خیلی به فهم سیستم و نگهداری آن کمک می‌کند.

می‌خواهید این نمودار را هم بسازم؟
