# Use Alibaba cloud to receive Sigfox callbacks

First create an Alibaba account... and log into the console. 

![](img1.png)

## Create the necessary instances on Alibaba 

### Create Virtual Private Cloud


Go to Networking -> Virtual Private Cloud then VPC in the left menu. 

Choose the Region that you want to use. In my example I will use Singapore. Note that all the other products will have to be created in the same Region. 

Click 'Create VPC'

* Give it a name, description and choose CIDR if you like. 

![](img2.png)

* Then click Next to create a Vswitch 

![](img3.png)

Give a name as well, choose the Zone, this is very important because it is mandatory that the ECS instance and the MongoDB are in the same zone, otherwise they won't be able to communicate. 
Choose a CIDR as well, I'm using 192.168.1.0/24 for this example.

![](img4.png)
 

### Create Mongodb

Click Home and then go to Apsara DB and choose Apsara DB for MongoDB 

Click 'Create Instance' (the image below has already one instance of MongoDB, we will create a second one). 

![](img11.png)

* Choose Pay as you go on top (ReplicaSet)
* Choose the Region and Zone (if the Zone that you chose for the VPC is not available, you will to restart creating the VPC with that same Zone). 
* Network type VPC
	* Choose the VPC and vswitch created. 
* Set password (this will be DB_PASSWORD for the rest of the article).

![](img10.png)
	
Then click Buy now. It will take a bit long to create the instance, so proceed with the ECS creation we'll come back here. 

![](img8.png)

### Create Elastic Cloud Instance 

Now click home. Then go to Elastic Computing -> Elastic Compute Service then Instances in the left menu.

Click 'Create Instance'

* Pricing model -> Pay as you go 
Choose the region and the zone. For our example, Singapore Zone A. 
* Choose an instance type. I chose Gen3 General type n4 Small. This will be sufficient for the example. 
* Network type : VPC
	* Choose the name of the VPC instance and VSwitch that you just created. 
* Select the default Security group 
Network bandwidth set to 100Mbps
For the OS we choose Centos 7.4 64 bits. 
* Check the Activate Server Guard
Storage Ultra Cloud Disk 40GB. 
* Authentication Method: Set Password
	* Choose a root password (a safe one). (this will be ECS_ROOT_PASSWORD now on). 
* User data -> Later
* Give the instance a name, buy 1 unit

Click Buy now. And then Activate. 

![](img6.png)

![](img7.png)

You should now have an additional line in the instances list. It will turn to green and be "running" in a few minutes. 

As soon as its ready you must take note of the IP Address (names PUBLIC_IP) of your server. Note the private IP address as well (named PRIVATE_IP) , it will be useful next. 

### Mongodb configuration

Now go back to the Console home and Apsara DB for Mongo DB. 

We need to do 2 things : 

1. Obtain the connection string for the database. 

Click 'manage' on the line of the desired instance. 
In the basic information tab, go at the bottom of the page, there is a 'Useful information', copy in your favorite notepad the mongoDB URI 

	mongodb://root:DB_PASSWORD@dds-gs5b4779ee0XXXXXX.mongodb.singapore.rds.aliyuncs.com:3717,dds-gs5b4779ee0XXXXXX.mongodb.singapore.rds.aliyuncs.com:3717/admin?replicaSet=mgset-300103085

![](img13.png)
![](img14.png)

2. Include the ECS server IP address (PRIVATE_IP) in the whitelist for the MongoDB. 

In the instance go to the security Tab. 
In Whitelist setting, click Import ECS Private IP. 
You should find the private IP address of the ECS instance just created. Select it, it should now be in the whitelist. 

![](img17.png)

All set ! Let's do some SysAdmin work... 

## Prepare the server

Open your favorite terminal and Log on the server :  

	ssh root@PUBLIC_IP

### Install node.js 


	# Download the latest stable Linux Binary 64 bits 
	wget https://nodejs.org/dist/v8.9.1/node-v8.9.1-linux-x64.tar.xz 

	# The installation procedure can be found here: https://github.com/nodejs/help/wiki/Installation 
	sudo mkdir /usr/lib/nodejs
	tar -xJvf node-v8.9.1-linux-x64.tar.xz  -C /usr/lib/nodejs
	sudo mv /usr/lib/nodejs/node-v8.9.1-linux-x64/ /usr/lib/nodejs/node-v8.9.1
	touch .profile
	# Edit your .profile file : 
	vi .profile

Add the lines below to your .profile : 

	# Nodejs
	export NODEJS_HOME=/usr/lib/nodejs/node-v8.9.1
	export PATH=$NODEJS_HOME/bin:$PATH

Exit vi ( esc : wq if you are wondering) 

	# edit ~/.bashrc
	vi ~/.bashrc

Add the line below to your .bashrc 

	. $HOME/.profile

Exit vi. 
	
	# Run straight away the .profile
	. ~/.profile

	# We can now test node and npm, this should give you nice output as shown in the node installation procedure. 

	node -v 
	npm -v

	# cleanup 
	rm node-v8.9.1-linux-x64.tar.xz
	
	# Now add permission for node to open port < 1024 with non root users 
	sudo setcap cap_net_bind_service=+ep `readlink -f \`which node\``

### Install mongodb tools

Procedure can be found here : [https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/]()

	#Edit the repo file before launching yum
	vi /etc/yum.repos.d/mongodb-org-3.4.repo 

Insert in the file: 

	[mongodb-org-3.4]
	name=MongoDB Repository
	baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
	gpgcheck=1
	enabled=1
	gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc

Save and exit

	sudo yum install -y mongodb-org-shell
	sudo yum install -y mongodb-org-tools

We can now test the connectivity between our ECS and mongoDB Server. 
!! Don't forget to add the ECS instance in the MongoDB Security list (if this was not done before) 

Connect to the mongoDB Instance : 

	mongo --host dds-gs5b4779ee0XXXXXX.mongodb.singapore.rds.aliyuncs.com:3717 --authenticationDatabase admin -u root -p

	show collections #This should return an empty list if the mongodb server is new
	exit

### Install the callback server

Now that we are set, let's use [https://github.com/sigfox/sigfox-nodejs-callbacks]() in our ECS server. It will create a HTTP server, that will accept the connections from the Sigfox backend, store them on the MongoDB server and serve a Web App that reads from the DB and displays the messages. 

	# We need unzip first... 
	yum install unzip

	# Get the archive and unzip it. 
	wget https://github.com/sigfox/sigfox-nodejs-callbacks/archive/master.zip
	unzip master.zip

	cd sigfox-nodejs-callbacks-master/

	# Run the installation procedure
	npm install

	# We need to create a local conf file : 
	vi config.local.js

Add the following content in your file (use the URI string that we captured in the mongodb instance config page, just add in your DB_PASSWORD) 

	'use strict';

	process.env.DEBUG = '*';
	process.env.DATABASE_URL = 'mongodb://root:DB_PASSWORD@dds-gs5b4779ee0XXXXXX.mongodb.
	singapore.rds.aliyuncs.com:3717,dds-gs5b4779ee0XXXXXX.mongodb.singapore.rds.aliyun
	cs.com:3717/admin?replicaSet=mgset-300103085';
	process.env.PORT = '80';

Now we can start our server 

	npm start

Test out the message reception : open another terminal window and run this from your local machine : 

	curl -X POST -d 'id=simulation&time=1500000000&station=future&data=d474' http://PUBLIC_IP/sigfox

Now you should see it in the dashboard at http://PUBLIC_IP/ 

![](img16.png)
![](img18.png)


![](https://media.giphy.com/media/vtVpHbnPi9TLa/giphy.gif)
 

### Final touch 

Let's not keep logging with the Root user on the Linux server (bad practice). 

	# Add 'webapp' user 
	sudo useradd webapp  -m -s /bin/bash 
	# Change its password
	sudo passwd webapp 
	# Move the directory to the webapp user
	mv sigfox-nodejs-callbacks-master/ 
	cd ~webapp/
	chown -hR webapp sigfox-nodejs-callbacks-master/
	#Edit the sudo permission 
	visudo
	
Go down the sudoers file and edit SecurePath (to add Node.js) and add Webapp to the users allowed to sudo

	
	Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/lib/nodejs/node-v8.9.1/bin
	## ... 
	webapp  ALL=(ALL)       ALL

Now we can install PM2 [https://github.com/Unitech/pm2]()
	
	npm install pm2 -g

Exit the ssh session and log back in with the newly created user : 

	ssh webapp@PUBLIC_IP

	cd sigfox-nodejs-callbacks-master
	sudo pm2 startup # the script will generate a command to copy and paste into the terminal. 
	sudo pm2 start server.js
	
	
Congrats, you can know go to your Alibaba console and make a snapshot of your seruver !! We are done with this.  
	
## Configure the Sigfox backend to send the callbacks to our newly created server

If you do not work in a location where Sigfox provides coverage, check the Sigfox SDR dongle here : [https://build.sigfox.com/sdr-dongle]()

If you do, Go to [https://backend.sigfox.com]()

Click on device Type menu at the top, search for your device and click on its name (right handside)

![](img19.png)

If we go back to our browser with the public IP address, as soon as the device sends a message, we should be able to see it ! 

![](img20.png)


## TODO

* Make server https
* Enable auth between Backend and Server
* ... 
