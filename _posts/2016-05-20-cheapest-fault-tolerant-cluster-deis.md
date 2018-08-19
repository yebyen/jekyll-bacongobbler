---
layout: post
title: Cheapest Fault-Tolerant Cluster For Deis V1 PaaS
description: "We look at a cheap, quick, and dirty way to bring up a fault-tolerant cluster for Deis V1 PaaS on DigitalOcean."
tags:
  - Demo
  - Fault-Tolerance
author: kingdon_barrett
---

*This guide is targeted to the [Deis v1.13 LTS branch](http://deis.io/deis-1-13-lts). With over 6 million downloads, Deis V1 PaaS has never been more popular. These instructions will not work with [Deis Workflow](https://deis.com/), the Kubernetes-native PaaS, which is still currently in beta.*

I am going to demo a cheap, quick, and dirty way to bring up a cluster for Deis V1 PaaS on [DigitalOcean](https://www.digitalocean.com/).  This is the cheapest fault-tolerant configuration that I could manage.  That is, a cluster on which some nodes can go down&mdash;but the cluster and its platform services remain available!

<!--more-->

Before we go on, let's distinguish *fault-tolerant* from *highly available*.

## Fault-Tolerant or Highly Available?

A fault-tolerant Deis cluster continues operating, even in the event of some partial or complete failure of some of its component nodes.  This guide will show you how even a cheap Deis cluster can be fault tolerant&mdash;and to what extent.

A highly available (HA) Deis cluster is one that is continuously operational, or never failing.  HA is the harder of the two to achieve, and this tutorial won't show you how to do that.

High availability is something that isn't provided out-of-the-box in most cases.  If HA is one of your goals, your sysadmin or dev team will be better qualified to say what obstacles you will face in maintaining a standard of high availability for your apps.

## Simplified Prerequisites for Deployment

To spin up some CoreOS hosts and install the Deis platform you will need: Git, the classic GNU make tool, and (depending on your architecture) you might need a Go compiler to build the Deis command-line interface (CLI) tools.

Compiling the CLI tools is an optional step on x86_64, the architecture preferred by CoreOS.  There, users can just download [`deis`](http://docs.deis.io/en/latest/using_deis/install-client/#install-the-deis-client) and [`deisctl`](http://docs.deis.io/en/latest/installing_deis/install-deisctl/#install-deisctl) directly.

The recommended way to provision a cluster is to use [Terraform](https://www.terraform.io/)&mdash;a tool from Hashicorp, the company that makes Vagrant.  Thanks to Terraform, the process is almost exactly the same on all supported cloud platforms.

With Terraform and DigitalOcean, as we will show, five nodes can be provisioned and be online in as little as two minutes!

## Preparing the Local Environment

The Terraform tools are distributed [as binary packages](https://www.terraform.io/downloads.html) and [source code](https://www.github.com/hashicorp/terraform).  The binary packages obviate the need to install a Ruby interpreter, gems, or platform-specific software.  Any platform-specific bits can be handled between the Terraform driver and the scripts in your platform's `contrib` directory in the Deis repository.

Download and install Terraform.

At the time of this writing, the [latest release](https://www.terraform.io/downloads.html) of Terraform was 0.6.16.

Next, make sure your DigitalOcean account is able to start five droplets.  In case you're not sure about that, go to [your account page](https://cloud.digitalocean.com/settings/profile) and check your droplet limit.

If you don't have a DigitalOcean account, you can click [my referral link](https://www.digitalocean.com/?refcode=bc8bb2bab204) to get some free credit when you sign up.

Another way to get free credits is to [contribute something to Deis](https://deis.com/blog/2015/code-for-credit-digitalocean/)!

If you prefer another cloud vendor, you can trivially adapt these instructions.

Before you continue, make sure you have cloned [the Deis repository](https://github.com/deis/deis) and checked out the latest release.

{% highlight text %}
$ git clone https://github.com/deis/deis
$ git checkout v1.13.1
{% endhighlight %}

## SSH and API Secrets

You must create an SSH keypair on your dev machine.

Generate a new SSH key by running:

```
$ ssh-keygen -q -t rsa -f ~/.ssh/deis -N '' -C deis
```

Now print its fingerprint and public part by running:

```
$ ssh-keygen -lf ~/.ssh/deis.pub
$ cat ~/.ssh/deis.pub
```

The public part must be uploaded to your DigitalOcean account prior to starting the cluster with Terraform. You can do this in your DigitalOcean account settings, under *User Security*, select *Add SSH Key*.

Now generate an API token. The option is under *API*, *Generate New Token*.

The generated token is the key to provisioning new CoreOS droplets using Terraform, so it should stay in your account's personal access tokens until after you've successfully run `terraform` to bring up the cluster nodes next.

You can safely keep the token after that, or delete it once Terraform has completed bringing up all five nodes.

### Deploying a CoreOS Cluster With Terraform

While you still have all of those strings handy, open a text-editor and prepare this blob of text by adding your API token and SSH key fingerprint to it.

(But don't run it just yet...)

```
$ terraform apply -var 'token=abcdef123456' \
   -var 'region=nyc3' -var 'prefix=deis' \
   -var 'instances=5' -var 'size=512MB' \
   -var 'ssh_keys=ab:cd:ef:12:34:56: ....' \
   contrib/digitalocean
```

Note `contrib/digitalocean`, a directory in the `deis/deis` git repository that you checked out earlier, contains a file `variables.tf` that sets default values for these parameters which you can compare against the specific options I provided above.  The other scripts in `contrib/` help Terraform provision CoreOS on a number of other, different providers.

The `apply` verb has two mandatory parameters that you must set now to make it work: `token` and `ssh_keys`.  Two more are required by this tutorial to make your cluster both cheap and fault tolerant: `instances=5` and `size=512MB`. The other parameters, `region` and `prefix`, already have good defaults defined in the `variables.tf` file, so you may omit them. They are shown here for completeness.

Running the `terraform apply` command will eventually complete, printing the IP addresses of your new CoreOS nodes.  You will need those IP addresses to set up DNS, so you might want to make a note of them. But: don't run it just yet!

The `token` argument should be set to your DigitalOcean API token.

The `ssh_keys` argument should be set to your SSH key fingerprint generated in the previous section.

Once you've prepared and run the correct command for your CoreOS cluster, you can watch Terraform spin up all five CoreOS nodes in parallel.  Then you can collect the host addresses it prints when it returns.

But again: don't run it just yet!  There's one more thing...

## Provisioning CoreOS for the Deis Cluster

Your Deis nodes will discover each other using the etcd discovery URL service. That requires a unique cluster URL to be generated, below with `make discovery-url`.  The generated discovery URL is informed by `DEIS_NUM_INSTANCES=5`, which tells etcd to expect five full member nodes.

Run the following commands together, in this order. Except instead of my example `terraform` command, you should be using the `terraform apply` command you created for yourself.

{% highlight text %}
$ cd deis
$ export DEIS_NUM_INSTANCES=5
$ make discovery-url
$ terraform apply -var 'token=abcdef123456' \
   -var 'region=nyc3' -var 'prefix=deis' \
   -var 'instances=5' -var 'size=512MB' \
   -var 'ssh_keys=ab:cd:ef:12:34:56: ....' \
   contrib/digitalocean
{% endhighlight %}

Make a note of the IP addresses that are printed when this step terminates, and save them for your [DNS config](http://docs.deis.io/en/latest/managing_deis/configure-dns/#dns-records) later, or use them in your `/etc/hosts` file for [a quick and dirty approach](https://gist.github.com/yebyen/efb65e358807f2ad4b52#file-hosts).

You can watch a two-minute demonstration of this entire process, exaggerated in pace and after a lot of practice, here:

<iframe width="683" height="384" src="https://www.youtube.com/embed/PR2mQMefu9c" frameborder="0" allowfullscreen></iframe>

After this is done, we need to first address the lack of sufficient RAM before we can begin deploying Deis. Terraform will only provision CoreOS, not Deis.

On a normally-sized cluster, we could proceed on from the end of the [platform-specific instructions for DigitalOcean](http://docs.deis.io/en/latest/installing_deis/digitalocean/#install-deis-platform) (you've just completed them&mdash;for the most part, and with some modification!) directly to [installing the Deis platform](http://docs.deis.io/en/latest/installing_deis/install-platform/#install-deis-platform).

However! Our $5 droplet does not have [the recommended](http://docs.deis.io/en/latest/installing_deis/system-requirements/#system-requirements) 4GB of RAM for running Deis.  But, we can sort that out by adding a swap file on each host.  We can even do this easily (i.e. via a global change) with the tools that CoreOS already provides.

If we don't do this, you will not be able to start Deis successfully on the cluster.

We do not have the recommended 40GB of disk space, so we shouldn't make our swap file much bigger than 4GB since we will easily run out of space.

This config worked for me reliably across repeated trials.  Of course, your mileage may vary, and there might be a "sweet spot" above or below that mark in your case.

## Preparing the Swap and Starting Deis Platform

Now that you've started your cluster, SSH into any one node as shown below, then download and register, and finally start the global swap file service unit with `fleetctl`.

{% highlight text %}
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh core@core-1.deis.example.com
$ curl -sSL -o swapon.service https://gist.githubusercontent.com/yebyen/efb65e358807f2ad4b52/raw/1e4d79292de162aafec29f63671e67e010fdf1ba/swapon.service
$ fleetctl register swapon.service
$ fleetctl start swapon.service
$ logout
{% endhighlight %}

 Go ahead and take a look at the contents of [swapon.service](https://gist.github.com/yebyen/efb65e358807f2ad4b52/#file-swapon-service) if you are curious about it.  This step will add a 4096Mb swap file to each of your new CoreOS nodes.

After that, you can finish the job by running `deisctl` to complete the [Deis platform installation](http://docs.deis.io/en/latest/installing_deis/install-platform/#install-deis-platform).

Next, configure Deis by using `deisctl` to upload your SSH private key to the cluster and set the domain name.

{% highlight text %}
$ export DEISCTL_TUNNEL=core-1.deis.example.com
$ deisctl config platform set sshPrivateKey=~/.ssh/deis
$ deisctl config platform set domain=example.com
$ deisctl install platform
$ deisctl start platform
{% endhighlight %}

Set the `domain` setting to your domain.

Even if you don't have a real DNS name, it's important for Deis to know what name it has, so decide on one here.  You can also [use xip.io](http://docs.deis.io/en/latest/managing_deis/configure-dns/#xip-io).

There's more detail in the links already provided, but if you do have DNS configured, you can use the DNS name of any node in the `DEISCTL_TUNNEL` above.

That's it. Those are all of the steps to start Deis platform.

Once it is running, `deisctl start platform` will take a while to complete. So, let's review some of the decisions we made, and why they were important.

# Configuration Decisions and Their Consequence

Why did we set `DEIS_NUM_INSTANCES` to `5`?

Including this setting will ensure etcd knows to expect five full member nodes, allowing two spares for fault tolerance.  This setting overrides the default setting of `3`.

Without this step, your cluster will only allow three etcd members to start.  Any remaining droplets that spun up would join as etcd *proxy nodes*. In that configuration, etcd (and therefore Deis) would not be reliably fault-tolerant, because proxy nodes do not contribute to the consensus-building capacity of the cluster.

You can experience strange failures if any one of a three node etcd cluster crashes. A cluster with five nodes can still shed two nodes and have an odd-numbered majority remaining to form a quorum. That odd-numbered-ness is important because of [the split-brain problem](https://en.wikipedia.org/wiki/Split-brain_(computing)).

When only a minority of etcd members are present and operating correctly, some Deis platform services will not function. So, set `DEIS_NUM_INSTANCES` to a value of `5`, and reap the benefits of increased fault tolerance.

The default value of `size` in the `variables.tf` provided by Deis is 8GB&mdash;that's about 16 times larger than the lowest possible value.  The default value for `instances` is three nodes. That's exactly two nodes too few for a Deis cluster to be reliably fault tolerant and resilient against node loss.

You could add more than five nodes...

You might think "more nodes, more redundancy", but for now, let's stick with five and keep it cheap and brief.  If you want high availability, using larger nodes would probably be more effective than adding more small nodes.

## Final Steps

If all has gone well, you now have a Deis cluster that can likely recover from the abrupt loss of any two nodes!

Put it through the paces.

Try powering off a droplet or two and see what happens.

Keep reading all of [the documentation](http://docs.deis.io/en/latest/) for some ideas about what to do with it.

You might start with `deis keys:add` and `deis create`, then follow that with `git push deis master`.  Also handy are `deis domains:add` and `deis ps:scale web=2`, or `web=4`, and so on.

Your cluster doesn't have a lot of capacity, but Deis is self-healing and distributed. Two nodes can come and go at the same time, and your app will keep working.

There are only two further caveats that must be highlighted.

## Caveats

### Apply Security Group Settings

Please check the current guidance on security group settings near the end of the platform-specific guide to [installing Deis on DigitalOcean](http://docs.deis.io/en/latest/installing_deis/digitalocean/#apply-security-group-settings).

Security group settings prevent internet traffic from reaching your cluster's Ceph, etcd, and generally any internal services that are otherwise meant to be private to the Deis platform.

Run this command :

```
$ for i in 1 2 3 4 5; do
  ssh core@deis-$i.example.com 'bash -s' < contrib/util/custom-firewall.sh
done
```

If you deviated from the guide and have more or less than five servers, adjust the `1 2 3 4 5` part to suit.

The private interfaces of your CoreOS nodes are now firewalled so that they only accept packets from each other.

### Disable User Registration

The security group setting does not protect against unknown persons registering as users or admins with the server.  You should make it a priority to use the `deis` client to [register the admin user](http://docs.deis.io/en/latest/using_deis/register-user/#register-with-a-controller).

The `deis register` command takes one argument: the URL of your Deis controller.

```
$ deis register http://deis.example.com
```

Substitute the domain you used earlier with `deisctl config platform set domain` here, instead of `example.com`.

You also need to take steps to keep others from anonymously pushing apps to your Deis cluster. So, read the [guidance on operational tasks](http://docs.deis.io/en/latest/managing_deis/operational_tasks/#disabling-user-registration), and disable user registration.

You can do that by running:

```
deisctl config controller set registrationMode="disabled"
```

## Monitoring

Monitoring would be a good next step for any serious deployment.

When you bump into the limits of RAM and disk, it will be helpful to know which node has reached the limit.

There are myriad solutions to the monitoring problem.

Deis and CoreOS do not directly expose any interfaces or primitives for external monitoring.

Monitoring is not considered any further in this post. However, should you wish to maintain a standard of availability, your next step after deploying apps should be to measure that availability and put some monitoring in place.

## Conclusion

We cut some corners, but: fault tolerance for $30 a month or less. I bet you never thought you could run your data center for so little.

Wait.

You probably can't. I hope nobody does! This guide was for demonstration purposes only. *Please don’t run your production systems like this!* :)

Also: do not forget to destroy your droplets when you are done with them. They cost a total of $0.035 per hour!
