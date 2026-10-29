## How do you install latest nginx on your ubuntu 26.04 LTS

> sudo apt update

> sudo apt install -y unattended-upgrades

> sudo apt install -y curl gnupg2 ca-certificates lsb-release ubuntu-keyring

``curl
curl -fsSL https://nginx.org/keys/nginx_signing.key \
    | sudo gpg --dearmor --yes \
        -o /usr/share/keyrings/nginx-archive-keyring.gpg
```

```
gpg --dry-run --quiet --no-keyring \
    --import --import-options import-show \
    /usr/share/keyrings/nginx-archive-keyring.gpg
```

```
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/ubuntu noble nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

```
printf 'Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n' \
    | sudo tee /etc/apt/preferences.d/99nginx
```

> `sudo apt install -y nginx`

> `nginx -v`

> `nginx -V`

sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx

sudo systemctl status nginx --no-pager
sudo nginx -t
