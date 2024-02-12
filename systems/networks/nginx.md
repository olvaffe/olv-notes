nginx
=====

## HTTP

- it should just work
  - `apt install nginx`
  - `ls -l /etc/nginx/sites-enabled/default`

## HTTPS

- certificates
  - `curl https://get.acme.sh | sh -s email=my@example.com`
    - this installs `acme.sh` to `~/.acme.sh`
    - creates a shell alias, `acme.sh`
    - creates a cron task, `crontab -l`
  - issue a certificate
    - `acme.sh --issue -d my.example.com -w /var/www/html`
    - this is the webroot mode
  - install a certificate
    - `acme.sh --install-cert -d my.example.com
         --key-file /etc/nginx/certs/my.example.com/key.pem
         --fullchain-file /etc/nginx/certs/my.example.com/cert.pem
         --reloadcmd "systemctl reload nginx"`
- configure nginx
  - edit `/etc/nginx/sites-enabled/default`
  - add
    - `listen 443 ssl default_server;`
    - `listen [::]:443 ssl default_server;`
    - `ssl_certificate /etc/nginx/certs/my.example.com/cert.pem;`
    - `ssl_certificate_key /etc/nginx/certs/my.example.com/key.pem;`
  - `systemctl reload nginx`
