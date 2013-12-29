---
title: Linux + Github
layout: post
guid: urn:uuid:7c3861ff-2cea-4e5f-bd0a-9b06e0c39d28
tags: daily
comments: no
---

**Setup GitHub In Your Linux**

1.Install Git (Ubuntu)
> sudo apt-get install git-core

2.Set SSH Key

Backup old SSH Keys
Create new SSH Keys
> ssh-keygen -C -t rsa "email\_address"

You'll be asked to provide a file name and a passphrase. The passphrase could be empty.
Copy those keys--"your\_file\_name" and "your\_file\_name.pub" into ~/.ssh/ directory. 
Rename them as "id\_rsa" and "id\_rsa.pub".

3.Add SSH Key to your GitHub account

Login GitHub, go into "Account Settings", click "SSH Keys", then "Add SSH Key".
Enter a title, copy the content of "id\_rsa.pub" to fill the key.

4.Test SSH
> ssh -v git@github.com

5.Setup Git Personal Information

These information will be used to connect to your GitHub account.
> git config --global user.name "your\_github\_username"
> git config --global user.email "your\_github\_email"

---- 

**Usage**

1.Clone an existing project
> git clone git@github.com:your\_github\_username/target\_project\_name.git

2.To be continued
