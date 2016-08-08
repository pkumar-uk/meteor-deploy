# Introduction
Here is some information about deploying a Meteor application to production. It assumes everything will be in one server.


## Warnings
* You should have a basic knowledge of working through Linux operating system. Somethings will be assumed that you know.

I am not covering nginx, ssl and firewall setup.


We are using:
1. Key based SSH for accessing your Linux box and assume you have root access to the box
2. Nodejs 4*
3. MongoDB 3.*
5. pm2 for starting Meteor app

Assumption
1. Ubuntu 14.04 vps or box where you have root access.
2. use /opt folder on target box for storing meteor app
3. you have a running meteor app(1.4* release) on your local machine
4. Mongodb will be in the same server

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


That ends up mongodb setup. Now you can load old data if you have.


## build your meteor app for deployment

This has to be done on your local machine where you have your meteor code. Follow Meteor guidelines for building the app. I use the following

```
meteor build ../bld-yourapp --architecture os.linux.x86_64 --server=https://yourdomain
```

this will create the deployment bundle  file in bld-yourapp folder in the format yourapp.tar.gz. Now you should copy this file to your server in the /opt folder

Login to your server and go to /opt folder. Now you need to unpack the copied bundle file. You can use the commands
```
sudo tar -xvzf yourapp.tar.gz
```
this will create a folder called bundle which contains your meteor app code. You can rename the bundle folder to yourapp folder now.

Assumed that you have renamed your bundle folder to your app

change directory and issue the following command

```
cd yourapp/programs/server
sudo npm install
```
if for some reason npm install gives error. Run the command with --unsafe-perm flag

it should rebuild node modules and give a list that has been built.


## install pm2
you are logged in to your server. Issue the following commands
```
sudo npm install pm2 -g
```

now you need to setup pm2 for ubuntu issue the command
```
sudo pm2 startup ubuntu
```

This will install pm2 for you to startup atutomatically using user safeuser. Now you have to define your app to run with pm2

create a file pm2.json in safeuser home folder it should contain in the following format

```
"apps": [{
  "name": "app-1",
  "cwd": "/opt/yourapp",
  "script": "main.js",
  "env": {
    "NODE_ENV": "production",
    "WORKER_ID": "0",
    "PORT": "3000",
    "ROOT_URL": "https://yourdomain",
    "MONGO_URL": "mongodb://localhost:27017/meteor",
    "MONGO_OPLOG_URL": "mongodb://localhost:27017/local",
    "HTTP_FORWARDED_COUNT": "1",
    "METEOR_SETTINGS": {}
  }
}]
```
you can enter as many node apps as you want to start using pm2. Load yourapp settings.json information to METEOR_SETTINGS in pm2.json. Here we assume the yourapp is going to run at port 3000.

Now let us check if your app comes up.

Run the following command .....assume you are in directory where pm2.json is present
```
sudo pm2 start pm2.json
```
If everything works fine you should see something like this

```
┌────────────┬────┬──────┬──────┬─────────┬─────────┬────────┬──────────────┬──────────┐
│ App name   │ id │ mode │ pid  │ status  │ restart │ uptime │ memory       │ watching │
├────────────┼────┼──────┼──────┼─────────┼─────────┼────────┼──────────────┼──────────┤
│ app-1      │ 0  │ fork │ 5009 │ online  │ 1       │ 28h    │ 160.406 MB   │ disabled │
└────────────┴────┴──────┴──────┴─────────┴─────────┴────────┴──────────────┴──────────┘

```

You can check pm2 commands to check other options. The log from the app will be written to

/home/safeuser/.pm2/logs    you can check if your app has come you correctly. If not, correct the errors.


you can check your app is running at http://yourserverip:3000

Good luck..........hope things go fine.

This is not the end. You now have to configure nginx and SSL, and setup firewall to protect your app.
