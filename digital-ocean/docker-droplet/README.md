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
```bash
$  ssh root@droplet_ip
```
* Add a new user:
```bash
$  adduser kieranroneill
```
* Use a strong password and skip the rest of the questions.
* Give the new user root privileges and add them to the `docker` group to access Docker:
```bash
$  usermod -aG sudo,docker kieranroneill
```
#### 2. Add SSH public key authentication 

* On your **local machine**, generate an ssh key:
```bash
$  ssh-keygen
```
* Using the default ssh key selection, you can output the public key using:
```bash
$  cat ~/.ssh/id_rsa.pub
```
* Copy this key.
* Back on the server, switch to the new user:
```bash
$  su - kieranroneill
```
* Create a new `.ssh` directory and change the user permissions:
```bash
$  mkdir ~/.ssh
$  chmod 700 ~/.ssh
```
* Create and open a new file:
```bash
$  nano ~/.ssh/authorized_keys
```
* Now insert your public key (which should be in your clipboard) by pasting it into the editor.
* Now restrict the permissions of the `authorized_keys` file with this command:
```bash
$  chmod 600 ~/.ssh/authorized_keys
```

#### 3. Disable password authentication

* As your new sudo user, open the SSH daemon configuration:
```bash
$  sudo nano /etc/ssh/sshd_config
```
* Ensure the following settings are set:
```bash
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```
* Reload the SSH daemon:
```bash
$  sudo systemctl reload sshd
```

#### 4. Setup basic firewall for SSH

* We need to make sure that the firewall allows SSH connections so that we can log back in next time. We can allow these connections by typing:
```bash
$  sudo ufw allow OpenSSH
```
* Now, enable the firewall:
```bash
$  sudo ufw enable
```

For more information, see [this](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) guide.

#### 5. Create `swapfile` (optional)

Creating a `swapfile` allows you to use disk space to as overflow if you want to extend the RAM of the machine (Useful for one-off commands that are memory intensive, but you don't want to have a more expensive droplet setup).

* Login as the root user:
```bash
$  kieranroneill@droplet_ip
```
* Create a 1Gb swapfile:
```bash
$  sudo fallocate -l 1G /swapfile
```
* Set the privileges to root:
```bash
$  sudo chmod 600 /swapfile
```
* Now mark the file as swap space by typing:
```bash
$  sudo mkswap /swapfile
```
* Now we can enable the swap file and start utilising the disk space for memory:
```bash
$  sudo swapon /swapfile
```
* Finally, add the swap settings to the `/etc/fstab` file to ensure the settings persist between server reboots:
```bash
$  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

* For more information, see [this](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04) guide.

#### 6. Add Git deployment keys (optional)

* Login to the server:
```bash
$  kieranroneill@droplet_ip
```
* Create a git ssh key:
```bash
$  ssh-keygen -t rsa
```
* Leave the passphrase blank and copy the key from the output of the public key:
```bash
$  cat ~/.ssh/id_rsa.pub
```
* Login to Github, under **Settings**>**Deploy keys** add the copied public key and give it a meaningful title, e.g. kieranroneill@droplet_ip.

#### 7. Add CircleCI deployment keys (optional)

* Login to the server:
```bash
$  kieranroneill@droplet_ip
```
* Create a deploy ssh key:
```bash
$  ssh-keygen -t rsa
```
* Leave the passphrase blank, name it `/home/loarg/.ssh/id_rsa_circleci` and copy the key from the output of the private key:
```bash
$  cat ~/.ssh/id_rsa_circleci
```
* Add the key to the `authorized_keys` file to avoid the public key being denied:
```bash
$  cat ~/.ssh/id_rsa_circleci.pub >> ~/.ssh/authorized_keys
```
* Login to CircleCI, under **Project settings**>**SSH Permissions** and add a new key, copying the private key from earlier. Use the server IP address as the host.
