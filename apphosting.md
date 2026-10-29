# Golang app hosting using nginx and systemd service

## nginx virtual hosting for workflow.lxroot.com
Assuming your app is listening to port `8100`

> `nano /etc/nginx/sites-available/workflow.lxroot.com`
```
server {
    listen 80;
    #listen [::]:80;

    server_name workflow.lxroot.com;
    client_max_body_size 25M;

    location / {
        proxy_pass http://127.0.0.1:8100;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```
> `ln -sf /etc/nginx/sites-available/workflow.lxroot.com /etc/nginx/conf.d/workflow.lxroot.com.conf`

## nginx configuration reload or restart
```
nginx -t
nginx -s reload | systemctl restart nginx
```

## Systemd service
> `nano /etc/systemd/system/workflow.lxroot.com.service`
```
[Unit]
Description=workflow.lxroot.com
After=network.target
[Service]
Type=simple
User=workflowlxrootcom
Group=mateors
Restart=on-failure
RestartSec=5s
WorkingDirectory=/var/www/vhosts/workflow.lxroot.com/public_html
#ExecStartPre=/usr/local/go/bin/go mod tidy
#ExecStartPre=/usr/local/go/bin/go build -o workflow.lxroot.com
ExecStart=/var/www/vhosts/workflow.lxroot.com/public_html/workflow.lxroot.com
[Install]
WantedBy=multi-user.target
```
### systemd reload
> `systemctl daemon-reload`

## Check your application status or start|stop|restart
```shell
systemctl start workflow.lxroot.com
systemctl stop workflow.lxroot.com
systemctl restart workflow.lxroot.com
systemctl status workflow.lxroot.com
```

> If everything alright now you should see the url https://workflow.lxroot.com (https because of cloudflare dns proxy used)
