# Setup the Gitlab server

## Install packages
> Note: Should perform as root user

**Step 1**: Install required packaged

* Debian family

```bash
sudo sudo apt install ca-certificates curl openssh-server postfix tzdata perl
```

* RHEL family

```bash
sudo dnf install -y curl policycoreutils openssh-server openssh-clients
```

When you see the installation prompted, select `Internet site` then enter the server domain name that shall be used for sending an email.

Example: `git.mydomain.com`

**Step 2**: Add the Gitlab repo

* Debian family

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

* RHEL family

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

**Step 3**: Install gitlab community edition

For latest version

* Debian family

```bash
sudo apt install gitlab-ce
```

* RHEL family

```bash
sudo dnf install -y gitlab-ce
```

Or specitfic version

```bash
sudo apt install gitlab-ce=17.3.4-ce.0
```

```bash
sudo dnf install -y gitlab-ce=17.3.4-ce.0
```

When the installation is complete, you will see the results as below.

![gitlab-installed.png](/ops/assets/images/gitlab-Installed.png)

**Step 5**: Config gitlab

Now we should configure the URL that will be used to access our GitLab server.

>Note: Please read [Addtion knowledge](#addtion-understand-knowledge) to understand the configuration

Find and update file `gitlab.rb` in `/etc/gitlab`

- Update **external_url**

    Let assume we use `gitlab.mydomain.com` as the domain for gitlab server

    ```rb
    external_url 'https://git.mydomain.com'
    ```

- Config nginx to forward gitlab to external, find and update content in file `gitlab.rb` as the content below

    ```rb
    nginx['redirect_http_to_https_port'] = false
    nginx['listen_port'] = 8081
    nginx['listen_addresses'] = ['127.0.0.1', "[::1]"]
    nginx['custom_gitlab_server_config'] = "location ^~/.well-known {\n alias /opt/gitlab/embedded/service/gitlab-rails/public/.well-known;\n}\n"
    nginx['listen_https'] = false
    ```

**Step 6**: Launch service

```bash
sudo gitlab-ctl reconfigure
```

## Configure

**CI/CD** (Skip this if not using gitlab CI/CD)

1. Open your browser and access to `https://git.mydomain.com` (Took the above example)
2. In the web UI, navigating to `Admin area` -> `Settings` -> `CI/CD` and edit some fields as below
   - Maximum artifacts size (MB) -> 500MB
   - Default artifacts expiration -> 30d
   - Archive jobs -> 30d
3. Save the changed

## Nginx 

>Note: Please read [Addtion knowledge](#addtion-understand-knowledge) to understand the configuration

### Setup

**Step 1**: Install nginx

* Debian family

```bash
sudo apt update && sudo apt install nginx
```

* RHEL family

```bash
sudo dnf update && dnf install nginx
```

**Step 2**: Enable service

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### Config

#### Step 1: Add the ssl cert to allow https. 

Let's assume we already have the cert file with the name as `fullchain.pem` and the  key for cert as `privkey.pem`. 

- Create a folder in `/etc/nginx/ssl`

  ```bash
  mkdir /etc/nginx/ssl
  ```

- Move these two cert and key file to the created folder

  ```bash
  mv fullchain.pem privkey.pem /etc/nginx/ssl
  ```

#### Step 2: Narrow to the nginx config folder and create new config file

```bash
cd /etc/nginx/conf.d
vi gitlab.conf
```

#### Step 3: Update the config file with content as below

```conf
server {
  listen                 80;
  server_name            git.mydomain.com;
  return                 301 https://$server_name$request_uri;
}

server {
  listen                 443 ssl;
  server_name            git.mydomain.com;

  ssl                    on;
  ssl_certificate        /etc/nginx/ssl/fullchain.pem;
  ssl_certificate_key    /etc/nginx/ssl/privkey.pem;

  ssl_ciphers            "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
  ssl_protocols          TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_session_timeout    5m;

  access_log             /var/log/nginx/gitlab.access.log;
  error_log              /var/log/nginx/gitlab.error.log;

  client_max_body_size   2048M;

  location / {
    proxy_pass          http://localhost:8081;
    proxy_redirect      off;
    proxy_set_header    Host             $host;
    proxy_set_header    X-Real-IP        $remote_addr;
    proxy_set_header    X-Forwarded-For  $proxy_add_x_forwarded_for;
    proxy_set_header    X-Client-Verify  SUCCESS;
    proxy_set_header    X-Client-DN      $ssl_client_s_dn;
    proxy_set_header    X-SSL-Subject    $ssl_client_s_dn;
    proxy_set_header    X-SSL-Issuer     $ssl_client_i_dn;
    proxy_read_timeout  90;
    proxy_connect_timeout 90;
  }
}
```

#### Step 4: Enable the config

```bash
sudo nginx -s reload
```

Then we can navigate to the url https://git.mydomain.com

>Note: For the first time, gitlab will request to change the root account password

## Addtion understand knowledge 

For the new gitlab version, it's already embed nginx inside the configuration to forward the gitlab webservice. But in this document we will use the external nginx, because:

- It's easier to control
- Other services hosted in the same server can use this nginx

However, we must use the origin nginx, which is embeded in gitlab bundle to forward the webserver first. Then we can able to forward an nginx to this. See diagram below