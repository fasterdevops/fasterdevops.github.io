---
layout: post
title: Setting up my first saltstack
published: true
---
# What is Salt?
Saltstack is a power systems orchestration tool used by sysadmins, developers,and large corporations based on the idea of "Infrastructure through code." 
![_config.yml]({{ site.baseurl }}/images/saltstack.jpg)

The official [Saltstack](http://saltstack.com) repo can be found [here](http://github.com/saltstack/salt) 

Salt is powered by python and works on any Unix-like operating system, OS X, and windows alike. It's modules are written in Python. There are many features available ranging from simple tasks to more advanced ones.

Let's get started

### Installing your Salt master-

```
curl -L https://bootstrap.saltstack.com -o install_salt.sh
sudo sh install_salt.sh -P -M
```

### Installing your Salt minion-

```
curl -L https://bootstrap.saltstack.com -o install_salt.sh
sudo sh install_salt.sh -P 
```
For this example setup, I'll be setting up 1 salt master and 6 minions on CentOS instances from [DigitalOcean](http://digitalocean.com) for multiple reasons. Mostly because they support Salt-cloud. I'll re-touch on this later.

After running the appropriate commands, let's add our master to each minion. Here, my master is riley.science. On each minion, run the following-

```
echo "master: riley.science" >> /etc/salt/minion 
```

This adds your master to each minion. Now, let's start the salt minions with the following-

```
systemctl enable salt-minion
systemctl start salt-minion

```

Start your salt master-

```
systemctl enable salt-minion
systemctl start salt-minion
```

Assuming your minions can ping the master,now we should be able to run the following to view and accept the keys from the master. From here,you will not need to ssh directly into the minions(SSH'ing into servers is soo 2016) again. Let's view the keys asking for a connection,and accept them-

```
[root@srv3 ~]# salt-key -L
Accepted Keys:
git.riley.science
srv.fasterdevops.com
srv4.riley.science
srv5.riley.science
srv6.riley.science
srv7.riley.science
srv8.riley.science
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```

Here, I have already accepted my minions. Run salt-key -A to accept the keys that will show under Unaccepted Keys.

That's it! You now have a salt master and minions connected to it. Test the connection as follows-

```
[root@srv3 ~]# salt '*' test.ping
srv4.riley.science:
    True
srv5.riley.science:
    True
git.riley.science:
    True
srv6.riley.science:
    True
srv8.riley.science:
    True
srv7.riley.science:
    True
```
Salt supports using wildcards when specifying the host, so with the above ping command if I wanted to ping every minion that started with  the hostname srv, I would replace the * with just srv*.

You can find some of my own salt formulas to help you get started on my [github](http://github.com/sadminriley). 

Other commands-

```
salt '*' disk.usage

salt '*' network.interfaces 

salt '*' cmd.run 'yum -y install whois'
```

If you were to clone my formulas, and put them into /srv/salt , you could run the following to apply the state 'require' to install the specified packages-

```
salt '*' state.apply require
```


You can learn more about advanced usage over at [https://docs.saltstack.com/](https://docs.saltstack.com/).

I hope this was helpful!