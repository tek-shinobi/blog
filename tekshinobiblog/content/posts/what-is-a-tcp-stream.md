---
title: "What Is a Tcp Stream"
date: 2020-02-08T18:34:10+03:00
draft: false 
categories: ["networking"]
tags: ["networking", "tcp"]
---

The TCP protocol is modeled on the concept of a ***single continuous stream*** of unlimited length. This is a very important concept to understand, and is the ***number one cause of confusion*** that we see.

What exactly does this mean, and how does it affect developers?

Imagine that you're trying to send a few messages over the socket. So you do something like this (in pseudocode):
```
socket.write("Hi Sandy.");
socket.write("Are you busy tonight?");
```

How does the data show up on the other end? If you think the other end will receive two separate sentences in two separate reads, then you've just fallen victim to a common pitfall! ***Gasp!*** Read on.

TCP does not treat the writes as separate data. TCP considers all writes to be part of a single continuous stream. So when you issue the above writes, TCP will simply copy the data into its buffer:

TCP_Buffer = "Hi Sandy.Are you busy tonight?"

and then proceed to send the data as fast as possible. And in order to send data over the network, TCP and other networking protocols will be required to break that data into small pieces that can be transmitted over the medium (ethernet, Wi-Fi, etc). In doing so, TCP may break apart the data in any way it sees fit. Here are some examples of how that data might be broken apart and sent:

1.  "Hi San" , "dy.Ar" , "e you " , "busy to" , "night?"
2.  "Hi Sandy.Are you busy" , " tonight?"
3.  "Hi Sandy.Are you busy tonight?"


The above examples also demonstrate how the data will arrive at the other end. Let's consider example 1 for a moment.

Sandy has issued a socket.read() command, and is waiting for data to arrive. So the result of her first read might be "Hi San". Sandy will likely begin to process that data. And while the application is processing the data, the TCP stream continues to receive the 2nd and 3rd packet. Sandy then issues another socket.read() command, and this time she gets "dy.Are you ".

This highlights the continuous stream nature of TCP. The TCP protocol, at the developer API level, has absolutely no concept of packets or separation of data.

But isn't this a major shortcoming? How do all those other protocols that use TCP work?

### Example: HTTP
HTTP is a great example because it's so simple, and because most everyone has seen it before. When a client connects to a server and sends a request, it does so in a very specific manner. It sends an HTTP header, and each line of the header is terminated with a CRLF (carriage return, line feed). So something like this:
```
GET /page.html HTTP/1.1
Host: google.com
```
Furthermore, the end of the HTTP header is signaled by two CRLF's in a row. Since the protocol specifies the terminators, it is easy to read data from a TCP socket until the terminators are reached.

Then the server sends the response:
```
HTTP/1.1 200 OK
Content-Length: 216
```
Again, the HTTP protocol makes it easy to use TCP. Read data until you get back-to-back CRLF. That's your header. Then parse the content-length from the header, and now you can simply read a certain number of bytes.

Returning to our original example, we could simply use a designated terminator for our messages:
```
socket.write("Hi Sandy.\n");
socket.write("Are you busy tonight?\n");
```
And if Sandy was using AsyncSocket she would be in luck! Because AsyncSocket provides really easy-to-use read methods that allow you to specify the terminator to look for. AsyncSocket does the rest for you, and would deliver two separate sentences in two separate reads!

### Writes
***What happens when you write data to a TCP socket?*** When the write is complete, does that mean the other party received that data? Can we at least assume the computer has sent the data? The answer is NO and NO.

Recall two things:
1.  All data sent and received must get broken into little pieces in order to send it over the network.
2.  TCP handles a lot of complicated issues such as resending lost packets, and providing in-order delivery so information arrives in the proper sequence.

So when you issue a write, the data is simply copied into an underlying buffer within the OS networking stack. At that point the TCP software will begin its magic, which consists of all the cool stuff mentioned earlier such as:
-   breaking the data into small pieces such that they can be sent over the network
-   ensuring that lost pieces get properly resent
-   ensuring that your data arrives at the remote destination in the proper order
-   watching out for congestion in the network
-   employing fancy algorithms to accomplish all of this as fast as possible

So when you issue the command, "write this data" the operating system responds with "I have your data, and I will do everything in my power to deliver this to the remote destination."

BUT… ***how do I know when the remote destination has received my data?***

And this is exactly where most people run into problems. A good way to think about it is like this:

Imagine you want to send a letter to a friend. Not an email, but the traditional snail mail. You know, through the post office. So you write the letter and put it in your mailbox. The mailman later comes by and picks it up. You can rest assured at this point that the post office will make every effort to deliver the letter to your friend. But how do you know for sure if your friend received the letter? I suppose if the letter came back with a "return to sender" stamped on it you can be certain your friend didn't receive it. But what if it doesn't come back? Is it enough to know that it made it into your friend's mailbox? (Assume this is a really, really important letter.) The answer is no. Maybe it never leaves the mailbox. Maybe his roommate picks it up and accidentally throws it away. And if the roommate was responsible and left the letter on your friends desk? Would that be enough? What if your friend was on vacation and your letter gets lost in a pile of junk mail? So the only way to truly know if your friend received the letter is when you receive their response.

This is a great metaphor for sockets. When you write data to a socket, that is like putting the letter in the mailbox. The operating system is like the local mailman that comes by and picks up the letter. The giant post office system that routes the letter toward its destination is like the network. And the mailman that drops off your letter in your friends mailbox is like the operating system on your friends computer. It is then up to the application on your friends computer to read the data from the OS and process it (fetch the letter from the mailbox, and actually read it).

So how do I know when the remote destination has received my data? This is not something that TCP can tell you. At best, it can only tell you that the letter was delivered into their mailbox. It can't tell you if the application has read that data and processed it. Maybe the application on the remote side crashed. Or maybe the remote user quit the application before it had a chance to read the data. Or maybe the remote user experienced a power outage. Long story short, it is up to the application layer to answer this question if need be.

### Key Points
-   TCP is a continuous stream.
-   The TCP stream is divided into packets arbitrarily. The divisions do not indicate anything about the syntax of the message.
-   To handle details such as determining the end of a message, you need an additional protocol on top of TCP. For example:
-   You can specify a termination sequence to indicate the end of a message.
-   You can send the length of a message as part of the communication.
-   The TCP protocol does not provide guarantees that messages have been sent or received by the application.
-   To determine whether messages have been sent or received, you need an additional protocol on top of TCP. For example, your application can specify 