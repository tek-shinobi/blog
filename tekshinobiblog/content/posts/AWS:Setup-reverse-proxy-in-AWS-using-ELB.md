---
title: "AWS:Setup Reverse Proxy in AWS Using ELB"
date: 2023-10-13T12:06:40+03:00
draft: false 
categories: ["backend", "grafana"]
tags: ["aws"]
---

We'll take grafana as example application. We will install grafana on an EC2 instance and configure reverse proxy for it.

Of course, ssh into the EC2 instance and install grafana like so:
```shell
sudo apt-get install -y software-properties-common wget apt-transport-https
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update && sudo apt-get install grafana 
```

then enable and start grafana server:
```shell
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

To make it easy, configure a public IP address for the EC2 instance ... you can easily do it via EC2 dashboard. Lets call it 1.2.3.4

You can launch the grafana dashboard in you browser like so: `http://your_ip_address:3000/login` i.e. `http://1.2.3.4:3000/login`

We will not do any grafana configuration, instead we will focus on creating a reverse proxy.
Note, if 3000 is not in your inboud rules in your security group for this instance, then you will need to do that first. Follow below where I will descriobe how to do it.

## Why reverse proxy?
 - __ssl terminaltion__: if you notice, grafana application has no notion of https. This is in general true for most microservice applications. We don't enable TLS in them. They stay http only and instead a reverse proxy like nginx provides ssl termination (meaning, it handles TLS and all applications behind it stay with http).
 - disable public access to application port: its idiomatic to have a front end send http/https requests to standard http/https ports of 80/443. A reverse proxy listens at 80 and 443 and based on the uri forwards the request to the respective application on its port. This decouples frontend with backend low-level details like internal application ports and good for security.

## Traditional nginx approach
Before we look at the AWS ELB approach, let's take a look at how traditionally its done for any server, including our EC2 server. Tradionally, you will need a webserver like Apache `httpd` or `nginx`. We will take `nginx`.

Install nginx and set it to launch at system start:
```shell
sudo apt-get update
sudo apt-get install nginx -y
sudo sytemctl enable nginx
sudo sytemctl start nginx
sudo sytemctl status nginx
```

if you now enter http://1.2.3.4 (i.e. your public IP of EC2 instance, without any port) in the browser, it will goto the default nginx page. Note, you will need to have port 80 and 443 (http and https ports) added to inbound rules for the security group assiciated with your EC2 instance. See below the process for making an inbound rule for 3000. 

Now we will configure nginx so that whenever we visit that ip address, it will forward that request to our grafana application on port 3000

```shell
sudo touch /etc/nginx/conf.d/amazon.conf
sudo nano /etc/nginx/conf.d/amazon.conf
```

here are the contents of `amazon.conf`:
```shell
server {
        server_name dashboard.helloexample.com;
        listen 80;
        listen [::]:80;
        location / {
                proxy_set_header Host dashboard.helloexample.com;
                proxy_set_header Origin http://dashboard.helloexample.com;
                proxy_pass http://localhost:3000;

        }
}
```
restart nginx:
```shell
sudo systemctl restart nginx
```

Now in the browser, `http://dashboard.helloexample.com` will proxy pass the request to grafana application running on port 3000.

Next, to setup TLS certificate and enable https. use a service like `certbot` that will make the whole process automated and it will install a free, auto renewable TLS certificate from a valid CA (Letsencrypt).

An example video is here: https://www.youtube.com/watch?v=R5d-hN9UtpU

## AWS approach
If you saw the above part, you would have noticed a lot of manual installation of things like nginx, certbot and TLS certificates. AWS provides multiple ways to automate all this. One option is to list out all of the above installs and configurations in a `userdata` file when creating EC2 instance. Then if you create any other instance with the same userdata, all this will be auto configured. But we will take a different approach. This approach is OK if you have a more complex reverse proxy configuration. In our case, this is overkill.

Ok, an overview of what we plan:

ELB Load balancer instance -> Elb listener on port 443 -> target group -> connects to target instance on port 3000
We will also generate a TLS certificate with AWS and use it when creating a target group for port 443.

That's it. 

ok, on to action:

First, create a security group for the EC2 instance where the grafana application is running. Remember, we care only about inbound rules for our purpose. So we goto "inbound rules" tab and -> Edit inboud rule -> Add rule: create a rule for port 3000 (Type: Custom TCP, Protocol: TCP, PortRange: 3000, Source (dropdown)-> AnywhereIPv4). Add a description for the rule if you want. Save the rule.

Do the same for IPv6 for port 3000.

So our security group, lets call it `GrafanaSG`, has 2 inbound rules for port 3000, one for ipv4 and another for ipv6.

Add this secruity group to our EC2 instance.

(if you will notice, EC2 instance is currently have a public IP. So making this inboud rule for port 3000 will effectively make this port publicly accessible. We can deal with it later by making the IP private for the EC2 instance). 

Next step is to piece together a reverse proxy using Route53 and ELB.

To make the load balancer act as reverse proxy, we need 3 cruicial components aside from the load balancer instance itself.
1. Listener (will be created from the load balancer instance page via "Add Listener")
1. target group
1. security group (for the load balancer instance)

First. let's create a security group. We created one for our EC2 instance (with only port 3000). This one will be for our Load Balancer instance. In this security group, create inbound rules for ports 80 and 443. You can create for only IPv4 or both IPv4 and IPv6.

Now, let's create the load balancer instance. Create a Load balancer (EC2Dashboard -> Load Balancers -> Create load balancer)

Make it "internet-facing", IP address type as IPv4. Network mapping -> choose all three mappings (for regions).
For security group, add the one we created for load balancer (there will also be one there by default, called "default". Leave it there). So this load balancer instance has 2 security groups, default and another with ports80 and 443 that we just created.

create target group. 
You will see a little hyperlink within the Create Loadbalancer page for creating a target group. Click there.

On create target group page:
`Choose a target type: Instances` 
it is Instances because we are routing to EC2 instance. 

Give this target group a name like `grafana-80`

Protocol:HTTP
Port: 80

IPAddress type IPv4

No need to choose VPC, its already selected for you.

Protocol version: HTTP1

"Health Checks" -> very important section. Here is where you hook the endpoint path for the healthckeck of you application. From grafana documentation, their healthcheck endpoint is at `/api/health`. So put this path in "Health check path". __IMPORTANT__: "Health checl protocol" HTTP (and NOT HTTPS). Since grafana application has no idea what https is, use HTTP.

thats it. save.

You dont need to create another target group for port 443. TLS certificate handling is done in the Load balancer itself. Let's see how to do this. Trick is to add listener on port 443. Click Add listener-> Protocol HTTPS (it will auto select 443 port for you)

Routing actions -> Forward to target groups

Target group -> choose the grafana-80 target group that we made

Since this listener is for HTTPS protocol, you will see a section named "Secure listener settings"
Here is where you will setup TLS certificate.
Certificate source -> From ACM (choose a certificate from the drop down. You can create a new ACM certificate here as well by clicking on the link)

thats it.

You can now check the "Healthy"  indicator of the target group associated with load balancer `grafana-80`. It will say "Total targets" 1 and "Healthy" 1 and "Unhealthy" 0.

Now all that is needed is DNS setup for the url. Goto Route53. Either create a hosted zone or if there is already a zone present.. like "helloexample.com", then you just need to add the subdomain "dashboard.helloexample.com"


 Click on the hosted zone instance. Then Create Record -> Simple Routing -> Define simple record, Records Type: CNAME; subdomain: dashboard; value: for this .. goto load balancer instance -> Monitoring tab and copy the "DNS name"

 i.e. copied value of "DNS name" is pasted in value for simple record. Thats it. This value indicates the endpoint where loadbalancer is located. So whenever someone will type "https://dashboard.helloexample.com", the request will be routed to load balancer endpoint. Now all that is left is to add a redirection for "http://dashboard.helloexample.com" and make it goto "https://dashboard.helloexample.com". To do this, just add a listener on load balancer on port 80 with routing action "Redict to URL" and choose the Protocol as HTTPS and Port:443 . thats it. 


