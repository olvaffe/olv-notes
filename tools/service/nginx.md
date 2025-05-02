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

## ACME

- ACME, Automatic Certificate Management Environment
  - <https://datatracker.ietf.org/doc/html/rfc8555>
  - developed by Let's Encrypt
- ACME challenge types
  - CAs use ACME challenges to verify the ownership of a domain
  - `HTTP-01`
    - the owner of a domain gets a token from the ACME server
    - the owner puts up a file at
      `http://<DOMAIN>/.well-known/acme-challenge/<TOKEN>`
    - the owner informs the ACME server
    - the ACM server retrieves and validates the file
    - pros/cons
      - tcp port 80 must not be blocked
      - cannot be used for wildcard certificates
  - `DNS-01`
    - the owner of a domain gets a token from the ACME server
    - the owner adds a `TXT` record for `_acme-challenge.<DOMAIN>`, and stores
      the token in the record
    - the owner informs the ACME server
    - the ACM server performs a DNS lookup
    - pros/cons
      - dns provider must support dynamic txt updates
      - can be used for wildcard certificates
  - `TLS-SNI-01`, deprecated due to security issues
    - it uses TLS handshake with SNI header on tcp port 443
  - `TLS-ALPN-01`
    - this is similar to `TLS-SNI-01` but uses ALPN

## `acme.sh`

- <https://github.com/acmesh-official/acme.sh>
- `acme.sh --install` installs `acme.sh`
  - it copies itself to `~/.acme.sh/acme.sh`
  - it adds `alias acme.sh=~/.acme.sh/acme.sh`
  - it adds a cronjob to `~/.acme.sh/acme.sh --cron` daily
  - args
    - `--home`, defaults to `~/.acme.sh`, specifies the install dir
    - `--config-home`, defaults to `--home`, specifies the data (configs,
      certs, and keys) dir
    - `--cert-home`, defaults to `--config-home`, specifies the certs dir
    - `--accountemail` specifies the email used for CA account registration
    - `--accountkey` specifies the private key used for CA account
    - `--nocron` disables cronjob
- `acme.sh --issue` issues certificates
  - the CA may issue these files
    - `chain.pem` is the intermediate CA certificate
      - it is signed by the root CA
    - `cert.pem` is our domain certificate (public key)
      - it is signed by the intermediate CA
    - `privkey.pem` is our domain private key
    - `fullchain.pem` is `cert.pem` plus `chain.pem`
      - this and `privkey.pem` are usually what daemons need
    - `bundle.pem` is `fullchain.pem` plus `privkey.pem`
  - webroot mode: `HTTP-01` using existing webserver
    - `acme.sh --issue -d <domain> -w <website-root-dir>`
    - the script writes the token to
      `<website-root-dir>/.well-known/acme-challenge/<token>`
  - standalone mode: `HTTP-01` using a temp webserver
    - `acme.sh --issue -d <domain> --standalone`
    - the script starts a temporary web server for the challenge
  - standalone alpn mode: `TLS-ALPN-01` using a temp webserver
    - `acme.sh --issue -d <domain> --alpn`
    - the script starts a temporary web server for the challenge
  - dns api mode: `DNS-01`
    - `DuckDNS_Token="<token>" acme.sh --issue -d mydomain.duckdns.org \
                                 -d *.mydomain.duckdns.org --dns dns_duckdns`
    - the script tells the dns provider to update `TXT` record for the
      challenge
  - dns manual mode: `DNS-01` manually, mainly for testing
  - dns alias mode: `DNS-01`, without giving the script the dns api token
  - apache mode: `HTTP-01` using apache
    - `acme.sh --issue -d <domain> --apache`
    - the script updates/restores apache config directly
  - nginx mode: `HTTP-01` using nginx
    - `acme.sh --issue -d <domain> --nginx`
    - the script updates/restores nginx config directly
- `acme.sh --install-cert` installs certificates for use by daemons
  - `acme.sh --install-cert -d <domain> --key-file <dst-path-1> \
             --fullchain-file <dst-path-2> \
             --reloadcmd 'sudo systemcl reload nginx'`
- more args
  - `acme.sh --upgrade` performs self-upgrade
  - `acme.sh --renew --domain <domain> --force` forces renewal
  - `acme.sh --list` lists certs managed by `acme.sh`
    - `~/.acme.sh/<domain>/<domain>.conf`
  - `acme.sh --info` shows `~/.acme.sh/account.conf`
  - `acme.sh --info --domain <doman>` shows `~/.acme.sh/<domain./<domain>.conf`
  - `acme.sh --install-cronjob` installs a crontab job
    - `crontab -l` to confirm
  - `acme.sh --uninstall-cronjob` uninstalls a crontab job
  - `acme.sh --cron` simulates a crontab job run
    - it invokes `acme.sh --renew-all` which invokes `acme --renew` on each
      domain
    - renew performs `--issue` and `--install-cert`, using the args saved in
      `~/.acme.sh/<domain./<domain>.conf`
