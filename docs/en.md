## **1. Server Requirements**

* Ubuntu/Debian-based Linux system
* At least 2 GB RAM, 2 CPU cores recommended
* PostgreSQL ≥ 13
* Ruby 3.2.x
* Node.js ≥ 16
* Nginx
* Bundler and other gems

---

## **2. Install System Dependencies**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 dirmngr sudo wget git build-essential \
  libssl-dev libreadline-dev zlib1g-dev postgresql postgresql-contrib \
  nginx nodejs yarn
```

---

## **3. Set Up PostgreSQL**

1. Create a PostgreSQL user for Zammad:

```bash
sudo -u postgres psql
CREATE ROLE zammad WITH LOGIN PASSWORD 'securepassword';
ALTER ROLE zammad CREATEDB;
\q
```

2. Create the database:

```bash
sudo -u postgres createdb zammad
sudo -u postgres psql -c "ALTER DATABASE zammad OWNER TO zammad;"
```

> **Troubleshooting:**
>
> * `PG::InsufficientPrivilege` → Ensure your PostgreSQL user has `CREATEDB` rights.
> * `permission denied` → Switch to `postgres` user before creating DB.

---

## **4. Install Ruby via rbenv**

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

> **Common Issue:** Bundler installed under one Ruby version may not be found by systemd if another Ruby is default.

---

## **5. Install Zammad**

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

## **6. Configure Zammad Database**

```bash
cd /opt/zammad
sudo -u zammad -i
RAILS_ENV=production bundle exec rails db:create
RAILS_ENV=production bundle exec rails db:migrate
RAILS_ENV=production bundle exec rails db:seed  # optional
exit
```

> **Common Issues:**
>
> * `Could not locate Gemfile` → Make sure you `cd /opt/zammad`
> * `PG::InsufficientPrivilege` → Check DB owner and PostgreSQL user permissions

---

## **7. Configure Puma**

Edit `/opt/zammad/config/puma.rb` if needed.

Start manually:

```bash
sudo -u zammad -i
cd /opt/zammad
RAILS_ENV=production bundle exec puma -C config/puma.rb
```

Check that Puma listens on the intended port (default 9292).

---

## **8. Configure systemd for Zammad**

Create `/etc/systemd/system/zammad.service`:

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

Reload and start service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable zammad
sudo systemctl start zammad
sudo systemctl status zammad
```

> **Common Issue:**
>
> * `bundler: command not found: puma` → Add full path to bundle (`/home/zammad/.rbenv/shims/bundle`) and include RBENV environment in systemd.

---

## **9. Configure Nginx**

Create `/etc/nginx/sites-available/zammad.conf`:

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

Enable site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/zammad.conf /etc/nginx/sites-enabled/zammad.conf
sudo nginx -t
sudo systemctl restart nginx
```

> **Common Issues:**
>
> * 502 Bad Gateway → Puma not running, or wrong upstream port
> * `ERR_CONNECTION_REFUSED` → Firewall, port mismatch, or Puma not bound

---

## **10. Firewall & Ports**

Ensure ports 80 & 443 are open:

```bash
sudo ufw allow 80,443/tcp
sudo ufw reload
```

---

## **11. Verify Installation**

```bash
curl -I https://127.0.0.1 -H "Host: sub.example.com"
```

Expected: `HTTP/1.1 200 OK` (or redirect to login page)

---

## **12. Common Pitfalls & Fixes**

| Issue                              | Cause                             | Solution                                                       |
| ---------------------------------- | --------------------------------- | -------------------------------------------------------------- |
| `PG::InsufficientPrivilege`        | DB user lacks permissions         | Grant CREATEDB and ensure ownership                            |
| `Could not locate Gemfile`         | Wrong working directory           | `cd /opt/zammad`                                               |
| `bundler: command not found: puma` | systemd environment missing paths | Use full paths + RBENV env in systemd                          |
| 502 Bad Gateway                    | Puma not running / wrong port     | Check `systemctl status zammad` & `lsof -i :9292`              |
| Nginx shows default page           | Wrong site config                 | Remove `/etc/nginx/sites-enabled/default` and link zammad.conf |
| SSL curl errors                    | Certificate hostname mismatch     | Use `-H "Host: your.domain"` with `curl -k`                    |

---

✅ After following these steps, **Zammad should be fully installed and accessible via your domain over HTTPS**.

---

If you want, I can also make a **diagram showing the full architecture**: Nginx → Puma → Rails → PostgreSQL, with ports and troubleshooting points. That makes the document super clear for team or future reference.

Do you want me to make that diagram too?
