# Setup Matrix-Synapse server

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
* [4. Setup Matrix-Synapse](#4-setup-matrix-synapse)
    * [1. Install Matrix-Synapse](#1-install-matrix-synapse)
    * [2. Configure Matrix-Synapse](#2-configure-matrix-synapse)
    * [3. Create an admin user](#3-create-an-admin-user)
    * [4. Securing Federation with SSL](#4-securing-federation-with-ssl)

## 1. Basic server setup

It is recommended that you follow this [guide](/ubuntu/initial-server-setup/README.md) on how to setup the a basic Ubuntu server.

#### 1. Configure firewall

* We need to make sure that the firewall allows SSH, HTTP/HTTPS (Nginx) and Matrix-Synapse connections. We can allow these connections by typing:
```shell script
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
sudo ufw allow 8448
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
8448                       ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx Full (v6)            ALLOW       Anywhere (v6)             
8448 (v6)                  ALLOW       Anywhere (v6)
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

* Run the following command to create certificates with Certbot, replace `matrix.example.com` with your domain/subdomain:
```shell script
sudo certbot certonly --nginx -d matrix.example.com
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
sudo nano /etc/nginx/conf.d/matrix.conf
```

* Then enter the following to enable the proxy with SSL termination. Replace the matrix.example.com with your domain in the server name:
```
server {
    listen 80;
	listen [::]:80;
    server_name matrix.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name matrix.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}

server {
    listen 8448 ssl default_server;
    listen [::]:8448 ssl default_server;
    server_name matrix.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;
    location / {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

* This configuration listens for secure connections as ports 443 for HTTPS and 8448 for Matrix federation server-to-server communication. Connections to the insecure HTTP port 80 are redirected to use HTTPS instead.

* Start Nginx and enable it to start when the server starts:
```shell script
sudo systemctl start nginx
sudo systemctl enable nginx
```

[&#8593; Back to the top](#table-of-contents)

## 4. Setup Matrix-Synapse

#### 1. Install Matrix-Synapse

* Update the local package index and add the official Matrix repository to APT:
```shell script
sudo apt-get update
sudo add-apt-repository https://matrix.org/packages/debian/
```

* Add the repository key. This will check to make sure any installations and updates have been signed by the developers and stop any unauthorized packages from being installed on your server:
```shell script
wget -qO - https://matrix.org/packages/debian/repo-key.asc | sudo apt-key add -
```

* Update the local package index and install the Matrix-Synapse server:
```shell script
sudo apt-get update
sudo apt-get install matrix-synapse
```

* During the installation, you will be prompted to enter a server name, which should be your domain name, ie. `matrix.example.com`.

* Start Matrix-Synapse and enable it to start when the server starts:
```shell script
sudo systemctl start matrix-synapse
sudo systemctl enable matrix-synapse
```

#### 2. Configure Matrix-Synapse

* Use the following command to generate a 64-character string:
```shell script
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1
```

* Copy the string you create, then open the Synapse configuration file:
```shell script
sudo nano /etc/matrix-synapse/homeserver.yaml
```

* Use the previously created 64-character string and add it to the `registration_shared_secret`:
```yaml
registration_shared_secret: "randomly_generated_string"
```

* Set the `enable_registration` to `false` to stop public registrations:
```yaml
# Enable registration for new users.
enable_registration: False
```

* Look for the `listeners` section and **ONLY** bind to `localhost` as well as ensuring the forward flag is set to `true`:
```yaml
- port: 8008
  tls: false
  bind_addresses: ['127.0.0.1']
  type: http
  x_forwarded: true
```

* Exit the editor and restart the service:
```shell script
sudo systemctl restart matrix-synapse
```

#### 3. Create an admin user

* Use this to create a new user. The -c flag specifies the configuration file, and uses the local Synapse instance which is listening on port `8448`:
```shell script
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml https://127.0.0.1:8448
```

* You will be prompted to choose a username and a password. You’ll also be asked to make the user an administrator, say `yes`.

[&#8593; Back to the top](#table-of-contents)

#### 4. Securing Federation with SSL

Now that Synapse is configured and can communicate with other homeservers. By default Synapse uses self signed certificates, you can increase its security by using the same SSL certificates you requested from Let’s Encrypt.

* Simply copy the certificates to your Synapse directory:
```shell script
    sudo cp /etc/letsencrypt/live/matrix.example.com/fullchain.pem /etc/matrix-synapse/fullchain.pem
    sudo cp /etc/letsencrypt/live/matrix.example.com/privkey.pem /etc/matrix-synapse/privkey.pem
```

* In order for these certificates to be updated when they are renewed you need to add these commands to your cron tab:
```shell script
sudo crontab -e
```

* Add the following lines to periodically copy the new certificates and restart the matrix-synapse service:
```
35 2 * * 1 sudo cp /etc/letsencrypt/live/example.com/fullchain.pem /etc/matrix-synapse/fullchain.pem
35 2 * * 1 sudo cp /etc/letsencrypt/live/example.com/privkey.pem /etc/matrix-synapse/privkey.pem
36 2 * * 1 sudo systemctl restart matrix-synapse
```

* Save and pen up the configuration file:
```shell script
sudo nano /etc/matrix-synapse/homeserver.yaml
```

* Using the same certificate you requested from Lets Encrypt, replace the paths in the configuration file:
```yaml
tls_certificate_path: "/etc/matrix-synapse/fullchain.pem"

tls_private_key_path: "/etc/matrix-synapse/privkey.pem"
```

* Finally, restart the service:
```shell script
sudo systemctl restart matrix-synapse
```
