# Introduction
Here is some information about deploying a Meteor application to production.


## Warnings
* You should have a basic knowledge of working through Linux operating system. Somethings will be assumed that you know.


## Setup for
* Ubuntu 14.04
* Meteor 1.4

# Preparing your server
## setting up a safeuser

Login to your server as root using password or ssh

Create a safeuser for installation(instead of safeuser you can call it any any name, we will assume safeuser for this document)
```
useradd -s /bin/bash -m -d /home/safeuser -c "safe user" safeuser

```
set password for safeuser
```
passwd safeuser
```
set sudo access for the safeuser
```
usermod -aG sudo safeuser
```

Now after creating the safeuser. Logout as root user.

Use safeuser id to login to your server for the following steps

## Install nodejs
We are going to install nodejs 4*.

You are logged in to your server. Execute the following commands

```
curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -
```
then issue command
```
sudo apt-get install -y nodejs
```

Hope fully if everything goes well you should have node and npm installed
you can check your installation

```
node -v
```
If it shows a version 4.* you are good to proceed to next steps

## setup MongoDB
Here we will be setting up mongodb with oplog tailing.

install mongodb by following steps(copied from mongodb site)

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
```
```
echo "deb http://repo.mongodb.org/apt/ubuntu trusty/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
```
```
sudo apt-get update
```
```
sudo apt-get install -y mongodb-org
```

if everything is fine mongodb should have been setup

you can check if mongodb is up by using mongo to enter mongo shell
```
mongo
```

By default in this setup mongo uses /var/lib/mondodb  for storing db files. If you need to change location, refer to mongodb site.

Let us setup oplog. You have to change mongod.conf which is in /etc folder for this.

Take a backup of mongod.conf before you edit the file.

use any of your favourite editor to edit /etc/mongod.conf file. go to line which shows

#replication:
you should replace it with below lines and save
```
replication:
  replSetName: meteor

```

now restart mongo service by issuing
```
service mongod restart
```

now connect to mongo shell and issue the following commands
```
var config = {_id: "meteor", members: [{_id: 0, host: "127.0.0.1:27017"}]}
```
```
rs.initiate(config)
```

you should get a response something like

{"ok":1}

come out of the mongo shell and restart mongo service
```
service mongod restart
```

now everything should be setup for enabling oplog tailing on meteor. To check  connect to mongo shell and  issue command
```
rs.status()
```

it should display something about primary node



We are using:
1. Key based SSH for accessing your Linux box and assume you have root access to the box
2. Nodejs 4*
3. MongoDB 3.*
5. pm2 for starting Meteor app

Assumption
1. Ubuntu 14.04 vps or box where you have root access.
2. use /opt folder on target box for storing meteor app
3. you have a running meteor app(1.4* release) on your local machine





1. PM2 for all process management (see sample **pm2.json** which you have to start manually the first time on the server) -- and read PM2 docs for more info
2. tengine with load balancer and sticky sessions -- see our sample **nginx.conf**
3. Shell scripts to push build via SSH -- **remote-build.sh**
4. Shell scripts to install build remotely on server -- **deploy-build.sh**
5. Of course, nodejs / npm should be installed on your server (as well as pm2) -- and .sh are executable
6. Key-based SSH

# Folders

On the server,

1. we assume the app will be started from /home/meteor/build
2. pm2.json and deploy-build.sh should be located in /home/meteor/scripts


# Process
1. We compile on our dev machine
2. Push the tarball onto server
3. Decompress and install on server
4. Reload PM2

All this is done by calling local script **./build-remote.sh** which takes care of everything (if you set things based on the folders above).

# A note about tengine
Tengine (http://tengine.taobao.org/documentation.html) is a fork of nginx but with solid production-grade features (that you would have to pay for with the pro version of nginx). nginx.conf in this repo contains our (sanitized) setup.

# Bonus
1. You will notice in the nginx.conf we are accessing the pm2 web interface to get real time view of how our processes are running (Really cool!) -- see image at top of this readme
2. You will also notice in nginx.conf that status.example.org provides real-time view of the three meteor processes we have launched (load balancer status -- see image below) ![Status](https://github.com/ramezrafla/meteor-deployment/blob/master/screenshots/status.png?raw=true)
3. We use Icinga2 (you can use Nagios too) to monitor server health, including checking on the load balancer status and pm2 processes (not included yet)
4. We use Cloudfront CDN (see http://joshowens.me/using-a-cdn-with-your-production-meteor-app/ with some improvements) and this is reflected in the nginx.conf

# Meteor-based load balancer / deployment

Many in the community use the Cluster package for managing meteor instances. This is a **lousy** solution. The explanation of the authors is that HAProxy is too complex. Agreed, but nginx / tengine include proper load balancers. We shouldn't use meteor to run webserver functions. This looks like the classical dev thinking s/he is a system admin.

I personally dislike mup and even more mupx. The latter uses docker. The approach suggested here is for truly production-grade deployments. This is such an important issue, that I believe a good company either outsources deployment / hosting (e.g. Galaxy) or does it right.
