---
title: "Linux:Iptables Cheatsheet"
date: 2023-10-17T13:23:19+03:00
draft: false 
categories: ["Linux"]
tags: ["iptables"]
---

5 types of tables: 
1. Filter
1. NAT
1. Mangle
1. Raw
1. Security

Last two tables are not very frequently used. Default table is Filter table.

List table:
```shell
sudo iptables -L -n -v
```
L: list
n: numeric format
v: verbose

since no table was mentioned in the above command, it will list all contents of default table, Filter.

below will display contents of mangle table:
```shell
sudo iptables -t mangle -L -n -v
```

Syntax of iptables is case sensitive.

```shell
iptables -t[table] -OPTIONS[CHAIN] [matching component] [Action component]
```

options are used to modify the chain. Specific chains are attached to specific tables. 

Options are of these types:
1. Append
1. Delete
1. Insert
1. Replace
1. Zero Counters
1. List
1. Policy
1. Rename
1. Flush
1. New User defined chain
1. Delate chain

To block a particular website: (example.com)

```shell
sudo iptables -A INPUT -s example.com -j DROP
```

this will block `example.com` website. You can as well use `REJECT` instead of `DROP`

first list, to note the rule number
to unblock:
```shell
sudo iptables -L -n -v
```
find at what location is the above DROP rule. Let's say that its the very first rule in the chain

Then:
```shell
sudo iptables -D INPUT 1
```


You can change the POLICY to default block all websites except those explicitly allowed:
```shell
sudo iptables -P INPUT DROP
```

Conversely, you can allow all websites except those explicitly blocked:
```shell
sudo iptables -P INPUT ACCEPT
```