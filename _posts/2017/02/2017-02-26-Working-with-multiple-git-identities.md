---
layout: post
title:  "Working with multiple Git identities"
date:   2017-02-26 00:00:00 +0000
categories: "ALM"
tags:
    - git
    - ssh
    - vsts
---

I recently rebuilt my Mac Book Pro - appears OSX also benefits from a spring
clean every now and then. When attempting to clone a Git repo hosted on Visual Studio
Team Services (VSTS) I was hitting an authentication error.

```bash
$ git clone ssh://account@account.visualstudio.com:22/DefaultCollection/TeamProject/_git/repo
Cloning into 'repo'...

Your Git command did not succeed.
Details:
	Public key authentication failed..

fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

> NOTE: All references to `account` should be replaced with your VSTS account name.

## Git Authentication

Let's backtrack slightly. On my Windows machine I use the [Git Credential
Manager for
Windows](https://github.com/Microsoft/Git-Credential-Manager-for-Windows) which
can be installed as part of the [Git for Windows
installer](https://git-scm.com/download/win). This authenticates with VSTS and
creates a PAT (Personal Access Token) for accessing. There is an equivalent
[credential manager for Mac and
Linux](https://github.com/Microsoft/Git-Credential-Manager-for-Mac-and-Linux)
but this requires installing an additional package so I decided to use SSH.

## SSH Keys

VSTS has some good [step by step
instructions](https://www.visualstudio.com/en-us/docs/git/use-ssh-keys-to-authenticate)
on using SSH to authenticate.

I already had an SSH key for GitHub (`~/.ssh/github_rsa`) - I think this was
created by the [GitHub Desktop app](https://desktop.github.com). Rather than
create a new SSH key in the default location `~/.ssh/id_rsa` I opted to create
one mirroring the naming of the GitHub one, `~/.ssh/vsts_rsa`. As this is not
the default identity we need to make [SSH aware of
it](https://www.visualstudio.com/en-us/docs/git/use-ssh-keys-to-authenticate#how-can-i-use-a-non-default-key-location-ie-not-sshidrsa-and-sshidrsapub-)
using `ssh-add`. You can verify that the key is registered using `ssh-add -l`.

At this point I expected to clone the repo and begin work, but not so fast. This
was the point I received the error at the start of this post.

Following the information in the VSTS docs, I tested the [SSH connection without
a Git
command](https://www.visualstudio.com/en-us/docs/git/use-ssh-keys-to-authenticate#how-can-i-test-my-ssh-connection-without-running-a-git-command).

```bash
ssh -T account@account.visualstudio.com
```

That test worked successfully, so Git was not using the SSH key.

The answer, to create an ssh config. After reading [this
post](http://nerderati.com/2011/03/17/simplify-your-life-with-an-ssh-config-file/)
I ended up with a config like below:

```
Host github.com
    User git
    HostName github.com
    IdentityFile ~/.ssh/github_rsa
Host account.visualstudio.com
    User account
    HostName account.visualstudio.com
    IdentityFile ~/.ssh/vsts_rsa
```

I use multiple VSTS accounts and this set-up allows for each account to be easily added.

## Different user name and email

With authentication handled there was one small item left to deal with, the name
and email associated with a commit. While not a big problem between my personal
VSTS account and GitHub I did want different values for work accounts. After
quite a bit of reading (this
[post](https://orrsella.com/2013/08/10/git-using-different-user-emails-for-different-repositories/)
dives into a lot of detail on options), I opted to use the `useConfigOnly` option
and force name and email to be specified per repository.

```bash
git config --global user.useConfigOnly true
```