# Hiveway Production Deployment Guide

**Disclaimer:**

This guide was written for [Ubuntu Server 16.04](https://www.ubuntu.com/server), you may run into issues if you are using another operating system. We are welcoming contributions for guides to other distributions.

This document is also written with the expectation that you have a technical level high enough to administrate Linux servers.

## What is this guide?

This guide is a walk through of the setup process of a [Hiveway](https://github.com/hiveway/hiveway/) instance.

We use example.com to represent a domain or sub-domain. Example.com should be replaced with your instance domain or sub-domain.

## Prerequisites

You will need the following for this guide:

- A server running [Ubuntu Server 16.04](https://www.ubuntu.com/server).
- Root access to the server.
- A domain or sub-domain to use for the instance.

## DNS

DNS records should be added before anything is done on the server.

The records added are:

-  A record (IPv4 address) for example.com
-  AAAA record (IPv6 address) for example.com

> ### A Helpful And Optional Note
>
> Using `tmux` when following through with this guide will be helpful.
>
>
> Not only will this help you not lose your place if you are disconnected, it will let you have multiple terminal windows open for switching contexts (root user versus the hiveway user).
>
> You can install [tmux](https://github.com/tmux/tmux/wiki) from the package manager:
>
> ```sh
> apt -y install tmux
> ```

## Dependency Installation

All dependencies should be installed as root.

### node.js Repository

You will need to add an external repository so we can have the version of [node.js](https://nodejs.org/en/) required.

We run this script to add the repository:

```sh
apt -y install curl
curl -sL https://deb.nodesource.com/setup_6.x | bash -
```

The [node.js](https://nodejs.org/en/) repository is now added.

###  Yarn Repository

Another repository needs to be added so we can get the version of [Yarn](https://yarnpkg.com/en/) used by [Hiveway](https://github.com/hiveway/hiveway/).

This is how you add the repository:

```sh
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt update
```

### Various Other Dependencies

Now you need to install [Yarn](https://yarnpkg.com/en/) plus some more software.

#### Explanation of the dependencies

- imagemagick - Hiveway uses imagemagick for image related operations
- ffmpeg - Hiveway uses ffmpeg for conversion of GIFs to MP4s
- libprotobuf-dev and protobuf-compiler - Hiveway uses these for language detection
- nginx - nginx is our frontend web server
- redis-* - Hiveway uses redis for its in-memory data structure store
- postgresql-* - Hiveway uses PostgreSQL as its SQL database
- nodejs - Node is used for Hiveway's streaming API
- yarn - Yarn is a Node.js package manager
- Other -dev packages, g++ - these are needed for the compilation of Ruby using ruby-build.

```sh
apt -y install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm3 libgdbm-dev nginx redis-server redis-tools postgresql postgresql-contrib letsencrypt yarn libidn11-dev libicu-dev
```

### Dependencies That Need To Be Added As A Non-Root User

Let us create this user first:

```sh
adduser hiveway
```

Log in as the `hiveway` user:


```sh
sudo su - hiveway
```

We will need to set up [`rbenv`](https://github.com/rbenv/rbenv) and [`ruby-build`](https://github.com/rbenv/ruby-build):

```sh
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
# Restart shell
exec bash
# Check if rbenv is correctly installed
type rbenv
# Install ruby-build as rbenv plugin
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

Now that [`rbenv`](https://github.com/rbenv/rbenv) and [`ruby-build`](https://github.com/rbenv/ruby-build) are installed, we will install the
[Ruby](https://www.ruby-lang.org/en/) version which [Hiveway](https://github.com/hiveway/hiveway/) uses. That version will also need to be enabled.

To enable [Ruby](https://www.ruby-lang.org/en/), run:

```sh
rbenv install 2.5.0
rbenv global 2.5.0
```

**This will take some time. Go stretch for a bit and drink some water while the commands run.**

### node.js And Ruby Dependencies

Now that [Ruby](https://www.ruby-lang.org/en/) is enabled, we will clone the [Hiveway GIT repository](https://github.com/hiveway/hiveway/) and install the [Ruby](https://www.ruby-lang.org/en/) and [node.js](https://nodejs.org/en/) dependancies.

Run the following to clone and install:

```sh
# Return to hiveway user's home directory
cd ~
# Clone the hiveway git repository into ~/live
git clone https://github.com/hiveway/hiveway.git live
# Change directory to ~live
cd ~/live
# Checkout to the latest stable branch
git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)
# Install bundler
gem install bundler
# Use bundler to install the rest of the Ruby dependencies
bundle install -j$(getconf _NPROCESSORS_ONLN) --deployment --without development test
# Use yarn to install node.js dependencies
yarn install --pure-lockfile
```

That is all we need to do for now with the `hiveway` user, you can now `exit` back to root.

## PostgreSQL Database Creation

[Hiveway](https://github.com/hiveway/hiveway/) requires access to a [PostgreSQL](https://www.postgresql.org) instance.

Create a user for a [PostgreSQL](https://www.postgresql.org) instance:

```
# Launch psql as the postgres user
sudo -u postgres psql

# In the following prompt
CREATE USER hiveway CREATEDB;
\q
```

**Note** that we do not set up a password of any kind, this is because we will be using ident authentication. This allows local users to access the database without a password.

## nginx Configuration

You need to configure [nginx](http://nginx.org) to serve your [Hiveway](https://github.com/hiveway/hiveway/) instance.

**Reminder: Replace all occurrences of example.com with your own instance's domain or sub-domain.**

`cd` to `/etc/nginx/sites-available` and open a new file:

`nano /etc/nginx/sites-available/example.com.conf`

Copy and paste the following and make edits as necessary:

```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name example.com;
  root /home/hiveway/live/public;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name example.com;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  keepalive_timeout    70;
  sendfile             on;
  client_max_body_size 0;

  root /home/hiveway/live/public;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location ~ ^/(emoji|packs|system/accounts/avatars|system/media_attachments/files) {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri @proxy;
  }
  
  location /sw.js {
    add_header Cache-Control "public, max-age=0";
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://127.0.0.1:3000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  location /api/v1/streaming {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";

    proxy_pass http://127.0.0.1:4000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}
```

Activate the [nginx](http://nginx.org) configuration added:

```sh
cd /etc/nginx/sites-enabled
ln -s ../sites-available/example.com.conf
```

This configuration makes the assumption you are using [Let's Encrypt](https://letsencrypt.org) as your TLS certificate provider.

**If you are going to be using Let's Encrypt as your TLS certificate provider, see the
next sub-section. If not edit the `ssl_certificate` and `ssl_certificate_key` values
accordingly.**

## Let's Encrypt

This section is only relevant if you are using [Let's Encrypt](https://letsencrypt.org/)
as your TLS certificate provider.

### Generation Of The Certificate

We need to generate Let's Encrypt certificates.

**Make sure to replace any occurrence of 'example.com' with your Hiveway instance's domain.**

Make sure that [nginx](http://nginx.org) is stopped at this point:

```sh
systemctl stop nginx
```

We will be creating the certificate twice, once with TLS SNI validation in standalone mode and the second time we will be using the webroot method. This is required due to the way
[nginx](http://nginx.org) and the [Let's Encrypt](https://letsencrypt.org/) tool works.

```sh
letsencrypt certonly --standalone -d example.com
```

After that successfully completes, we will use the webroot method. This requires [nginx](http://nginx.org) to be running:

```sh
systemctl start nginx
# The letsencrypt tool will ask if you want issue a new cert, please choose that option
letsencrypt certonly --webroot -d example.com -w /home/hiveway/live/public/
```

### Automated Renewal Of Let's Encrypt Certificate

[Let's Encrypt](https://letsencrypt.org/) certificates have a validity period of 90 days.

You need to renew your certificate before the expiration date. Not doing so will make users of your instance unable to access the instance and users of other instances unable to federate with yours.

We can create a cron job that runs daily to do this:

```sh
nano /etc/cron.daily/letsencrypt-renew
```

Copy and paste this script into that file:

```sh
#!/usr/bin/env bash
letsencrypt renew
systemctl reload nginx
```

Save and exit the file.

Make the script executable and restart the cron daemon so that the script runs daily:

```sh
chmod +x /etc/cron.daily/letsencrypt-renew
systemctl restart cron
```

That is it. Your server will renew your [Let's Encrypt](https://letsencrypt.org/) certificate.

## Hiveway Application Configuration

We will configure the Hiveway application.

For this we will switch to the `hiveway` system user:


```sh
sudo su - hiveway
```

Change directory to `~live` and edit the [Hiveway](https://github.com/hiveway/hiveway/) application configuration:

```sh
cd ~/live
cp .env.production.sample .env.production
nano .env.production
```

For the purposes of this guide, these are the values to be edited:

```
# Your Redis host
REDIS_HOST=127.0.0.1
# Your Redis port
REDIS_PORT=6379
# Your PostgreSQL host
DB_HOST=/var/run/postgresql
# Your PostgreSQL user
DB_USER=hiveway
# Your PostgreSQL DB name
DB_NAME=hiveway_production
# Leave DB password empty
DB_PASS=
# Your DB_PORT
DB_PORT=5432

# Your instance's domain
LOCAL_DOMAIN=example.com
# We have HTTPS enabled
LOCAL_HTTPS=true

# Application secrets
# Generate each with `RAILS_ENV=production bundle exec rake secret`
PAPERCLIP_SECRET=
SECRET_KEY_BASE=
OTP_SECRET=

# Web Push VAPID keys
# Generate with `RAILS_ENV=production bundle exec rake hiveway:webpush:generate_vapid_key`
VAPID_PRIVATE_KEY=
VAPID_PUBLIC_KEY=

# All SMTP details, Mailgun and Sparkpost have free tiers
SMTP_SERVER=
SMTP_PORT=
SMTP_LOGIN=
SMTP_PASSWORD=
SMTP_FROM_ADDRESS=
```

We now need to set up the [PostgreSQL](https://www.postgresql.org) database for the first time:

```sh
RAILS_ENV=production bundle exec rails db:setup
```

Then we will need to precompile all CSS and JavaScript files:

```sh
RAILS_ENV=production bundle exec rails assets:precompile
```

**The assets precompilation takes a couple minutes, so this is a good time to take another break.**

## Hiveway systemd Service Files

We will need three [systemd](https://github.com/systemd/systemd) service files for each Hiveway service.

Now switch back to the root user.

For the [Hiveway](https://github.com/hiveway/hiveway/) web workers service place the following in `/etc/systemd/system/hiveway-web.service`:

```
[Unit]
Description=hiveway-web
After=network.target

[Service]
Type=simple
User=hiveway
WorkingDirectory=/home/hiveway/live
Environment="RAILS_ENV=production"
Environment="PORT=3000"
ExecStart=/home/hiveway/.rbenv/shims/bundle exec puma -C config/puma.rb
ExecReload=/bin/kill -SIGUSR1 $MAINPID
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

For [Hiveway](https://github.com/hiveway/hiveway/) background queue service, place the following in `/etc/systemd/system/hiveway-sidekiq.service`:

```
[Unit]
Description=hiveway-sidekiq
After=network.target

[Service]
Type=simple
User=hiveway
WorkingDirectory=/home/hiveway/live
Environment="RAILS_ENV=production"
Environment="DB_POOL=5"
ExecStart=/home/hiveway/.rbenv/shims/bundle exec sidekiq -c 5 -q default -q mailers -q pull -q push
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

For the [Hiveway](https://github.com/hiveway/hiveway/) streaming API service place the following in `/etc/systemd/system/hiveway-streaming.service`:

```
[Unit]
Description=hiveway-streaming
After=network.target

[Service]
Type=simple
User=hiveway
WorkingDirectory=/home/hiveway/live
Environment="NODE_ENV=production"
Environment="PORT=4000"
ExecStart=/usr/bin/npm run start
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

Now you need to enable all of these services:

```sh
systemctl enable /etc/systemd/system/hiveway-*.service
```

Now start the services:

```sh
systemctl start hiveway-streaming.service
systemctl start hiveway-sidekiq.service
systemctl start hiveway-web.service
```

Check that they are properly running:

```sh
systemctl status hiveway-*.service
```

## Remote media attachment cache cleanup

hiveway downloads media attachments from other instances and caches it locally for viewing. This cache can grow quite large if
not cleaned up periodically and can cause issues such as low disk space or a bloated S3 bucket.

The recommended method to clean up the remote media cache is a cron job that runs daily like so (put this in the hiveway system user's crontab with `crontab -e`.)

```sh
RAILS_ENV=production
@daily cd /home/hiveway/live && /home/hiveway/.rbenv/shims/bundle exec rake hiveway:media:remove_remote
```

That rake task removes cached remote media attachments that are older than NUM_DAYS, NUM_DAYS defaults to 7 days (1 week) if not specified. NUM_DAYS is another environment variable so you can specify it like so:

```sh
RAILS_ENV=production
NUM_DAYS=14
@daily cd /home/hiveway/live && /home/hiveway/.rbenv/shims/bundle exec rake hiveway:media:remove_remote
```

## Email Service

If you plan on receiving email notifications or running more than just a single-user instance, you likely will want to get set up with an email provider.

There are several free email providers out there- a couple of decent ones are Mailgun.com, which requires a credit card but gives 10,000 free emails, and Sparkpost.com, which gives 15,000 with no credit card but requires you not be on a .space tld.

It may be easier to use a subdomain to setup your email with a custom provider - in this case, when registering your domain with the email service, sign up as something like "mail.domain.com"

Once you create your account, follow the instructions each provider gives you for updating your DNS records.  Once you have all the information ready to go and the service validates your DNS configuration, edit your config file.  These records should already exist in the configuration, but here's a sample setup that uses Mailgun that you can replace with your own personal info:

SMTP_SERVER=smtp.mailgun.org
SMTP_PORT=587
SMTP_LOGIN=anAccountThatIsntPostmaster@mstdn.domain.com
SMTP_PASSWORD=HolySnacksAPassword
SMTP_FROM_ADDRESS=Domain.com Hiveway Admin <notifications@domain.com>


That is all! If everything was done correctly, an [Hiveway](https://github.com/hiveway/hiveway/) instance will appear when you visit `https://example.com` in a web browser.

Congratulations !
