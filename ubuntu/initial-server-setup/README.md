# Initial server setup

#### Table of contents

* [1. Create new user](#1-create-new-user)
* [2. Add SSH public key authentication](#2-add-ssh-public-key-authentication)
* [3. Disable password authentication](#3-disable-password-authentication)
* [4. Setup basic firewall for SSH](#4-setup-basic-firewall-for-ssh)
* [5. Create `swapfile` (optional)](#5-create-swapfile-optional)
* [6. Add Git deployment keys (optional)](#6-add-git-deployment-keys-optional)
* [7. Add CircleCI deployment keys (optional)](#7-add-circleci-deployment-keys-optional)

## 1. Create new user

* Assuming you have SSH access to your server, login using:
```shell script
ssh root@server_ip
```

* Add a new user:
```shell script
adduser ubuntu
```

* Use a strong password and skip the rest of the questions.

* Give the new user root privileges:
```shell script
usermod -aG sudo ubuntu
```

## 2. Add SSH public key authentication 

* On your **local machine**, generate an ssh key:
```shell script
ssh-keygen
```

* Using the default ssh key selection, you can output the public key using:
```shell script
cat ~/.ssh/id_rsa.pub
```

* Copy this key.

* Back on the server, switch to the new user:
```shell script
su - matrix
```

* Create a new `.ssh` directory and change the user permissions:
```shell script
mkdir ~/.ssh
chmod 700 ~/.ssh
```

* Create and open a new file:
```shell script
nano ~/.ssh/authorized_keys
```

* Now insert your public key (which should be in your clipboard) by pasting it into the editor.

* Now restrict the permissions of the `authorized_keys` file with this command:
```shell script
chmod 600 ~/.ssh/authorized_keys
```

## 3. Disable password authentication

* As your new sudo user, open the SSH daemon configuration:
```shell script
sudo nano /etc/ssh/sshd_config
```

* Ensure the following settings are set:
```shell script
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

* Reload the SSH daemon:
```shell script
sudo systemctl reload sshd
```

## 4. Setup basic firewall for SSH

* We need to make sure that the firewall allows SSH, HTTP/HTTPS and Matrix-Synapse connections. We can allow these connections by typing:
```shell script
sudo ufw allow OpenSSH
```

* Now, enable the firewall:
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
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

## 5. Create `swapfile` (optional)

Creating a `swapfile` allows you to use disk space to as overflow if you want to extend the RAM of the machine (Useful for one-off commands that are memory intensive, but you don't want to have a more expensive droplet setup).

* Create a 1Gb swapfile:
```shell script
sudo fallocate -l 1G /swapfile
```

* Set the privileges to root:
```shell script
sudo chmod 600 /swapfile
```

* Now mark the file as swap space by typing:
```shell script
sudo mkswap /swapfile
```

* Now we can enable the swap file and start utilising the disk space for memory:
```shell script
sudo swapon /swapfile
```

* Finally, add the swap settings to the `/etc/fstab` file to ensure the settings persist between server reboots:
```shell script
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

* For more information, see [this](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04) guide.

## 6. Add Git deployment keys (optional)

* Create a git ssh key:
```shell script
ssh-keygen -t rsa
```

* Leave the passphrase blank, name it `/home/ubuntu/.ssh/id_rsa_github` and copy the key from the output of the private key:
```shell script
cat ~/.ssh/id_rsa_github
```

* Add the key to the `authorized_keys` file to avoid the public key being denied:
```shell script
cat ~/.ssh/id_rsa_github.pub >> ~/.ssh/authorized_keys
```

* Login to Github, under **Settings**>**Deploy keys** add the copied public key and give it a meaningful title, e.g. `ubuntu@server_ip`.

## 7. Add CircleCI deployment keys (optional)

* Create a deploy ssh key:
```shell script
ssh-keygen -t rsa
```

* Leave the passphrase blank, name it `/home/ubuntu/.ssh/id_rsa_circleci` and copy the key from the output of the private key:
```shell script
cat ~/.ssh/id_rsa_circleci
```

* Add the key to the `authorized_keys` file to avoid the public key being denied:
```shell script
cat ~/.ssh/id_rsa_circleci.pub >> ~/.ssh/authorized_keys
```

* Login to CircleCI, under **Project settings**>**SSH Permissions** and add a new key, copying the private key from earlier. Use the server IP address as the host.
