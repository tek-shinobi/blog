---
title: "Github:managing Multiple Accounts: SSH configuration"
date: 2023-11-06T19:53:20+02:00
draft: false 
categories: ["github"]
tags: ["github"]
---

Notes for SSH key configuration with multiple Github accounts.

I have multiple github accounts. Say one for work and one that is personal.
> NOTE: This logic can be extended to more than two accounts.


The setup can be done in 5 easy steps:
## Steps:
- [Step 1](#step-1) : Create SSH keys for all accounts
- [Step 2](#step-2) : Add SSH keys to SSH Agent
- [Step 3](#step-3) : Add SSH public key to the Github
- [Step 4](#step-4) : Create a Config File and Make Host Entries
- [Step 5](#step-5) : Clone GitHub repositories using different accounts OR modify remote url for existing repos

<br>

## Step 1
### Create SSH keys for all accounts
First make sure your current directory is your **.ssh** folder.
```sh
     $ cd ~/.ssh
```
Syntax for generating unique ssh key for each account is:
```sh
     ssh-keygen -t ed25519 -C "your-work-email-address@gmail.com" -f "github-work"
     ssh-keygen -t ed25519 -C "your-personal-email-address@gmail.com" -f "github-personal"
```
here,

**-C** stands for comment to help identify your ssh key

**-f** stands for the file name where your ssh key get saved

Notice here the two emails are the github account emails for each of my github accounts.

After entering the command the terminal will ask for passphrase, leave it empty and proceed.

> Now after adding keys , in your .ssh folder, a public key and a private will get generated.
>The public key will have an extention __.pub__ and private key will be there without any extention both having same name which you have passed after __-f__ option in the above command. (in my case __github-work__ and __github-personal__)

<br>

## Step 2
### Add SSH keys to SSH Agent
Now we have the keys but it cannot be used until we add them to the SSH Agent.
```sh
     ssh-add -K ~/.ssh/github-work
     ssh-add -K ~/.ssh/github-personal
```

You can read more about adding keys to SSH Agent [here.](https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

<br>

## Step 3
### Add SSH public key to the Github
For the next step we need to add our public key (that we have generated in our previous step) and add it to corresponding github accounts.

For doing this we need to:

__1. Copy the public key__

     We can copy the public key by opening the github-work.pub file in nano and then copying the content of it.
```sh
     nano ~/.ssh/github-work.pub
     nano ~/.ssh/github-personal.pub
```



__2. Paste the public key on Github__

* Sign in to Github Account
* Goto **Settings** > **SSH and GPG keys** > **New SSH Key**
* Paste your copied public key and give it a Title of your choice.

<br>

## Step 4
### Create a Config File and Make Host Entries

The **~/.ssh/config** file allows us specify many config options for SSH.

If **config** file not already exists then create one (make sure you are in **~/.ssh** directory)

```sh
     touch config
```

Paste the following in the `config` file:
```shell
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-personal

Host github.com-github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-work
```
The first one is the default github account.

Now you need to ensure that you are successfully able to ssh with either account creds.

checking the default account:
```shell
ssh git@github.com
```
this will check the ssh access to default account. It should say something like this if its successful:
> Hi <user_name> You've successfully authenticated, but GitHub does not provide shell access.
>Connection to github.com closed.

checking the `github-work` account:
```shell
ssh git@github.com-github-work
```

this will check the ssh access to github-work account. It should say something like this if its successful:
> Hi <user_name> You've successfully authenticated, but GitHub does not provide shell access.
>Connection to github.com closed.

## Step 5
### Clone a repo
The default account is easy. you can copy paste the ssh git clone link that is provided by the github repo you want to clone.
```shell
git clone git@github.com:kubernetes/kubernetes.git
```

To clone a repo using the work account.. do it like so:
```shell
git clone git@github.com-github-work:kubernetes/kubernetes.git
```
where `github.com-github-work` is the label that is used in `~/.ssh/config` file for the work github account

### modify existing repo
In case you have an existing repo, if that repo is related to your default account, nothing needs to be done. 

For the repos related to the work account, you will either need to reset remote origin or set one (if not already set):
goto the respective repo folder. and the cd to `.git` directory. Now do `nano config` and make sure that `url` looks like this:
```shell
[remote "origin"]
	url = github.com-github-work:kubernetes/kubernetes
	fetch = +refs/heads/*:refs/remotes/origin/*
```

Here, `kubernetes/kubernetes`  is in format `username/reponame` format. You will of course have your own repo name here, but this part will be the same `github.com-github-work`

You will need to do this to all of your existing work repos. 
