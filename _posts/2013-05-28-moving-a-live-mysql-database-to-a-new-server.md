---
layout: post
title: "Moving a Live MySQL Database to a New Server"
author: "Ben Roesch"
---
At [Inkling Markets][1], we recently moved off of dedicated servers and onto Rackspace Cloud Servers. As part of this transition, we needed to move our mysql database onto our new database server. If you Google for how to move a mysql database, most of the answers that you find will involve stopping mysql, taking a full backup, copying that backup to a new server, and then restoring it. In our case, with a non-trivial sized database, doing that would take far too long. In order to minimize downtime during the transition, we came up with a different approach.  
  
The general outline of the plan was:  
  
  
1. Set up new db server as a slave of the old master  
  
2. Put up a maintenance page for our app and let the new db finish replicating old master  
  
3. Promote the new db server to master  
  
4. Profit!!!  
  
  
In our old setup, we had one server acting as master and one replication slave. To start, I spun up a new db server, installed the same version of mysql that was running on our old servers, and added firewall rules to allow the new db server to talk to both the master and slave. On the new server, which was going to serve as our future master, I configured it to have binlog on, relaylog on, and log-slave-updates off. In our case, the config settings look like this (log-slave-updates is off by default):  
  
> `log-bin=/usr/local/mysql/var/newhostname-bin-loglog-bin-index=/usr/local/mysql/var/newhostname-bin-log.indexrelay-log=/usr/local/mysql/var/newhostname-relay-logrelay-log-index=/usr/local/mysql/var/newhostname-relay-log.index`

  
  
With this setup, the new db server can be run as a slave of the old master, but will not log the updates it gets from that replication in its binlog. However, since its binlog is on, it is ready to be promoted to master in the future (ie. when we made the actual transition to it being our production db).  
  
Since the plan here is to have the new server start out as a slave of the old one, I added a replication user to the old server for the new server\'s ip. On the old server:  
> `mysql> CREATE USER 'repl'@'new.server.ip' identified by 'repl
> password here';mysql> GRANT REPLICATION SLAVE ON *.* TO
> 'repl'@'new.server.ip';`

  
  
Next, I stopped mysql on the new server and set `server-id=3` in its config file. In order to get data over to the new db server, I stopped the replication on our old slave:  
  
> `mysql> STOP SLAVE;mysql> FLUSH TABLES;$ /etc/init.d/mysql stop`

  
  
And used rsync to copy over the mysql data directory from the old slave to the new db server:  
  
> `$ rsync -avz user@old.slave.ip:/usr/local/mysql/var
> /usr/local/mysql/var`

  
  
I then turned the old slave back on and let it catch back up to master:  
  
> `$ /etc/init.d/mysql start`

  
  
And to verify the slave there is working (you can run this command twice and Exec\_Master\_Log\_Pos should be increasing):  
> `mysql> SHOW SLAVE STATUS\\G`

  
  
Next, some files in the data dir needed to be renamed and/or updated. MySQL names some of the files according to the name of the host it is on, so they needed to be changed from the old host name to the new one. You\'d need to replace newhostname with the name of your server:  
  
> `$ cd /usr/local/mysql/var$ mv oldhost-relay-log.index
> newhostname-relay-log.index$ mv oldhost-relay-log.#####
> newhostname-relay-log.#####$ sudo vim relay-log.info (change path &
> name of the relay log on the first line to the new location & host)$
> sudo vim newhostname-relay-log.info (change path & name of the relay
> log on the first line to the new location & host)$ sudo vim
> master.info (make sure the replication user and pw are correct)`

  
  
Once I fired up mysql on the new server, it immediately began pulling updates from the old server, since the data directory was copied directly off our old slave. Now, the actual switch over to the new server.  
  
I put up a maintenance page on our web servers, so that no new requests would go through to the application. Then, I locked the tables on the old server so that no new updates could be made there:  
> `mysql> FLUSH TABLES WITH READ LOCK;`

  
  
And then waited for the new db server to finish processing updates. You can verify this by checking the process list (on the new server), which should say \'Has read all relay log.\'  
> `mysql> SHOW PROCESSLIST;`

  
  
I made a copy of the data directory, just to be safe:  
> `$ cp -r /usr/local/mysql/var /usr/local/mysql/var_backup/`

  
  
And then promoted the new db to master:  
> `mysql> STOP SLAVE;mysql> RESET MASTER;`

  
  
After a few other non-db related tasks, we changed our DNS entries to point to our new servers, and the process was complete.  
  
  
If you found this helpful, I\'d love to hear from you on twitter: [@bcroesch][2]

[1]: http://inklingmarkets.com
[2]: http://twitter.com/bcroesch
