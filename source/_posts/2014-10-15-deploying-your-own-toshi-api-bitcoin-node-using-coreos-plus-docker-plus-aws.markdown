---
layout: post
title: "Deploying your own Toshi API Bitcoin Node using CoreOS + Docker + AWS"
date: 2014-10-15 15:37:20 -0400
comments: true
categories: 
- Bitcoin
- Cryptocurrency
- Docker
- CoreOS
- AWS
---

In September, Coinbase [open-sourced Toshi](http://blog.coinbase.com/post/97671295752/introducing-toshi-an-open-source-bitcoin-node-for), their in-house Bitcoin full node for querying the Bitcoin blockchain and broadcasting transactions, powered by Ruby and PostgreSQL. Compared to Bitcoin Core, it allows much richer SQL  querying at the expense of a much larger data store (220GB vs 25GB as of September). <!--more-->You can read more about what Toshi can do over in their full [README](https://github.com/coinbase/toshi/).

In this blog post, I'd like to guide you through the process of deploying Toshi using Docker on CoreOS Linux, hosted on Amazon AWS. 

[Docker](http://www.docker.com) is a way to get all the custom configurations required for a particular application in an isolated container without the overhead of a full virtual machine. The Toshi team has provided us with a Docker container so this is a fantastic  way to get all our required configurations to run Toshi in one go.

We'll be using [CoreOS](http://www.coreos.com), which is a very lightweight Linux distribution that requires all of its applications to be deployed using Docker. Since we'll be using Docker and git, and don't need much else bogging down our EC2 machine, this is perfect for our purposes.

We'll set up three Docker containers:

1. The Toshi container containing the Toshi API
2. A Redis container
3. A PostgreSQL database container

We'll link these three docker containers and set up the correct environment variables so we can get Toshi up and running.

Setting up our AWS instance
------------------------------------------

First, we'll launch a new instance on [Amazon EC2](http://aws.amazon.com/ec2/). I will be assuming you know how to do this, but here is the configuration you will need:

- CoreOS beta 310.1.0 as your AMI: http://aws.amazon.com/marketplace/pp/B00KBCWCRG
- m3.medium
- 400GB General SSD Storage on EBS
- For the security group: set ports 22, 5000 TCP open inbound and all traffic open outbound
- Attached to a permanent Elastic IP for convenience

You will need your .pem file from the key pair you launched your instance with to SSH into the launched instance. CoreOS requires you to SSH as the 'core' user, like so from terminal:

	ssh -i /PATH/TO/YOUR/KEY.pem core@YOUR_EC2_PUBLIC_ELASTIC_IP
	
Setting up Toshi, Redis, PostgreSQL
------------------------------------------------

First simply, clone the Toshi repository:
	
	git clone https://github.com/coinbase/toshi.git
	cd toshi

Now we'll build the Docker container for Toshi. Docker looks at the Dockerfile in the folder to figure out how to setup the right configuration for our Toshi container:

	docker build -t=coinbase/node .
	
Don't forget the "." at the end of that line.

Next we'll setup the Redis and PostgreSQL containers that Toshi requires to store blockchain information:

	docker run -d --name toshi_db postgres
	docker run -d --name toshi_redis redis
	
Here, we're using the official Redis and PostgreSQL Docker containers (postgres & redis) with default configurations, since these work perfectly well for our purposes. We've named our two containers 'toshi_db' and 'toshi_redis' to make them easy to reference.
	
Type in `docker ps` to see your two containers running. You should see something like:

	CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
	ebd13a7a2085        redis:latest        "/entrypoint.sh redi   1 seconds ago       Up 1 seconds        6379/tcp            toshi_redis         
	805b03a9e3f0        postgres:latest     "/docker-entrypoint.   11 seconds ago      Up 11 seconds       5432/tcp            toshi_db
	
Now, we'll launch our Toshi container:

	docker run -p 5000:5000 --name toshi -t -i --link toshi_db:db --link toshi_redis:redis coinbase/toshi
	
We're doing a few important things here while launching our Toshi container:

- Naming our container 'toshi' for easy reference
- Mapping port 5000 on the container to port 5000 on our actual EC2 instance so we can access Toshi at this port once we have it up and running.
- Linking our toshi_redis and toshi_db containers to our toshi container. Docker containers are usually live in complete isolation, so we need to do this so they can communicate and exchange data with each other. Docker will give our Toshi container environment variables which tell it what IP and port to connect to our Redis and PostgreSQL at, which we'll use shortly.

After running that command, you should be sitting in the root prompt for your Toshi container, something like this:

	root@f87ea74ff3b2:/toshi#
	
Where f87ea74ff3b2 will be your Toshi container ID. Now, we set environment variables so Toshi knows where to find our Redis and PostgreSQL databases, using the environment variables Docker provided us during linking to know where these are:

	export DATABASE_URL=postgres://postgres:@$DB_PORT_5432_TCP_ADDR:$DB_PORT_5432_TCP_PORT
	export REDIS_URL=redis://$REDIS_PORT_6379_TCP_ADDR:$REDIS_PORT_6379_TCP_PORT
	
Please note that we are using the default login credentials postgres:\<no password\> for our PostgreSQL container. In a production environment, you would obviously set actual login credentials, but this is fine for our purposes of just getting Toshi running.

We need to set the environment that Toshi will work in. For our purposes now, let's use the Testnet (not the actual Bitcoin blockchain):

	export TOSHI_ENV=test
	
To download and work with actual Bitcoin transactions, you want to set 'test' above to 'production'.

We run database migrations for Toshi:

	bundle exec rake db:migrate
	
Finally, we are ready to launch Toshi! We launch Toshi using foreman:

	foreman start
	
Voila! You should see a whole stream of output from Toshi, hopefully with no errors. You'll see it grabbing blocks and transactions from the network. This will take a very, very long time (depending on your network bandwidth). I have not yet timed how long this may take, but suffice to say it will be a long time. I will post more numbers about this as I continue to play around with Toshi.

To see Toshi's interface which shows blocks and transactions as they are downloaded, go to http://YOUR_EC2_ELASTIC_IP:5000 in your browser, where you should see something like [this](https://testnet3.toshi.io/).

You can also play around with the full range of API calls that Toshi supports, like going to http://YOUR_EC2_ELASTIC_IP:5000/api/v0/blocks/<block_hash> to get block information directly from the blockchain. Check out everything you can do with Toshi from the [README](https://github.com/coinbase/toshi).

Congratulations! You've just deployed your own Toshi full Bitcoin node!  In future blog posts, I'd like to help you get more out of your Toshi node, including:

- How to set data-only containers for PostgreSQL and Redis, so that we can make sure the downloaded blockchain remains persistent.
- Play around with Toshi API to find out some cool information from the Bitcoin blockchain. 
- Querying our PostgreSQL with raw SQL to find some *really* cool information from the Bitcoin blockchain.

Till then, enjoy and let me know any issues you run into at me _AT_ soroushjp.com or on Twitter at [@soroushjp](http://twitter.com/soroushjp). Thanks for reading guys and gals :)