+++
draft = false
date = 2019-06-26T15:38:02-07:00
title = "Seattle Cassandra Users: An OSS Java Abstraction Layer for Cassandra"
slug = ""
+++

* Date : June 26, 2019
* Location: Seattle, WA
* Event: [Seattle Cassandra Users](https://www.meetup.com/Cassandra-Seattle-Users/events/262450272/)
* Slides: [SlideShare](https://www.slideshare.net/JoshTurner5/seattle-cassandra-users-an-oss-java-abstraction-layer-for-cassandra)

### Abstract

Project Casquatch is a database abstraction layer with code generation designed to streamline Cassandra development. Out of the box it comes pre-tuned with high available policies including load balancing, geo-redundancy, connection pooling, etc., sitting on top of the DataStax driver using native APIs. All of this is abstracted behind the ever prevalent POJO. Instead of writing CQL, we utilize generic programming that allows you to simply pass a generated POJO to a save() method or populate with a getById(). This is the same code reportedly used by T-Mobile for multiple national platforms including the activation of the Apple Watch and Galaxy Watch, T-Mobile Payments, Digits, and many others.
