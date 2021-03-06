---
layout: post
title: Incremental, Snapshot-based Backup with Rsync and SSH
---

[time-machine]: http://en.wikipedia.org/wiki/Time_Machine_(Mac_OS)

## Introduction

I've been looking for a no-frills, cross-platform, and incremental (à la [Time Machine][time-machine]) way to backup important files on my MacBook Pro. This backup solution also needed to work wherever I had an Internet connection, be it at home or elsewhere. After (too) much futzing, I settled on a process using SSH, rsync, an Ubuntu server, and a little bit of [Dropbox](http://dropbox.com). 

In short, every thirty minutes, my laptop connects (via SSH) to an Ubuntu server running at my home and copies (via rsync) an incremental version of my Dropbox directory.

What I like about this process is that it keeps my eggs in a couple different baskets: Dropbox syncs my backup directory to the cloud while my home server incrementally backs up the my important files to a local disk. This way, if Dropbox somehow loses my files, I should have recent backups available at home. If, on the other hand, my home backups are corrupted/destroyed, Dropbox should have the most recent copy of my backup directory. 

And if both Dropbox and my home server happen to fail at the same time? Well, then I'll go to a bar and have a drink (or three).

This post will describe how I cobbled together the necessary pieces and also include links to the original sources from which most of this information came. Although this process can be adapted to a variety of platforms, I've only tested it on my machine, which is a MacBook Pro running OS X 10.7. The server to which I'm syncing is an Ubuntu 11.04 Virtual Box virtual machine running under Windows 7. 

The steps to achieve this backup solution are as follows:

1.  [Set up SSH Server](#1)
1.  [Set up (Mostly) Passwordless SSH Authentication](#2)
1.  [Configure Dynamic DNS and Port Forwarding](#3)
1.  [Configure Rsync Shell Script](#4)
1.  [Create a Cronjob](#5)

In the [Conclusion](#conclusion) section, I'll go over an extra step I've taken due to what can probably be described as data-loss paranoia.

## <a name="1"></a> 1. Set Up SSH Server

The first part of this project was to set up the server that will store the laptop's backups. The backup server must run 24/7 with a consistent Internet connection. I used [Ubuntu 11.04](http://ubuntu.com) as the operating system of my server, but any OS that can run an SSH server and rsync should be sufficient. 

In my case, since I keep a Windows 7 PC running at home all day, I
decided to set up a Virtual Box virtual machine running Ubuntu 11.04.
Setting up a Linux-based virtual machine isn't a particularly difficult
process, but for those who haven't done it before, here's a [link to a
tutorial](http://www.psychocats.net/ubuntu/virtualbox).

Once running, the server must be configured to accept SSH connections. If you're using Ubuntu as your home server, you can install the OpenSSH server from the command line by typing "sudo apt-get install openssh-server". If you're using a different flavor of Linux, your installation process may vary.

To make sure the SSH server is running correctly, try logging into it from the command line of the laptop you are backing up. For example, on my MacBook, I entered the command "ssh user@192.168.1.x" where "user" is the username I created during the Ubuntu installation process and "192.168.1.x" is the local IP address of my Ubuntu server. If you can log in successfully, the SSH server is functioning properly. For more information on the OpenSSH, check out the [Ubuntu documentation](https://help.ubuntu.com/11.04/serverguide/C/openssh-server.html).

## <a name="2"></a> 2. Set Up (Mostly) Passwordless SSH Authentication

If you were able to log into the SSH server, you were probably prompted for a password. Since I didn't want to be asked for a password every time the backup script ran, I set up a key-based authentication. To do this, you'll need to create a pair of SSH keys. In OS X, a key pair can be generated by typing "ssh-keygen -t rsa" from the command line. If the above command works for you, press enter to place generated key files in the default location. Next, type in a password (why introduce another password? I'll explain later.) and press enter. 

Now that the SSH private/public keys have been generated, the public key must be installed on the Ubuntu server. From a text editor, open the public key on your laptop (default location: ~/.ssh/id\_rsa.pub) and copy the file's contents to the clipboard. Next, SSH into the Ubuntu server (ssh user@192.168.1.x), and open a text file in the following location on the server: ~/.ssh/authorized\_keys (example command: nano ~/.ssh/authorized\_keys). Finally, paste the contents of id\_rsa.pub into the authorized\_keys file. 

With your laptop's public key installed on the server, if you log out of the SSH session and attempt log back in, you will be asked for the SSH key's password (as opposed to the password for the Ubuntu user). For more detailed information on key-based authentication, [check out this post](http://linuxproblem.org/art_9.html).

It may seem like this key-based authentication just trades one password for another, but there is one more step required to smooth out the process while retaining much of the security that the password provides. 

Stepping back a moment: we could have simply hit enter when ssh-keygen asked for us to type a password. If we had done that, a password would not be required at all to log in using the key-based authentication.  However, that would mean if the laptop is stolen or lost, a nefarious (and unusually technically savvy) person could use the key to gain access to your server. With a password attached to the key pair, this hypothetical morally unscrupulous person would have to successfully break the password in order to use the key. 

Of course, I'd still prefer not to be hassled for a password each time the script runs. Thankfully, it's possible to get some of the security a password provides with less hassle by using a great shell script called [sshkeychain](http://sshkeychain.sourceforge.net/). Sshkeychain manages the ssh-agent on your laptop such that entering the SSH password once will allow all other scripts to run within that session. In other words, you can type in the password one time when the laptop boots, and the backup script won't ask for the password again until the laptop reboots. 

There are a number of ways to install sshkeychain, but I think the easiest way on OS X is to use the [homebrew](http://mxcl.github.com/homebrew/) project. If you haven't heard of it, homebrew is a package manager for Mac OS X. 

To install homebrew, follow the directions [here]
(https://github.com/mxcl/homebrew/wiki/installation). 
 
To install sshkeychain with homebrew, type "brew install keychain" on the command line. [This page](http://www.cyberciti.biz/faq/ssh-passwordless-login-with-keychain-for-scripts/) has step-by-step details on how to set sshkeychain up, but I'll post the short version of it here. 

In the ~/.profile file on your laptop (assuming it is running Mac OS X or Linux), paste the following lines and the save the file:

{% highlight bash %}

    #SSH keychain -- passwordless login
    /usr/local/bin/keychain $HOME/.ssh/id_rsa
    source $HOME/.keychain/$HOSTNAME-sh

{% endhighlight %}

With the updated ~/.profile, the next time you open a terminal window, you'll be prompted for the SSH password. After that, you won't be asked again until the laptop is rebooted.

To recap what has happened thus far: we've created and booted up up an always-on Ubuntu server, installed an OpenSSH server, set up key-based authentication for SSH, and installed sshkeychain on the laptop.

## <a name="3"></a> 3. Configure Dynamic DNS and Port Forwarding

A major requirement of this project is to be able to do backups over the
web, even when the laptop and server aren't on the same subnet (i.e.
when I'm not at home). Therefore, the backup script will need to have an
address with which it can remotely access the server. One solution is to
use the current public IP address of router (as opposed to the local
192.168.1.x address). However, my public IP address is dynamically
assigned, and I can't count on the address staying the same for any
length of time. Fortunately, my wireless router has a feature called
"Dynamic DNS", which uses a third party service to provide a consistent
hostname associated with whatever IP address my router picks up from my
ISP. To configure Dynamic DNS, I opened a free account at one such
third-party service called DynDNS (there are others) and configured my
router to use that account. [Here are more specific
instructions](http://www.cctvcamerapros.com/linksys-router-dynamic-dns-s/129.htm) on how
to set up DynDNS with a Linksys router. 

Once your DynDNS account is created your router is configured, your dyndns.org hostname will be associated with your public IP address (e.g. my\_name.dyndns.org). If your router does not have Dynamic DNS, your only option may be to use your public IP address and alter your script when your IP address renews.

There's one more step required to facilitate SSH connectivity between
your laptop and home server: before you will be able to log in to SSH
via my\_name.dyndns.org (or your public IP address), you'll first have
to set up port forwarding on your router so that the SSH port (default:
22) points to the local IP address of your Ubuntu server (e.g.
192.168.1.x). I had a hard time finding a good tutorial on port
forwarding, but [this website](http://portforward.com) may be helpful if you're having trouble setting that up.

Once port forwarding is working properly, you can confirm your SSH connectivity by logging in to your SSH server via your DynDNS hostname. For example, type "ssh user@my\_name.dyndns.org" from your laptop's command line. If you're dropped into the command line of your server, you're ready to put together the backup script.

## <a name="4"></a> 4. Configure Rsync Shell Script

When I got to this point, I had to do a little bit of thinking about which files were important enough to backup so frequently. Since I already use Dropbox to sync my important files to multiple computers (and the cloud), I figured that the Dropbox directory is a good choice to incrementally backed up. 

One side note: I've made heavy use of [symbolic links](http://lifehacker.com/5154698/sync-files-and-folders-outside-your-my-dropbox-folder) to bring files from around my filesystem into my Dropbox directory. You might also find it handy to link important files in disparate parts of your file system to one common directory. Examples of files I link to my Dropbox folder are various configuration files (.vimrc, .profile) and my programming workspaces. 

Having picked out the files I want to back up, I needed to find, modify, or create a shell script that does the following: 1) connects to my Ubuntu server via SSH, 2) copies the local Dropbox files to a timestamped directory on the server, 3) leverages hard links to avoid using up disk space on duplicate files, and 4) makes sure there is only one instance of the script running at a time. 

A Google search proved fruitful. A helpful blogger posted [this great tutorial](http://blog.interlinked.org/tutorials/rsync_time_machine.html) on how to use rsync to do exactly what the previous paragraph describes. For those unfamiliar with the tool, rsync is a handy utility that can copy files across SSH. It also has many features that this backup solution requires, including the ability to create hard links.

The script linked to above copies the files in a local directory to a remote location over SSH. While the backup session is in progress, files are placed in a directory called "incomplete-back-YYYY-MM-DDTHH\_MM\_SS" (where YYYY is the year, MM is the month, DD is the day, HH is the hour, MM is the minute, and SS is the seconds). When the backup is complete, the directory is renamed to back-YYYY-MM-DDTHH\_MM\_SS. Finally, the directory is symlink'd to "current" so you'll always know which backup is the most recent.

One of the best features of this script is that, if a file already exists in the "current" directory, it won't be recopied. Instead, a hard link is established. Like a symbolic link, a hard link is just a reference to the original file. A hard link differs from a symbolic link, however, in that the original file isn't deleted when the original file is deleted. The original file is deleted only when all hard links to it have been removed.

The script from the above blog post is great, but I also wanted to add a feature that makes sure only one instance of the script runs at a time. That way, if I'm backing up a lot of data or just have a slow connection, multiple instances of the script won't be running on top of each other. Once again, a helpful person on the Internet came through with this [locking mechanism](http://stackoverflow.com/questions/185451/quick-and-dirty-way-to-ensure-only-one-instance-of-a-shell-script-is-running-at-a).

The main gist of the locking mechanism is that it creates a file when the backup script starts, and every new instance of the backup script checks to see if this file exists before it continues. If the file exists, the script won't go through the backup process. If the file does not exist, the backup proceeds.

Combining the two scripts, I ended up with this shell script to perform my backups:

{% highlight bash %}

    #!/bin/sh

    LOCAL=${HOME}/Dropbox/
    REMOTE=/media/backup_drive/
    HOST=my_name.dyndns.org

    #http://stackoverflow.com/questions/185451/quick-and-dirty-way-to-ensure-only-one-instance-of-a-shell-script-is-running-at-a
    #http://blog.interlinked.org/tutorials/rsync_time_machine.html
    LOCKFILE=${HOME}/temp/lock.txt
    if [ -e ${LOCKFILE} ] && kill -0 `cat ${LOCKFILE}`; then
        echo "already running"
        exit
    fi

    # make sure the lockfile is removed when we exit and then claim it
    trap "rm -f ${LOCKFILE}; exit" INT TERM EXIT
    echo $$ > ${LOCKFILE}

    date=`date "+%Y-%m-%dT%H_%M_%S"`

    source ${HOME}/.keychain/${HOSTNAME}-sh 

    rsync -azP \
      --log-file=${HOME}/log/rsync_snapshots.log \
      --delete \
      --link-dest=../current \
      --copy-links \
      ${LOCAL} ${HOST}:${REMOTE}/incomplete_back-${date} \
      && ssh ${HOST} \
      "mv ${REMOTE}/incomplete_back-$date $REMOTE/back-${date} \
      && rm -f ${REMOTE}/current \
      && ln -s back-${date} ${REMOTE}/current"

    sleep 1
    rm -f ${LOCKFILE}

{% endhighlight %}

Paste the above script into a text file and save it to your laptop (I saved the script to ~/Scripts/rsync\_snapshots.sh). You'll have to change the LOCAL, REMOTE, and HOST variables to match your configuration. The LOCAL variable is the directory you wish to backup on your laptop. The REMOTE directory is the full path to the directory on the Ubuntu server in which the backups will be stored. The HOST variable stores your dynamic DNS hostname or the public IP of your router.

Next, we need to make the script executable. From the command line, go to the directory of the script (e.g cd ~/Scripts) and type "chmod +x rsync\_snapshots.sh". 

Now that the script is executable on your laptop, you can test it out. From the script's directory, type "./rsync\_snapshots.sh". 

Hopefully, the script will run and you'll see output similar to this:

{% highlight bash %}

    building file list ... 
    9587 files to consider
    created directory /media/backup_drive/incomplete_back-2011-11-20T19_05_05

{% endhighlight %}

This output will be followed by a growing list of files that have been copied over to your server. You can go check out the remote location on the server (e.g. /media/backup\_drive/) and see if the files were successfully copied over.

## <a name="5"></a> 5. Create a Cronjob

Finally, we need to schedule the backup script to run every thirty minutes. To do this, we'll set up a cronjob.

_Note to Mac OS X users:_ There's a weird problem when editing crontab
files using vim. For whatever reason, the saved crontab file never
overwrites the original. This is apparently a known problem. [Here is a
post](http://tim.theenchanter.com/2008_07_01_archive.html) with a snippet that can be put in your .vimrc file to alleviate that problem. If you don't wish to use vim to edit the crontab, you can modify the EDITOR environment variable to something else (e.g. type "export EDITOR=nano" on the command line).

To create a cronjob, type "crontab -e" on the command line and paste in the following lines:

{% highlight bash %}

    MAILTO=""                                                                       
    */30 * * * * /path/to/script/rsync_snapshots.sh

{% endhighlight %}

You'll need to change the /Path/to/script/rsync\_snapshots.sh to the full path of your rsync backup script. Note that the first line (MAILTO="") is just a quick way to avoid rsync from sending your user account mail in the case of an error. If you'd like to receive mail regarding errors, simply delete that line. After you've made your changes, save the file and exit the editor.

Now, every thirty minutes (e.g. 12:00 and 12:30), the script should run. You should notice the backup directory on your server adding directories each time the script runs. For reference, in my tests, when the backup runs and no files have changed, the "empty" backup directory takes up approximately 12 megabytes of disk space. I think this overhead is worth the piece of mind the incremental backup provides.

##Conclusion {#conclusion}

Assuming the above steps worked out for you, your backup directory on your laptop should be (incrementally) copied over to the server every thirty minutes. If an important file is corrupted or deleted from your laptop, you can go to the backup directory on the server and pull down the last good version of the file.

Additionally, I've gone the extra step of copying my incremental backups to another disk twice daily. That way, if my backup drive ever dies, I'll still have copies of my files on the backup-backup drive.

Here's what that script looks like:

{% highlight bash %}

    #!/bin/sh

    LOCKFILE=${HOME}/temp/lock.txt
    if [ -e ${LOCKFILE} ] && kill -0 `cat ${LOCKFILE}`; then
        echo "already running"
        exit
    fi

    # make sure the lockfile is removed when we exit and then claim it
    trap "rm -f ${LOCKFILE}; exit" INT TERM EXIT
    echo $$ > ${LOCKFILE}

    rsync -azPr --hard-links --links --log-file=/var/log/rsync_backup.log --exclude "lost+found" --delete /media/backup_drive/ /media/backup_drive2/

    sleep 1
    rm -f ${LOCKFILE}

    exit 0

{% endhighlight %}

After placing the above script on my Ubuntu server and making it executable, I set up a new cronjob that ran the script twice a day:

{% highlight bash %}

    00 10,23 * * * /path/to/script/rsync_backup.sh

{% endhighlight %}

More in information about setting up cronjobs can be found [here](http://unixgeeks.org/security/newbie/unix/cron-1.html).
 
There are many other ways this backup solution can be improved. For example, it'd be nice to have another script on the server that deletes backup snapshots when they reach a certain age or when the remote backup directory takes up a certain amount of space. Also, a mechanism that reports errors would be helpful so that backups aren't failing silently. 

I hope this post was helpful, and please [let me know](mailto:matt@mtmckenna.com) if something in incorrect or could be improved.

