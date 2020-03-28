---
title: "Bad HR expirience or How new repo was born"
date: 2020-03-28T16:03:07+01:00
draft: false
comments: false
cover: "img/abstract-art-blur-bright.jpg"
tags:
  - hr
  - story
---

# [Logektor ⚡️](https://github.com/lalabuy948/logektor)

## Forewords

This repo has some background, let me tell about it. Once, quite some time ago, I came to the company on the interview, for quite good and interesting position. We were talking about what they need and what I will need to do on this position. Meeting it self was good and I was so excited about that position.

But after few days I received tests assignment from them. I was a bit confused, coz for that kind of positions there is no test assignment usually. I was busy and working 24/7 in other company and had a business trip for 3 days (two of the days was weekend) that week and I told them about it.

After a that week I got an email where they were saying thanks for your effort but you failed tests assignment. Since I even didn't touch it. I was super disappointed.. Company even didn't try to look into the situation.

I do not remember all requirements. But basically they asked to build high-loaded system to store tracking events. Few days ago I had an insomnia and build it within one night. 

## System architecture

Let me tell about architecture. Since you need to track events from the website and after that show some reports, I decided to use kinda cqrs as global patter for this. We need high event throughput, that's why client in go (fasthttp) + Kafka. Workers which are storing that events in db are written in python, but it really doesn't matter, because I wanted to show that this architecture allows you to use polyglot approach. For database I used postgres to simplify the solution, but I would recommend something like clickhouse or redshift. Essentially column oriented databases. Each component of the system is designed in a way to be scaled and replaceable in the future. Need to change worker? Sure. Need to change warehouse? Go for that, replace it with Hadoop or whatever. I didn't create dashboard app, but maybe I will ask some frontend friend to help we with that in the future. I just wanted to show the principle and architecture.

[Logektor diagram](https://github.com/lalabuy948/logektor/blob/master/github/EventTrackingSA.svg)

[Logektor repo](https://github.com/lalabuy948/logektor)

Why I decided to write this article? To show to HR or other 
lead people who in charge of hiring how you can lose a great future employee. Im not saying that I'm a great future employee, just showing my own experience. Might it's correct to follow some company's corporate laws and such but we are all people, unique people with diffirent situations in life

Not sure if it will be useful to anyone, but i decided to share it.

Cover from www.pexels.com by @pixabay.
