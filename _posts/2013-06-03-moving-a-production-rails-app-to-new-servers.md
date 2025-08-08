---
layout: post
title: "Moving a Production Rails App to New Servers"
author: "Ben Roesch"
---
At [Inkling Markets][1] we recently moved off of dedicated servers and onto Rackspace Cloud Servers. In order to accomplish this, we created something that we called the \"migration playbook,\" which outlined the steps we needed to take to ensure a smooth transition. When we first started, I looked around for something similar but didn\'t find much, so I thought I\'d share some interesting pieces and lessons learned from our process. I left out a number of tasks that were less interesting or difficult, but some our major pieces included:  
  
* New web/app servers
* Database migration
* Application data/image migration
* Email migration
* DNS Transfer
* Day-of Playbook

  
Our plan called for setting up the new enivronment in its entirety, beating on that environment for a while to work out any kinks, and then having a final cut-over where we actually started pointing the inklingmarkets.com domain to the new servers.  
  
## New Web/App Servers

The first thing we did was set up new web/app servers. As part of the transition, we decided to switch from our old apache/haproxy/mongrel setup to nginx/unicorn. In addition to getting onto a newer and hopefully faster stack, this allowed us to cut out the haproxy piece from our puzzle. While installing nginx and unicorn is pretty straight forward, we did run into one application-related snag. When serving up user-uploaded images from the application, our app would add an X-Sendfile header to the response headers. This would result in apache serving the file directly, instead of having the mongrel servers do it. Nginx doesn\'t support X-Sendfile, but fortunately it has a similar feature, called X-Accel. So instead of adding an X-Sendfile header, we changed the rails app to add an X-Accel-Redirect header, containing the path to the file.   
  
## Database Migration

The database migration and cutover was probably the biggest single piece of the transition. If you\'re interested in how we handled it, I wrote up a separate article that covers it in depth, which you can read [here][2].  
  
## Application Data &amp; Image Migration

Our app included some user generated images that are saved on disk. To migrate this data, we used rsync. From the destination server, we used a command that looked like this:  
> `rsync -aiv deployuser@source.server.ip:/path/to/images/dir/
> /path/to/images/dir/`

To minimize downtime at the time of the actual transition, we ran this when setting up the servers to move over the bulk of the files. We then ran it again on the day of cutover, and then again after we put up maintenance pages on our old servers. The subsequent instances only needed to move the changes since the last time we ran the command, so they were fast relative to the first time.  
  
## Email Migration

Moving our outbound email was another major piece that merited its own post. You can find it [here][3].  
  
## DNS Transfer

When the day of cutover came, we had a number of tasks listed that had to get done, but the one that made the switch \"official\" was updating our DNS settings to point to the new servers. The process for this, while suspenseful, was pretty simple:  
  
1. Lower the DNS TTL (we put it down to 30 seconds) far enough ahead of time that the old records will expire. (ie. if your ttl is 30 mins, lower it more than 30 mins before you want to make the switch)  
2. Once the other tasks are complete and the new servers are ready to start getting traffic, update the A records to point to new load balancer  
3. Verify that things are working correctly on the new servers  
4. Once satisfied, increase the TTL back to the previous value.  
  
## Day-of Playbook

If anyone is going through a similar process, there\'s a final piece of advice that I\'d give. For our migration, we put together a day-of playbook that listed, in sequential order, all of the tasks that needed to be completed. Where pertinent, it included specific commands to run and where to run them. This helped the migration go smoothly and quickly, and I would highly recommend it. 

[1]: http://inklingmarkets.com
[2]: http://benroesch.com/2013/05/28/moving-a-live-mysql-database-to-a-new-server/
[3]: http://benroesch.com/2013/05/30/migrating-application-email-sending-to-a-new-server/
