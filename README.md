# Linux Server Setup & MERN App Deployment

These are the steps to setup an Ubuntu server from scratch and deploy a MERN app with the PM2 process manager and Nginx.
I`m using OVH, but you could just as well use a different cloud provider or your own machine or VM.
I tried to combined different guides in one place for personal use and as reminder.
Original sources are gathered around the Internet from:

- OVH guides
- DigitalOcean guides
- bradtraversy
- etc

### First Step

Connecting to your VPS
Connecting from a command terminal, type the following to log in to your VPS with the information provided in the email (username and IPv4 address):

```bash
ssh username@IPv4_of_your_VPS
```

Updating and upgrading your packages

```
sudo apt update
sudo apt upgrade
```

### Second Step

Since you are now logged in with elevated privileges (a sudo user), you can enter commands to perform administrative tasks. It is recommendable to first change your password:

```bash
~$ sudo passwd username
New password:
Retype new password:
passwd: password updated successfully
```

Note that passwords are not displayed. Next, switch to the “root” user and set your admin password:

```bash
~$ sudo su -
~# passwd
New password:
Retype new password:
passwd: password updated successfully
```

Changing the default SSH listening port
One of the first things to do on your server is configuring the SSH service’s listening port. It is set to port 22 by default, therefore server hacking attempts by robots will target this port. Modifying this setting by using a different port is a simple measure to harden your server against automated attacks.

To do this, modify the service configuration file with a text editor of your choice (nano used in this example):

```bash
~$ sudo nano /etc/ssh/sshd_config
```

You should find the following or similar lines:

```bash
# What ports, IPs and protocols we listen for
Port 22
```

Replace the number 22 with the port number of your choice. Please do not enter a port number already used on your system. To be safe, use a number between 49152 and 65535.
Save and exit the configuration file.

Restart the service:

```bash
sudo systemctl restart sshd
```

This should be sufficient to apply the changes. Alternatively, reboot the VPS `(~$ sudo reboot)`.

Remember that you will have to indicate the new port any time you request an SSH connection to your server, for example:

```bash
ssh username@IPv4_of_your_VPS -p NewPortNumber
```

### Third Step

Time to start setting up environment.

## Installing ZSH

```bash
apt install zsh
```

Installing oh-my-zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Install Node.js

to install Node.js with curl using the following commands

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Check to see if node was installed
node --version
npm --version
```

## Setup Git

For Git to work properly, we need to let it know who we are so that it can link a local Git user (you) to GitHub. When working on a team, this allows people to see what you have committed and who committed each line of code.

The commands below will configure Git. Be sure to enter your own information inside the quotes (but include the quotation marks)!

```bash
git config --global user.name "Your Name"
git config --global user.email "yourname@example.com"
git config --global init.defaultBranch main
git config --global color.ui auto
```

## Create an SSH Key for accessing git repository

```bash
ssh-keygen -t ed25519 -C <youremail>
```

When it prompts you for a location to save the generated key, just push Enter.
Next, it will ask you for a password; enter one if you wish, but it’s not required.

## Link Your SSH Key with GitHub

To display your SSh key enter:

```bash
cat ~/.ssh/id_ed25519.pub
```

Highlight and copy the output, which starts with ssh-ed25519 and ends with your email address.
You can add this key for specific repo.

## Install Nginx

Because Nginx is available in Ubuntu’s default repositories, it is possible to install it from these repositories using the apt packaging system.

```bash
sudo apt update
sudo apt install nginx

```

## Adjusting the Firewall

!!!Warning be carefull with this step!!!
When you activate firewall DON`T forget to add your PORT selected in Step 2. If you skip this you will loose your access to server!!!

Before testing Nginx, the firewall software needs to be adjusted to allow access to the service. Nginx registers itself as a service with ufw upon installation, making it straightforward to allow Nginx access.

List the application configurations that ufw knows how to work with by typing:

```bash
sudo ufw app list
```

It is recommended that you enable the most restrictive profile that will still allow the traffic you’ve configured. Right now, we will only need to allow traffic on port 80.

You can enable this by typing:

```bash
sudo ufw allow 'Nginx HTTP'

```

To check status:

```bash
sudo ufw status
```

## Managing the Nginx Process

```bash
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
```

## Setting Up Server Blocks (Recommended)

For example if you trying to host client site, admin site and backend for both use something like: admin.your_domain, server.your_domain or your_domain for main site

When using the Nginx web server, server blocks (similar to virtual hosts in Apache) can be used to encapsulate configuration details and host more than one domain from a single server. We will set up a domain called your_domain, but you should replace this with your own domain name.

Nginx on Ubuntu 20.04 has one server block enabled by default that is configured to serve documents out of a directory at /var/www/html. While this works well for a single site, it can become unwieldy if you are hosting multiple sites. Instead of modifying /var/www/html, let’s create a directory structure within /var/www for our your_domain site, leaving /var/www/html in place as the default directory to be served if a client request doesn’t match any other sites.

Create the directory for your_domain as follows, using the -p flag to create any necessary parent directories:

```bash
sudo mkdir -p /var/www/your_domain/html

sudo chmod -R 755 /var/www/your_domain
```

Next, create a sample index.html page using nano or your favorite editor:

```bash
sudo nano /var/www/your_domain/html/index.html
```

Inside, add the following sample HTML:

```bash
<html>
    <head>
        <title>Welcome to your_domain!</title>
    </head>
    <body>
        <h1>Success!  The your_domain server block is working!</h1>
    </body>
</html>
```

In order for Nginx to serve this content, it’s necessary to create a server block with the correct directives. Instead of modifying the default configuration file directly, let’s make a new one at /etc/nginx/sites-available/your_domain:

```bash
sudo nano /etc/nginx/sites-available/your_domain

```

Paste in the following configuration block, which is similar to the default, but updated for our new directory and domain name:

```bash
server {
        listen 80;
        listen [::]:80;

        root /var/www/your_domain/html;
        index index.html index.htm index.nginx-debian.html;

        server_name your_domain www.your_domain;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

Notice that we’ve updated the root configuration to our new directory, and the server_name to our domain name.
Next, let’s enable the file by creating a link from it to the sites-enabled directory, which Nginx reads from during startup:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

To avoid a possible hash bucket memory problem that can arise from adding additional server names, it is necessary to adjust a single value in the /etc/nginx/nginx.conf file. Open the file:

```bash
sudo nano /etc/nginx/nginx.conf
```

Find the server_names_hash_bucket_size directive and remove the # symbol to uncomment the line.

Next, test to make sure that there are no syntax errors in any of your Nginx files:

```bash
sudo nginx -t
```

If there aren’t any problems, restart Nginx to enable your changes:

```bash
sudo systemctl restart nginx
```

### Forth Step

Get files on the server
On your SERVER, go to where you want the app to live and clone the repo you want to deply from GitHub (or where ever else)

I prefer to create directory called sites:

```bash
mkdir sites
```

After you created directory navigate there:

```bash
cd sites
```

Clone your repo here with:

```bash
git clone your_repo.git
```

### Fifth Step

App setup
Navigate to your folder with project and install all dep with

```bash
npm install or yarn
```

Don`t forget to create your .env files and populate them:

```bash
sudo nano .env
```

After installing all dependencies build your frontend project and copy it to the corresponding folder. You need to be in the project folder to copy from it:

```bash
sudo cp -R ./build /var/www/your_domain/html
```

## PM2 Setup

PM2 is a production process manager fro Node.js. It allows us to keep Node apps running without having to have terminal open with npm start, etc like we do for development.

Let's first install PM2 globally with NPM

```bash
sudo npm install -g pm2
```

Run with PM2

```bash
pm2 start server.js # or whatever your entry file is
```

There are other pm2 commands for various tasks as well that are pretty self explanatory:

- pm2 show app
- pm2 status
- pm2 restart app
- pm2 stop app
- pm2 logs (Show log stream)
- pm2 flush (Clear logs)

to be cont...
