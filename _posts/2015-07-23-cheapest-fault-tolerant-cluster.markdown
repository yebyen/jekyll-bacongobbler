---
layout: post
title:  "Cheapest Fault-tolerant Deis Cluster"
date:   2015-07-23 15:50:10
categories: deis tutorial
---

You can build fault-tolerant clusters on DigitalOcean for less than a half-dollar an hour.  To be fault-tolerant, a Deis cluster needs at least 4GB of RAM per node (really, a bit more than that is needed to facilitate a Ceph recovery when one node goes down, but 4GB of RAM + 4GB of swap on each node seems to do the trick on my test cluster with a light workload).

I figured I could probably do better than $7/day for an "absolute minimum viable cluster."

It's easy to add a swap file (simulating extra RAM) to every node on a CoreOS cluster using fleetctl, and DigitalOcean offers 512MB nodes with 20GB of fast SSD storage for their absolute minimum price-point per node.

According to [installing_deis/system-requirements](http://docs.deis.io/en/latest/installing_deis/system-requirements/) one needs 40GB of storage (or, at the very least, 30GB) on each node in a cluster in order to run Deis.  My prior testing found the requirements, at least to initially set up a cluster, are in fact quite a bit lower than that.  I wanted to see if I could build a cluster out of five or more 512MB droplets, with the minimal 20GB of SSD storage each.

Before anyone tries this in production, it should be noted prominently that this is in all likelihood, an absolutely terrible idea.

It might make the most sense to run the allocation of these nodes using docl and deisctl from an x86-64 machine, if you have one; that is the architecture expected by the precompiled deisctl installer, supported by the Deis documentation for DigitalOcean, etc...

What I had in front of me at the moment of this spark of inspiration was a Samsung 1st-generation (armhf) chromebook model 303C with only 2GB of RAM, sporting a pre-installed chroot with Debian Stretch.

That seemed to be able to handle both of the deis client tools needed for this guide.  Thankfully only python2.7 and golang are required on the machine that we will use to spawn our cluster, so stretch is probably overkill.  I think you could get this done from Debian stable (Jessie, or possibly even Wheezy).

Here are the steps to reproduce this configuration:

1. Get the latest ruby (install 2.2.2 with rvm as of this writing).
2. Fetch the `docl` gem and feed it an auth token from your DO account.
3. Clone [github.com/deis/deis](https://github.com/deis/deis) and checkout the latest version (`git checkout v1.8.0`).
4. Install golang (go1.4.1) from source, then `make installer` in deisctl (v1.8.0) and use the installer in a writable directory in your path.  This will install a compiled `deisctl` there.  (If you are not using armhf, you can just [install deisctl](http://docs.deis.io/en/latest/installing_deis/install-deisctl/) in the usual way.)
5. Install the python `deis-cli` (and dependencies: `libpython2.7-dev`, `python-pip`).  Make sure you have the python cli, NOT the go re-implementation, that version of deis-cli is not finished.
6. Contact DigitalOcean and get your droplet limit increased (the default of 2 is not enough, 5 or more is needed for fault tolerance.)  This guide was tested with 5 nodes; the storage reqs of a node in a cluster with only 3 nodes is likely to be about 40% higher by virtue of having 2 fewer nodes than tested here, so YMMV with other than 5 nodes.<br/><small>Ceph requires to keep three replicas of any data shard in order to maintain `HEALTH_OK`.  Fewer than 3 nodes is absolutely guaranteed not to work.  This step may require a few hours of waiting, depending on the time of day and the level of motivation of the person who accepts your ticket at Digital Ocean.  The correct way to make this request is in your account settings, under User Profile.  You'll find a droplet limit of 2 with a form to request an `Increase`.</small>
7. Make sure you have $40 of DO credit to cover one month of hosting for your cluster:
`0.007*24*30*5` = $25.200, and `0.015*3*30*24` = $32.400, but I was warned by DO support to keep 1 month of credit for all the droplets I planned to run... seems like sound advice.<br/><small>So put about $40 in if you plan to run continuously for any longer than a few days.</small>
8. `export DEIS_NUM_INSTANCES=5`
9. <small>... sanity check? Now would be a good time to consider your monitoring solution, if you planned to include one at all.</small>
10. Follow the [DigitalOcean guide for Deis](http://docs.deis.io/en/latest/installing_deis/digitalocean/) up to the step "Install Deis Platform".  Instead of 4GB, use 512mb as the size.  When you get to the last steps, do not `deisctl install platform` or `deisctl start platform`.  You will need to add your swap first.
11. NB: `docl upload_key` as described in the Deis docs for DO linked above does not seem to be supported anymore, so be logged into your DO web interface to take care of this part.  You will need to add an ssh key to be able to grab the ID for a later instruction with `docl keys`.
12. NB also: "Apply Security Group Settings" throws some errors about invalid tables, a warning that you could be leaking private packets if you are not careful... `iptables: No chain/target/match by that name.` this looks like it could be something to worry about?  I don't know how to check.  Adjust this step for the `$DEIS_NUM_INSTANCES` value that you set in step 8 here... `for i in 1 2 3 4 5`
13. Set up at least 4GB of swap on each node (I used 7168MB, might also succeed with less):<br/>
[gist.github.com/romaninsh/118952ce61643914fb00](https://gist.github.com/romaninsh/118952ce61643914fb00)
14. Now proceed with the rest of the guide from step 10: install the platform with `deisctl install platform`, start the platform, verify that DNS is correctly configured, register a user with the deis-cli...
15. Deploy a test application and scale up to 2 or more workers.  I would suggest scaling up four workers, specifically on the nodes that aren't running deis-builder, but I don't know a clean way to suggest how to do that.  `deis ps:scale web=4`
16. `deisctl install store-admin`
17. ssh into one of the nodes and check out Ceph health status:<br/>
`nse deis-store-admin`, then `ceph -s`
18. Notice the builder node probably fills its disk first and then you are getting `HEALTH_ERR`.  At least I had this problem.  Now that your app is built, you can uninstall it with `deisctl uninstall builder`.  Go ahead and delete some images from the node that was hosting deis-builder, to free up enough space on that node for ceph recovery to proceed safely.  If you need to, you can even disable the swap on that node running builder so it has more space for image building.
19. Reboot or power off that node (or any other node).  Keep watching `ceph -s` until the only warnings you see say running out of space or less severe version of that warning.  You don't want to see any nodes run completely out of disk.  If it happens, log into that node and pick some components you don't need.  You can destroy them with `systemctl stop` and `docker rm` or `docker rmi`.
20. Let the node you tore down in step 19 come back; it adds to the overall storage availability of the cluster, it only costs $5/mo, and 5 nodes being an odd number it prevents your new, cheapest fault-tolerant Deis cluster from getting a split-brained etcd or something bad like that.

You probably also noticed that node's deis-store-metadata went down too, when that node ran out of disk.  So, ssh in and bring it back up with `sudo systemctl start deis-store-metadata`.

That's it!  If all went well, all of your nodes are reporting low disk (but aren't full), and ceph is reporting a healthy `HEALTH_WARN` again, not that lousy `HEALTH_ERR` you saw when one node ran out of disk.  Our app is still up and responding without interruption since step 15!

Fault tolerance for $40/mo or less.  I bet you never thought you could run your data center on only $300 per annum.  (You probably can't.  I hope nobody does.  It can only end in misfortune.  Please for the love of... are you trying it now?)

Don't forget to destroy your droplets when you are done with them, they cost $0.035/hour!

