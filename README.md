# Stalwart-Setup

## Deploy NGINX

- Install required dependencies: 

```bash
sudo rpm-ostree install certbot nginx policycoreutils-python-utils
sudo reboot
```

- Run [setup.sh](https://github.com/Metropolis-nexus/NGINX-Setup) to get a simple standard NGINX setup.
- Generate a certificate for the mail server with certbot:

```bash
sudo certbot certonly \
    --webroot --webroot-path /srv/nginx \
    --no-eff-email \
    --key-type ecdsa \
    --reuse-key \
    --deploy-hook "nginx -s reload && systemctl restart stalwart" \
    -d mail.yourdomain.tld
```

- Add `/etc/nginx/conf.d/stalwart.conf`:

```
server {
    listen 443 quic;
    listen 443 ssl;
    listen [::]:443 quic;
    listen [::]:443 ssl;

    server_name mail.yourdomain.tld;

    ssl_certificate /etc/letsencrypt/live/mail.yourdomain.tld/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mail.yourdomain.tld/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/mail.yourdomain.tld/chain.pem;

    include snippets/security.conf;
    include snippets/cross-origin-security.conf;
    include snippets/quic.conf;
    include snippets/proxy.conf;
    include snippets/robots.conf;
    include snippets/universal_paths.conf;

    proxy_ssl_name mail.yourdomain.tld;

    location / {
        proxy_pass https://127.0.0.1:8443;
    }
}
```

## Deploy Stalwart

- Copy the appropriate Stalwart related files onto your system from [Quadlet-Files](https://github.com/Metropolis-nexus/Quadlet-Files).

**Note**: As of this writing (06/06/2026), the initial setup is a bit janky, but you need to follow it to make Stalwart read from the local configuratio properly. Modifying `/srv/stalwart/stalwart/etc/config.toml` from the get-go will not work.

- Change `PublishPort=127.0.0.1:8443:443` to `PublishPort=443:443` in `/etc/systemd/containers/stalwart.container`

- Start the Stalwart target and get your admin password:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now stalwart.target
sudo systemctl status stalwart
```

At this point, your `config.toml` should look like this:

![Config1](Config1.png)

This means that it isn't reading new configs in `config.toml` just yet.

- Log into the web UI with your password
- Hit Settings on the left sidebar
- Network -> Set the hostname -> Save & Reload
- System
    - Save & Reload without any change
    - Change `store.*` to `*`. Delete the of the local settings. Hit "Save & Reload" again
- Click on listeners. Then click on system again. You may see that the local settings get repopulated below `*`. Delete all of them and hit "Save & Reload" yet again.

Your `config.toml` should now look like this:

![Config2](Config2.png)

This means that it will start reading from/writing to `config.toml` properly.

- Add the following at the end of `/srv/stalwart/stalwart/etc/config.toml`:

```
certificate.LetsEncrypt.cert = "%{file:/etc/letsencrypt/live/mail.yourdomain.tld/fullchain.pem}%"
certificate.LetsEncrypt.default = true
certificate.LetsEncrypt.private-key = "%{file:/etc/letsencrypt/live/mail.yourdomain.tld/privkey.pem}%"
certificate.LetsEncrypt.subjects = "mail.yourdomain.tld"
```

- Restart Stalwart:

```bash
sudo systemctl restart stalwart
```

Stalwart should load with the correct certitificate:

![Stalwart Login](Stalwart-Login.png)

Change `PublishPort=443:443` back to `PublishPort=127.0.0.1:8443:443` in `/etc/systemd/containers/stalwart.container`.

```bash
sudo systemctl daemon-reload
sudo systemctl restart stalwart
sudo systemctl start nginx
```