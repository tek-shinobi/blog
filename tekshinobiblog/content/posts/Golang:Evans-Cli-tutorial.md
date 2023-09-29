---
title: "Golang:Evans Cli Tutorial"
date: 2017-09-29T14:17:37+03:00
draft: false 
categories: ["golang"]
tags: ["tools"]
---
to launch evans on localhost and port 8080:
`evans --host localhost --port 8080 -r repl`

to list all rpc services:
`show service`
**Note**: if in above step, you see nothing, probably you are in the wrong package. Services belong to a package. Switch to the right package by `show package` and then choose the package by `package <package_name>` 

call an rpc named: `Login`
`call Login`
(you will be prompted to fill out the request object params)





