# Reverse proxy with Nginx

#### Table of contents

* [1. Basic server setup](#1-basic-server-setup)
    * [1. Configure firewall](#1-configure-firewall)
* [2. Enable TLS through Let's Encrypt](#2-enable-tls-through-lets-encrypt)
    * [1. Install Certbot and dependencies](#1-install-certbot-and-dependencies)
    * [2. Get certificates](#2-get-certificates)
    * [3. Create Crontab job for certificate renewal](#3-create-crontab-job-for-certificate-renewal)
* [3. Nginx reverse proxy setup](#3-nginx-reverse-proxy-setup)
    * [1. Install Nginx](#1-install-nginx)
    * [2. Configure proxy](#2-configure-proxy)

## 1. Basic server setup

It is recommended that you follow this [guide](/ubuntu/initial-server-setup/README.md) on how to setup the a basic Ubuntu server.

#### 1. Configure firewall

* We need to make sure that the firewall allows SSH and HTTP/HTTPS (Nginx) connections. We can allow these connections by typing:
```shell script
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
```

* Reload UFW:
```shell script
sudo ufw reload
```

* If UFW isn’t enabled, enable it:
```shell script
sudo ufw enable
```

* Check the status:
```shell script
sudo ufw status
```

* You should see:
```shell script
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx Full                 ALLOW       Anywhere                 
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

For more information, see [this](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) guide.

[&#8593; Back to the top](#table-of-contents)

## 2. Enable TLS through Let's Encrypt

Matrix Synapse now requires TLS enabled by default to allow the server to be used to securely. The easiest way to configure TLS is to obtain SSL certificates from a trusted Certificate Authority such as Let’s Encrypt.

#### 1. Install Certbot and dependencies

* Run the following commands that installs the common software properties, adds the Certbot repository and finally installs Certbot as well as the `--nginx` plugin:
```shell script
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot python-certbot-nginx
```

#### 2. Get certificates

* Run the following command to create certificates with Certbot, replace `example.com` with your domain/subdomain:
```shell script
sudo certbot certonly --nginx -d example.com
```

* The command starts an interactive configuration script that asks a couple of questions to help with managing certificates.
    * On the first installation on any specific host, you’ll need to enter a contact email.
    * Then go through the Let’s Encrypt Terms of Service and select Agree if you accept the terms and wish to use the service.
    * Choose whether you wish to share your email address with the Electronic Frontier Foundation (EFF) for updates on their work.
  
* If the client was successful at obtaining a certificate you can find a confirmation and certificate expiration date at the end of the client output.

#### 3. Create Crontab job for certificate renewal

The previously created certificate is only valid for 3 months, therefore, we need to create a daily Crontab job to check if the certificate can be renewed and if so, use Certbot to renew the certifcate.

* Firstly, create a new script:
```shell script
sudo nano /etc/cron.daily/letsencrypt-renew
``` 

* Use the following script:
```shell script
#!/bin/sh
if certbot renew > /var/log/letsencrypt/renew.log 2>&1 ; then
   nginx -s reload
fi

exit
```

* Set the file to executable:
```shell script
sudo chmod +x /etc/cron.daily/letsencrypt-renew
```

* Run the Crontab using:
```shell script
sudo crontab -e
```

* Add this line to the Crontab:
```
01 02,14 * * * /etc/cron.daily/letsencrypt-renew
```

* Let’s Encrypt recommends setting the automated renewal script to run twice a day on a random minute within the hour. The above example runs on 02:01 and 14:01 but you can select any time slot you wish.

[&#8593; Back to the top](#table-of-contents)

## 3. Nginx reverse proxy setup

#### 1. Install Nginx

* Install Nginx:
```shell script
sudo apt-get install nginx
```

#### 2. Configure proxy

* Create a configuration file:
```shell script
sudo nano /etc/nginx/conf.d/example.com.conf
```

* Then enter the following to enable the proxy with SSL termination. Replace the matrix.example.com with your domain in the server name:
```
server {
    listen 80;
	listen [::]:80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

* This configuration listens for secure connections as ports 443 for HTTPS and uses SSL termination to decrypt the SSL requests at Nginx and sends them unencrypted to the server running locally (in the above example the server running on: `127.0.0.1:1337`). Connections to the insecure HTTP port 80 are redirected to use HTTPS instead.

* Start Nginx and enable it to start when the server starts:
```shell script
sudo systemctl start nginx
sudo systemctl enable nginx
```

[&#8593; Back to the top](#table-of-contents)
