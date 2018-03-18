# About

Stuff learnt from an AMA by the sysadmins of Reddit. https://www.reddit.com/r/sysadmin/comments/r6zfv/we_are_sysadmins_reddit_ask_us_anything/

Date: ~2013


## Stats

- Peak bandwidth: 924.21MBits / second. They used Akamai heavily
- Aggregate size of databases: 2.4TB. Seems to be growing a few GB per week
- On load balancer: ~8K established connections, ~250K in time wait (with very short time wait timeout)


## What they use

- Akamai
- AWS (284 running instances, 161 were app servers)
- Puppet
- [Ganglia](http://ganglia.sourceforge.net/)
- [Zenoss](http://www.zenoss.com/)
- RabbitMQ
- MCollective
- Central memcached servers (with pylibmc). Each app server has small memcached instance for _very_ local caching that cannot suffer network latency
- rsyslog
- Log consolidation: rsyslog with RELP module
- Hadoop (for in-house data warehouse)


## Interesting stuff

- They use HAProxy on EC2 instances instead of ELB. Total 8 instances
  - ELB is HAProxy with an API. Limited control over instance size of ELB. Initially set to very small instance
  - ELB load balancing is done via round-robin DNS. When one of the backing instances crashes, any cached DNS on the Internet is going to suck. A lot of devices/software/ISPs still cache DNS incorrectly
  - If ELB has these, it will be useful:
    - Static VIP support. Just round-robin DNS is not acceptable
    - Granular control over instance size that backs ELB
    - More rule functionality in load balancing. Very limited compared to HAProxy
- At one point, Postgres replication issues were taking down the site very often.
  - These were due to EBS failures. They had to login and start addressing replication immediately to prevent really bad breakages
  - Upgrading to Postgres 9 and moving away from EBS took care of it
- When they took Reddit down during SOPA protest, they had to prepare for severe amount of immediate load because everyone knew the site was coming back online
  - So they cannot do anything that cause the caching layers to clear. Otherwise site would have fallen flat on its face when it came back online
- Load testing: users
  - They do not have a load testing infra that can replicate user traffic
  - At every place one of them has worked at, one of the most difficult problems is to simulate load properly. With dynamic services like reddit, it takes a lot of work to develop a suitable load simulator
- Non logged in traffic hits Akamai's cache
- Security focus: ensuring evildoers cannot get into app and do evil things. Since they are only hosting web, the infra has a very small number of vectors which are under decent security controls
  - Most common attack: people trying to 'DDOS' them by scraping one URL over and over again
- For async stuff, RabbitMQ is used. For instance:
  - Votes
  - Comment tree recomputing
  - New comments
  - Thumbnailer
  - Search engine updates
- IPv6: Akamai supports it and takes most burden off them
- They keep a close eye on request rate hitting infra and real time stats from Google Analytics
- Worst downtime: https://redditblog.com/2011/03/17/why-reddit-was-down-for-6-of-the-last-24-hours/
- Silliest downtime: `iptables -t nat -L` to check rules on primary load balancer. This loads all the iptables modules, including conntrack. Conntrack table immediately filled up and took site down for a few seconds
- Servers are patched as necessary. They subscribe to all security alert notification lists
- Backup strategies: encrypt and send to S3. There's also one backup Postgres server where everything from every database cluster is written to (for more real time backup needs)


## Challenges

- Starting from scratch on a lot of stuff
- Bottlenecks constantly popping up. Fix one bottleneck and the increased throughput introduces multiple new bottlenecks
- Cannot touch memcached boxes. Reheating them will be very painful
  - At their scale, they must make heavy use of caching whenever possible. Hence shutting everything down and starting everything back up is a painful process
  - Need to engineer a clean way to reheat caches without having users hit the site
    - One idea is to replay access logs against front-end hosts
    - Another idea is to send increasing amounts of real traffic. Say every 1 in 4 requests gets to somewhere other than the maintenance page


## Advice

- Spend a lot of time working on own stuff. Eg, set up a web / database server just for the hell of it.
  - Break stuff, rebuild it, repeat
  - Find every interesting thing you can do on your home server and try it. Even if you are never going to use it personally.
  - If anything breaks or doesn't make sense, don't drop it until you truly understand what is going on
  - Avoid adopting any cargo cult mentality at all costs
  - If that sounds like an extreme bore, reconsider sysadmin aspirations
- Certs _may_ help you get an interview at some companies and leverage for promotions at current workplace
  - But they mostly demonstrate at most a shallow understanding of a system
  - If you already know a system inside out, doesn't hurt to spend a small amount of time getting a cert


## Bare metal vs. cloud

- Bare metal:
  - Load balancers and database servers will benefit from bare metal
  - Plus point: can experiment with new hardware
- Cloud:
  - App servers will benefit from cloud
  - Plus points: nice to not have to worry about things like networking infra, installing new hardware, ordering new hardware, rack power, etc


## Mistakes they made

- Everything used to be in one security group


## What they were working on

- Automating most infrastructure tasks, such as building out new servers
- Getting the site to run in more than one region. Huge project that will require a lot of work throughout entire stack
