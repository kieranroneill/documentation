# Docker droplet setup

#### Table of contents

* [Basic server setup](#basic-server-setup)
    * [1. Create new user](#1-create-new-user)
    * [2. Add SSH public key authentication](#2-add-ssh-public-key-authentication)
    * [3. Disable password authentication](#3-disable-password-authentication)
    * [4. Setup basic firewall for SSH](#4-setup-basic-firewall-for-ssh)
    * [5. Create `swapfile` (optional)](#5-create-swapfile-optional)
    * [6. Add Git deployment keys (optional)](#6-add-git-deployment-keys-optional)
    * [7. Add CircleCI deployment keys (optional)](#7-add-circleci-deployment-keys-optional)

## Basic server setup

#### 1. Create new user

* Login to Digital Ocean and create a new droplet; use the one-click app and choose the Docker droplet and make sure you add your SSH key.
* When it has built, login to the droplet:
```shell script
ssh root@droplet_ip
```
* Add a new user:
```shell script
adduser kieranroneill
```
* Use a strong password and skip the rest of the questions.
* Give the new user root privileges and add them to the `docker` group to access Docker:
```shell script
usermod -aG sudo,docker kieranroneill
```
#### 2. Add SSH public key authentication 

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
su - kieranroneill
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

#### 3. Disable password authentication

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

#### 4. Setup basic firewall for SSH

* We need to make sure that the firewall allows SSH connections so that we can log back in next time. We can allow these connections by typing:
```shell script
sudo ufw allow OpenSSH
```
* Now, enable the firewall:
```shell script
sudo ufw enable
```

For more information, see [this](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) guide.

#### 5. Create `swapfile` (optional)

Creating a `swapfile` allows you to use disk space to as overflow if you want to extend the RAM of the machine (Useful for one-off commands that are memory intensive, but you don't want to have a more expensive droplet setup).

* Login as the root user:
```shell script
kieranroneill@droplet_ip
```
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

#### 6. Add Git deployment keys (optional)

* Login to the server:
```shell script
kieranroneill@droplet_ip
```
* Create a git ssh key:
```shell script
ssh-keygen -t rsa
```
* Leave the passphrase blank and copy the key from the output of the public key:
```shell script
cat ~/.ssh/id_rsa.pub
```
* Login to Github, under **Settings**>**Deploy keys** add the copied public key and give it a meaningful title, e.g. kieranroneill@droplet_ip.

#### 7. Add CircleCI deployment keys (optional)

* Login to the server:
```shell script
kieranroneill@droplet_ip
```
* Create a deploy ssh key:
```shell script
ssh-keygen -t rsa
```
* Leave the passphrase blank, name it `/home/loarg/.ssh/id_rsa_circleci` and copy the key from the output of the private key:
```shell script
cat ~/.ssh/id_rsa_circleci
```
* Add the key to the `authorized_keys` file to avoid the public key being denied:
```shell script
cat ~/.ssh/id_rsa_circleci.pub >> ~/.ssh/authorized_keys
```
* Login to CircleCI, under **Project settings**>**SSH Permissions** and add a new key, copying the private key from earlier. Use the server IP address as the host.
