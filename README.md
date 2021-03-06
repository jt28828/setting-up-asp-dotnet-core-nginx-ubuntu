### Setting up Nginx with Dotnet core.

Things I haven't covered (TODO):

1. Forwarding headers

### Dotnet

Most of my steps assume that you've had no problems (I didn't...) but if you did have problems, there are links at the bottom, that I got these commands from.

This gets you the microsoft packages list
```
wget -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```
Now we can install dotnet
```
sudo apt-get install apt-transport-https
sudo apt-get update
sudo apt-get install dotnet-sdk-2.1
```
At this point, you can run the application
```
dotnet /path/to/dll/applicationName.dll
curl localhost:5000/api/AValidGetRoute
```

### Nginx 

Now we can install Nginx as a reverse proxy
```
sudo apt-get install nginx
sudo service nginx start
```
Verify it's running
```
curl http://localhost/index.nginx-debian.html
```
Now we edit Nginx's settings
```
sudo nano /etc/nginx/sites-available/default
```
Use CMD + K to delete all lines (one by one)
Replace text with this below. Change the IP (required), and ports if needed.
```
server {
    listen        80;
    server_name   IP/DOMAIN NAME;
    location / {
        proxy_pass         http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```
Test if the Nginx setup is valid
```
sudo nginx -t
```
Reload Nginx
```
sudo nginx -s reload
```
Now you can run the dotnet app again
```
dotnet /path/to/dll/applicationName.dll
```
This time, just try go through the port you specified instead of 5000/5001.

Note that you can omit the :80 in the next statement
```
curl localhost:80/api/AValidGetRoute
```

### Kestrel

Next we just need to set the dotnet app up as a kestrel service.

Create the service definition file (replace APPNAME)
```
sudo nano /etc/systemd/system/kestrel-APPNAME.service
```
This is an example body for kestrel-APPNAME.service
```
[Unit]
Description=Example .NET Web API App running on Ubuntu

[Service]
WorkingDirectory=/var/www/APPNAME
ExecStart=/usr/bin/dotnet /var/www/APPNAME/APPNAME.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=dotnet-APPNAME
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target

```
Now we can enable the service, start it, and check it's status
```
sudo systemctl enable kestrel-APPNAME.service
sudo systemctl start kestrel-APPNAME.service
sudo systemctl status kestrel-APPNAME.service
```
Some useful commands for the kestrel service:
```
sudo systemctl restart kestrel-APPNAME.service
sudo systemctl stop kestrel-APPNAME.service
sudo journalctl -fu kestrel-APPNAME.service
```

### Certbot/SSL/HTTPS

This is optional, and is getting a SSL Certificate for HTTPS requests.

```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

Running this next command will come with a bunch of questions the first time
```
sudo certbot --nginx -d domain.com
```
If needed, some helpful certbot commands:

Now we view Nginx's settings again.

Ensure that the localhost:5000 is still http (it's http locally, but https from client to Nginx) and that it's listening on port 443
```
sudo nano /etc/nginx/sites-available/default
```

Delete will prompt with which certificate to delete
```
sudo certbot certificates
sudo certbot delete
```
You can renew certificates using this (make a task to run this every 90ish days)
```
sudo certbot renew
```

Sources:

https://dotnet.microsoft.com/download/linux-package-manager/ubuntu18-04/sdk-current

https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-2.1

https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx