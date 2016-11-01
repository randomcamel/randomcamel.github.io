

Chef is a mighty set of tools that can automate and verify your infrastructure, but its full power is a lot to digest. Most of us learn new tools by trying them out for an hour or two, and that's usually how we make a decision. Today I want to show how you can get real value out of Chef real fast, maybe on a Friday afternoon with a couple of hours set aside for experimenting, by replacing a shell script with a list of Chef declarations. Then we can run that list using `chef-apply`, which executes a Chef runlist in a single file.

### A Brief Note

Before we get started, it's worth mentioning that even though I'm going to pretend Chef is a scripting language, a Chef runlist **describes** the desired final state of a system, rather than providing a list of actions to execute. In other words, each Chef resource triggers code like this:

if system state already matches what the resource describes
then
don't do anything
else
perform whatever actions are needed to get the system into the correct state

So if I tell Chef I want a file present, and specify its contents:

[ruby]
file "/tmp/foo" do
content "This will be the file's content."
end
[/ruby]

Chef will check to see if the file is there with that content, and only update it (or create it) if needed.

For the times when life doesn't fit that model, we have the [`execute`](https://docs.chef.io/resource\_execute.html) resource to run some shell code. Even then, we can add a `not\_if` or `only\_if` condition to prevent it from executing on every Chef run.

### You're still talking. You told me this would be useful.

Enough! It's Friday afternoon, I want to try out Chef, and I want to get something done!

At home I run a music server called `squeezeboxserver`, and because later versions removed my favorite features, I use an older version that won't run on anything later than Ubuntu 11. I have an Ubuntu 10 virtual machine, using [Vagrant](http://www.vagrantup.com) to manage the [VirtualBox](https://www.virtualbox.org) VM. `squeezeboxserver` scans my music library and stores the file and playlist data in MySQL.

`squeezeboxserver` is very particular about its dependencies and configuration. I used its build scripts to build a Debian package, but it doesn't install any dependencies. It needs MySQL Server installed, but by default it runs its own `mysqld` rather than using a system-global `mysqld`. I originally wrote a Bash script to set the machine up:

[bash]
rm /etc/localtime
ln -s /usr/share/zoneinfo/America/Los\_Angeles /etc/localtime
apt-get update
export DEBIAN\_FRONTEND=noninteractive
apt-get -y install mysql-server-5.1 libmysqlclient16-dev mysql-client-5.1
dpkg -i /home/chris/squeezebox/squeezeboxserver\_7.5.6~32834\_all.deb
tar -C / -xf /home/chris/backups/sbserver_prefs.tar
/etc/init.d/squeezeboxserver restart
[/bash]

This mostly worked, but it's surprisingly brittle for an 8-line shell script. The VM still required a lot of manual tweaks, and it was actually fragile enough that I avoided taking the host server down for maintenance because I dreaded trying to bring the VM back up. It also wasn't reliable if any of those commands encountered an error-this happened far more often than I would have expected-and it didn't clearly communicate what it was doing. Deleting and re-creating the VM was similarly hazardous.

With the most recent maintenance of my host server, I decided I'd had enough, and I converted it using `chef-apply`. `chef-apply` is a tool that ships with Chef, and it basically lets you pretend Chef is a scripting language: no recipes, no directory structure, just a single file of Chef resources. You can even shebang a `chef-apply` script as you would a shell script:

[shell]#!/usr/bin/env chef-apply[/shell]

I went through my original Bash script line by line to convert it to Chef.

[bash]
rm /etc/localtime
ln -s /usr/share/zoneinfo/America/Los\_Angeles /etc/localtime
[/bash]

There are a few ways to set your machine's timezone; this just happens to be the one I'm familiar with. Easy enough, Chef has a [`link`](https://docs.chef.io/resource\_link.html) resource.

[ruby]
link "/etc/localtime" do
to "/usr/share/zoneinfo/America/Los\_Angeles"
end
[/ruby]

Here I'm telling Chef "the correct state of the server has this symlink"; Chef will check to see if the symlink already exists, and do nothing if it's already there and pointing to the right file.

[shell]
export DEBIAN\_FRONTEND=noninteractive
apt-get -y install mysql-server-5.1 libmysqlclient16-dev mysql-client-5.1
[/shell]

Chef's [`package`](https://docs.chef.io/resource\_package.html) resource has us covered.

[ruby]
package mysql-server-5.1
package libmysqlclient16-dev
package mysql-client-5.1
[/ruby]

Or, if you're comfortable with Ruby and you don't like the repetition, you can do it with a loop which does the exact same thing.

[ruby]
["mysql-server-5.1", "libmysqlclient16-dev", "mysql-client-5.1"].each do |pkg\_name|
package pkg\_name
end
[/ruby]

But wait-installing `mysql-server-5.1` starts the system `mysqld`. I could stop it (`service mysql stop`) but then it will restart whenever I reboot the VM. I'm a software engineer, not a sysadmin, so I'm far too lazy to figure out how to permanently disable the system `mysqld`. Chef's [`service`](https://docs.chef.io/resource\_service.html) resource has an action for this.

[code]
service "mysql" do
action :disable
end
[/code]

Now `mysqld` won't start up on system boot.

So much for the prep work. Now to install and configure `squeezeboxserver` itself.

[code]dpkg -i /home/chris/squeezebox/squeezeboxserver\_7.5.6~32834\_all.deb[/code]

Here we encounter our first diversion, which merits some background. Chef **resources** are backed by **providers**, where the actual heavy lifting of a Chef resource happens. When we wrote `package mysql-client-5.1` above, we didn't have to concern ourselves with what operating system we were on: Chef will figure it out and call the correct provider for your system. On a RedHat-based system, this is the `yum` provider, and on Debian-based systems, of course, the `package` resource resolves to the `apt` provider. As a result, to install from a `.deb` file, I had to use the [`dpkg\_package`](https://docs.chef.io/resource\_dpkg\_package.html) resource, which explicitly uses the `dpkg` provider.

[code]
dpkg\_package "squeezeboxserver\_7.5.6" do
source "/home/chris/squeezebox/squeezeboxserver\_7.5.6~32834\_all.deb"
end
[/code]

The `.deb` file leaves the server running, but I've been running `squeezeboxserver` for a long time, and I have a `.tar` of prefs files I want to use.

[code]
tar -C / -xf /home/chris/backups/sbserver_prefs.tar
[/code]

This is not so bad, and in fact there is no core `tar` resource, so I just move it verbatim into Chef with the `execute` resource. (There's a `tar` cookbook, but it's Friday afternoon and I don't want anything complicated.) Normally we use Chef resources to describe our desired final state; sometimes that's not enough, so we have `execute`, which essentially shells out a command.

[code]
execute "tar -C / -xf /home/chris/backups/squeezeboxserver.tar"
[/code]

Then,
Now restart the server, and tell it to scan my music collection to populate the MySQL database.

[code]
/etc/init.d/squeezeboxserver restart
[/code]

Easy enough. Tell Chef to disable, then re-enable the service.

[code]
service "squeezeboxserver" do
action :restart
end
[/code]

And...that's it.

Since I was on a roll, I decided to add some cronjobs, since Chef makes managing cronjobs trivial. It's such trouble to modify crontabs safely in Bash that I had never bothered trying.

[ruby]
cron "backup\_config" do
minute "0"
hour "0"
user "vagrant"
command "tar -C / -cf /home/chris/backups/sbserver_prefs.tar /var/lib/squeezeboxserver/prefs"
end
[/ruby]

A couple more like that, and now I have up-to-date backups of the config, the playlists, and the MySQL data. Boom. I deleted and re-created the VM a few times-once or twice, it was just for fun. Chef re-built it identically every time. If something goes wrong, wonder of wonders, it stops and tells me. It's true, the `chef-apply` script has a few more lines in it. It's also more reliable and has more features than the original.

The deep truth about Chef is that it's not doing anything clever, which is a real relief to those of us who think "clever" means "hard to debug." It's simply doing all the checking ("is this file present and with the right contents?") and cross-platform implementation ("use apt-get on Debian, Yum on RHEL, pkg\_add/pkgng on FreeBSD...") that we would find tedious and error-prone. It does all this always and only in the order we tell it to.

`chef-apply` is sort of "Chef Lite": Chef's quick-hack, Get Stuff Done tool. You definitely don't want to run your whole infrastructure on it. It runs just a single file, with no cookbooks, file templates, roles, users, orgs, or any of the other Chef features you'd want in your automation. When you're ready for the full power of Chef, you have a foundation of cross-platform code you can copy into your recipes.

I've included the full `chef-apply` script below. Happy hacking!


link "/etc/localtime" do
  to "/usr/share/zoneinfo/America/Los\_Angeles"
end
package mysql-server-5.1
package libmysqlclient16-dev
package mysql-client-5.1
service "mysql" do
  action :disable
end
dpkg\_package "squeezeboxserver\_7.5.6" do
  source "/home/chris/squeezebox/#{DEB}"
end
\# o/~ meet the new server / same as the old server o/~
execute "tar -C / -xf /home/chris/backups/sbserver\_prefs.tar"
service "squeezeboxserver" do
  action :restart
end
\# ---- end app setup ----
cron "nightly\_rescan" do
  minute "30"
  hour "0"
  user "vagrant"
  command "/usr/sbin/squeezeboxserver-scanner --rescan"
end
cron "backup\_config" do
  minute "0"
  hour "0"
  user "vagrant"
  command "tar -C / -cf /home/chris/backups/sbserver\_prefs.tar /var/lib/squeezeboxserver/prefs"
end
cron "backup\_mysql" do
  minute "5"
  hour "0"
  user "vagrant"
  command "sudo mysqldump -S #{MYSQL\_SOCK} slimserver | gzip -c > /home/chris/backups/sbserver_sql.tgz"
end
cron "backup\_playlists" do
  minute "10"
  hour "0"
  user "vagrant"
  command "tar -cf /home/chris/backups/playlists.tar /home/chris/music/playlists"
end

