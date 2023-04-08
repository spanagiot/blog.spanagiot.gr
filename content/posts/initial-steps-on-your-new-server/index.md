+++
categories = ["beginner", "cloud", "tutorial", "server", "ubuntu"]
date = 2017-01-27T20:11:00Z
description = ""
draft = false
slug = "initial-steps-on-your-new-server"
tags = ["beginner", "cloud", "tutorial", "server", "ubuntu"]
title = "Initial steps on your new server"

+++


It was not long ago when I received a message from my friend DaKnOb. He told me "Hey, I found that the domain spanagiot.gr is free. Why don't you register it and set up your own server?". In no time, I got the domain and a server running the latest Ubuntu (16.04 at this point). Here is what I learned in my short journey:

### First three commands

Let's say that your server has been up and running and it's waiting for you to connect and deploy all these amazing services you have thought of. Usually, the infrastructure provider recommends you to specify a public key to use to login in your machine. After you copy and paste it in the website(you can find it here: $HOME/.ssh/id_rsa.pub) SSH to your machine (usually using: ```ssh root@SERVER-IP```)
Once you are in, you should really do this
```
apt-get update
apt-get upgrade
reboot 
```
*Most of the times, your installation might be on the latest major version but there are many security updates and patches that your cloud provider didn't add yet to the installation image. With this step we make sure that the system is up-to-date and will not be affected by recent vulnerabilities.*

### Secure our root user
After our system finishes the reboot, our next step is to change the root password. The reason is that we don't know what the original root password is, and even worse, we don't know who might know what the root password is ;). The command to change a user's password in Linux systems is ```passwd USERNAME```.
So we execute ```passwd root``` and we get a prompt to enter the new password. *Personally, I use 1Password (I find it very convenient) so I generated a strong password and saved it in my vault.*

### Create a new user
Our next step is to add a new user which we will use as the main account on this machine. The command to do this is ```useradd -m USERNAME```.
The -m option creates automatically the home directory of the user (the place that our user stores all his documents, his photos, many of his settings etc.). We have our new user but there are not a lot of things that he can do. But we need this account to be able to manage the server. In Linux systems there is the command sudo which allows you to execute commands as an other user without the hassling of logging in and out interrupting your session. If no user is specified it will execute our command after it as the root user. *As you can understand, it's a powerful command and so it needs special permissions for a user to use it. Our user must be in the sudo group and we can add him with* ```usermod -a -G sudo USERNAME```. In order to enable passwordless sudo run `visudo`, find the line that starts with `%sudo` and edit it to this `%sudo ALL=(ALL) NOPASSWD: ALL`

### SSH keys for the new user
Our next step is to add the public key of the machine we used to ssh to the user so we can connect without using a password. This can be done by copying our public key (which can be found here ```$HOME/.ssh/id_rsa.pub```) and pasting it in /home/USERNAME/.ssh/authorized_keys. Now, we will try to connect to our new user to verify that everything works as intended(ssh USERNAME@SERVER-IP). If we managed to login successfully, we must also see if our user can execute commands as root. To test this, all we have to do is to write ```sudo echo 1``` and press enter. If the output is 1 everything is working fine. *So, to further secure the machine, I recommend to disable every authorized key from the root account.* This can be done by deleting the file authorized_keys (rm /root/.ssh/authorized_keys). To check if it is working properly, try to ssh in the root account. It shouldn't allow you to do so.

### Backup keys for the new user
Another step to ensure that we will not stay out of our machine is to create a backup pair of keys. In your machine, execute this ```ssh-keygen -t rsa``` and follow the on-screen instructions. If you want you can add a passphrase to your key.
```
Enter passphrase (empty for no passphrase):
```
Entering a passphrase means that when you want to use this set of keys you have to know and type this passphrase. The whole procedure is generally straightforward. When the pair is generated, you can make the server use this key by executing in your machine ```ssh-copy-id -i PATH_OF_PUBLIC_KEY USERNAME@SERVER-IP```.
Again, try to login using this key set

```
ssh -i PATH_OF_NEW_PRIVATE_KEY USERNAME@SERVER-IP
```

If you can login, your new pair of keys is successfully installed in your server. *Because I use these keys as backup, I copy them to 1Password as Secure Notes, so they are encrypted and available from everywhere in case I don't carry my computer with me, and delete them from my machine.*

### Manage security updates automatically
The last step in a newly created machine before doing anything else is to install unattended-upgrades. ```sudo apt-get install unattended-upgrades```
This package keeps your server current with the latest security updates automatically. The default configuration will work fine but in case you want to tune some settings you can learn more about it in Debian's page.

*At this point, we finished a very basic setup of a server. Keep in mind that these steps cover only initial steps of securing our machine but provide us a stable foundation where we can start our online presence.*

