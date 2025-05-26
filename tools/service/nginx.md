nginx
=====

## HTTP

- it should just work
  - `apt install nginx`
  - `ls -l /etc/nginx/sites-enabled/default`

## Config

- `location [modifier] [URI] { ... }`
  - without modifier, it performs prefix matchinig
  - `=` performs exact matching
  - `~` performs case-sensitive regex matching
  - `~*` performs case-insensitive regex matching
  - `^~` performs prefix matchinig (see below)
- given a request uri, nginx picks the matching block by
  - if a `=` block matches, use the block
  - if a prefeix matching block matches,
    - if `^~`, use the block
    - otherwise, remember the block but keeps searching
  - if a regex matching block matches, use the block
  - otherwise, use the remembered block

## PHP

- `apt install php-fpm`
  - make sure the service is enabled and the daemon is running
- `location ~ \.php$`
  - `include snippets/fastcgi-php.conf;`
  - `fastcgi_pass unix:/run/php/php-fpm.sock;`

## HTTPS

- certificates
  - `curl https://get.acme.sh | sh -s email=my@example.com`
    - this installs `acme.sh` to `~/.acme.sh` and registers an account with CA
  - edit `~/.acme.sh/account.conf` to add `DuckDNS_Token='<token>'`
    - this adds the duckdns api key
  - `acme.sh --issue -d mydomain.duckdns.org --dns dns_duckdns`
    - this issues a cert using DNS-01 mode
  - `mkdir -m 700 /etc/nginx/certs`
  - `acme.sh --install-cert -d mydomain.duckdns.org \
         --key-file /etc/nginx/certs/mydomain.duckdns.org.key \
         --fullchain-file /etc/nginx/certs/mydomain.duckdns.org.cer \
         --reloadcmd "systemctl reload nginx"`
    - this installs cert/privkey and reloads nginx
- configure nginx
  - edit `/etc/nginx/sites-enabled/default`
  - add
    - `listen 443 ssl default_server;`
    - `listen [::]:443 ssl default_server;`
    - `ssl_certificate /etc/nginx/certs/mydomain.duckdns.org.cer;`
    - `ssl_certificate_key /etc/nginx/certs/mydomain.duckdns.org.key`
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
    - `--email` specifies the email used for CA account registration
    - `--no-cron` disables cronjob
- `acme.sh --register-account -m my@example.com` registers an account with CA
  - account credential is at `~/.acme.sh/ca/<server>`
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
    - `DuckDNS_Token="<token>"` is saved to `~/.acme.sh/account.conf`, for
      cron renewal
    - certs are saved to `~/.acme.sh/<domain>_ecc`
      - `<domain>.conf` is the config, for cron renewal
      - `<domain>.csr` is the request
      - `ca.cer` is the CA certs (root CA and intermediate CAs)
      - `<domain>.cer` is the domain cert
      - `<domain>.key` is the private key
      - `fullchain.cer` is CA certs plus domain cert
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
  - `~/.acme.sh/<domain>_ecc/<domain>.conf` is updated for cron renewal
- more args
  - `acme.sh --upgrade` performs self-upgrade
  - `acme.sh --renew --domain <domain> --force` forces renewal
  - `acme.sh --list` lists certs managed by `acme.sh`
    - `~/.acme.sh/<domain>/<domain>.conf`
  - `acme.sh --info` shows `~/.acme.sh/account.conf`
  - `acme.sh --info --domain <doman>` shows `~/.acme.sh/<domain>_ecc/<domain>.conf`
  - `acme.sh --install-cronjob` installs a crontab job
    - `crontab -l` to confirm
  - `acme.sh --uninstall-cronjob` uninstalls a crontab job
  - `acme.sh --cron` simulates a crontab job run
    - it invokes `acme.sh --renew-all` which invokes `acme --renew` on each
      domain
    - renew performs `--issue` and `--install-cert`, using the args saved in
      `~/.acme.sh/<domain>_ecc/<domain>.conf`
- <https://github.com/acmesh-official/acme.sh/wiki/Using-systemd-units-instead-of-cron>
  - create `/etc/systemd/system/acme.sh.service`
    - `[Unit]`
    - `Description=Renew certificates using acme.sh`
    - `After=network-online.target nss-lookup.target`
    - `[Service]`
    - `Type=oneshot`
    - `SyslogIdentifier=acme.sh`
    - `ExecStart=/usr/bin/acme.sh --home /etc/acme.sh --cron`
  - create `/etc/systemd/system/acme.sh.timer`
    - `[Unit]`
    - `Description=Daily check for acme.sh`
    - `[Timer]`
    - `OnCalendar=daily`
    - `RandomizedDelaySec=1h`
    - `[Install]`
    - `WantedBy=timers.target`
  - `systemctl daemon-reload`
  - `systemctl enable --now acme.sh.timer`
