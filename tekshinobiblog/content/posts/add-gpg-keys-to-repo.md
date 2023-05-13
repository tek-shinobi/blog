---
title: "Add Gpg Keys to Repo"
date: 2023-05-13T22:53:11+03:00
draft: false 
---

# Add GPG keys to github repo and local 

The process involves a set of simple steps:
1. Create GPG keys from commandline terminal
2. Export the created keys using `armor` into a text blob.
3. Take the blob and head over to `github account` -> `Settings` -> `SSH and GPG keys` and add it as GPG key
4. The most important point: `cd` to you local repo directory and configure the key there. This step will need to be done for every local repo.

So, lets get started:
In terminal:
```
gpg --full-generate-key
```

It will guid you through creating the key.

Now you can view the key like this:
```
gpg --list-secret-keys --keyid-format=long
```

you will see something like this:
```
$ gpg --list-secret-keys --keyid-format=long
/Users/hubot/.gnupg/secring.gpg
------------------------------------
sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
uid                          Hubot <hubot@example.com>
ssb   4096R/4BB6D45482678BE3 2016-03-10
```

that `3AA5C34371567BD2` in `sec` is your short form key-id of sorts that you will use in local-config of this GPG key

Now get the text blob to add in github like so:
```
gpg --armor --export 3AA5C34371567BD2
```

Paste the blob into your github account per step 3 (care that in the paste, you don't add extra empty line at the bottom)

Now, comes the confusing step that is sometimes not explained well in github manuals. At least, here's how I managed to do. There has to be a better way, but for now, following is fine with me.

##### The GPG key needs to be added manually to every local repo in your computer (or remote-server).

First `cd` to the root of your git repo that you want to sign, commit and push. 

Then
```
git config --local commit.gpgsign true
```
and sfter that:
```
git config --local user.signingkey 3AA5C34371567BD2
```

that's it. 

```
git commit -S -m "test commit"
```

will do the commit.

If you get that annoying message when trying to push that `github removed using password ... blah .. blah`, add a `Personal Access Token` to your github account and setup your `.netrc` file to automate this login process
