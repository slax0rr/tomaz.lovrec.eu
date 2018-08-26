+++
draft = true
date = 2018-08-26T12:58:27+02:00
title = "Cleartext passwords are bad and you should feel bad"
slug = "cleartext-passwords-are-bad-and-you-should-feel-bad"
tags = ["development","security","authentication"]
categories = ["development","security"]
+++

I have just finished reading a slightly older Twitter
[thread](https://twitter.com/tmobileat/status/981418339653300224) involving
storage of passwords in clear text format in the database by one of the largest
mobile network company in Austria. This enraged me enough to go an write this
blog post.

What bothers me more than practicing this, is the very apparent lack of sense of
security by their employees, stating how their system is "amazingly good", and
that no one could breach their defenses. This makes me wonder, how can one even
think that? How can one even make such a statement after admitting they are
storing passwords in clear text format? Well, after a little bit of examination,
I came across this image, proving that their awesome security is in fact a big
fat [lie](https://pbs.twimg.com/media/DaIHlx2W4AEx_ww.jpg:large). PHP version
5.1.6, released back in 2006, long since reached End-Of-Life. Kernel version of
the server, 2.6.18, also released back in 2006. And yet they try to convince us
that their 12 year old software is "amazingly good" secure. Yeah, I'm going to
take a 'no' on that.

Even if their software *was* up-to-date and "amazingly good" secure, this is
still no excuse to store your passwords in clear text format. Another fact that
the employees of T-Mobile Austria seem to not know about. Even if you are
capable of guaranteeing that no one could breach your security, how can you
guarantee that some employee who has access to the database, and gets laid off
on some unfair terms for them? How can you guarantee they wont make a copy of
the database, and dump it out somewhere on the internet? There are countless
more possible scenarios, but lets just leave it at, no system is ever secure
enough that you could store sensitive information in plain text format. Period.

There is literally no excuse you can use to store such sensitive information in
plain text. There is even no valid reason why you would need to be able to
re-use the users authentication information. User should know this to
authenticate themself against their product, and this is it. No, encrypting
passwords is also not OK. You should salt and hash them with a strong algorithm
that will not be brute-forced by a pocket calculator. Yes I am looking at you
MD5 and SHA1.

With this I will end my rant, and hope that I will be able to provide a better
reading material the next time. Until next time!
