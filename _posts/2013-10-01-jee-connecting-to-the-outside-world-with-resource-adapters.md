---
layout: post
title: "JEE: Connecting to the outside world with JCA connectors - Part 1"
description: "Thoughts about configuration and how to make it simpler by using JCA connectors"
category: Java
tags: [JEE, JCA, Connector, Resource Adapter]
---
{% include JB/setup %}

In most JEE projects I came across so far there were issues with the configuration. The handling is somehow cumbersome and error-prone. Feeding changes to all configuration files for all testing environments is not my favorite work. 

##Two kinds of configuration data

The configuration of an application consisted out of some business configuration if at all and connectivity configuration for accessing remote systems. Please have a look at your own configuration files, I hope you'll agree. 

In a distributed world you'll have web applications or EJBs that need to connect to remote systems. You'll have to configure this access somewhere. In real life for accessing these properties people might:

* Inject a JNDI entry
* Read a system property
* Build your own configuration mechanism

Whereas JNDI would be a JEE way for doing this people often want to have their properties in one single property file. The JNDI approach gets rated as too complicated, especially when it comes to testing. In most case people start to write their own configuration framework, either simply for loading properties as system properties or for loading and accessing. That's when things get out of control:

* You can't put the properties files into your war or jar because urls, users and passwords 
  will be different in production than in test
  
  * Therefore you place it outside the JEE container somewhere on the file system, what in my opinion is a habit

* You'll provide some singleton class for accessing the configuration
  * Unit tests will start to fail just because some other test initialized that class with different values
  * Singletons (not the JEE ones) are evil anyway

* You will test less because configuration is a pain in the ass

###Have a close look at your current application configuration

What to do? When I looked closely at the application configuration I noticed that there is mostly just connectivity configuration. There's also configuration concerning the business logic but do we really need that?

###Business configuration data

Business configuration configures the business behavior of your code. In my experience these configuration values are set once by the developer and never get changed again. Not for test and not even for production. If it gets changed then a developer is doing the change. Maybe even only in the trunk of your source code. Therefore this configuration property is actually not needed at all if you use it as default in your code. This blog is not about this kind of configuration and how to access it (I would prefer JNDI) that's why I stop here.

###Connectivity configuration data

The configuration that changes often on different test and prod environments is connectivity configuration. This might be urls of remote systems, users and passwords. 

Why are these values in our application configuration? Ever used a database connection? Yes? What do we do with Data Sources? 

* We configure it in the container and inject it into the code
* The database configuration is outside our application configuration. 

If we could do this with the configuration for our other remote systems **the common application configuration might become obsolete** :-)

##Treat remote systems like Data Sources

Now let's go JEE. A database is a remote system or a remote resource. Let's do the same with all remote resources. JDBC is a very old standard. With JEE the [Java EE Connector Architecture](http://en.wikipedia.org/wiki/Java_EE_Connector_Architecture) was introduced for accessing remote resources. Actually, JDBC is (almost) the same as a JCA connector. To make this clear have a look at JBoss. JBoss uses [Ironjacamar](www.ironjacamar.or) as the JCA implementation. To define a Data Source you can deploy it as Ironjacamar connector, therefore, on JBoss your database connection is realized as a connector.

###Benefits

OK, let's assume we put our remote connection code into JCA connectors, what are the benefits of that except more (maven) projects, more code, more complexity

* The business code works on connection objects and does not care about how this connection is established

  * This is the same handling that we already know from JDBC Data Sources
* The business code can easily be mocked for unit testing (but this might also be true with the current approach)
* In a JEE container you can mock the remote system by implementing a second connector with the same api

  * This might become handy if a remote system is not available on all test environments. No code or configuration change is needed in your business code.
* If multiple modules connect to the same remote system deploy the connector once and create instances with different properties for each application in the container
* Code generation (e.g. from WSDL) is done in the connector api project. Your business code project is free of WS frameworks for code generation.
* Confidential configuration like user names and passwords are no longer in the application configuration and under the control of the developer

  * It's now under control of the [application deployer](http://docs.oracle.com/javaee/6/tutorial/doc/bnaca.html#bnaci) or the operator. Of course this could still be the same developer but in a different role at a different time

###Counter-arguments you'll face

If you mention connectors or JCA in your office people will probably call you crazy. You'll hear counter-arguments like
* This is too much overhead
* This is over-engineered
* This is too complicated
* This makes configuration complicated

Yes, there is overhead. You'll end with several new (maven) projects. I would suggest two per remote system. One for the api and one for the connector implementation. The benefit of this is that these projects are quite simple and pom.xml files for example are easy to understand even if we do some kind of source generation from WSDL. On the release management side you can provide bug fixes without rebuilding and redeploying your business application, you simple deploy the connector.

Is it over-engineered? I would call it well-engineered. It's JEE to the core! It splits your code into smaller artefacts and you'll get a better [separation of concerns](http://en.wikipedia.org/wiki/Separation_of_concerns)


Is it complicated? Well, you should have a look at a JCA example before you start or base your connectors on a base implementation. Adam Bien provides a very simple example of a connector in his [connectorz project](http://connectorz.adam-bien.com/). It's not that complicated at all. But yes, when it comes to deployment there's more to deploy that one single war file. In my opinion you should prefer the additional deployment complexity to a huge and messy project (and pom.xml file) that mix business code with connectivity.

When it comes to configuration I say that it makes configuration easier. Your application configuration is smaller or even disappears and depending on your company's setup you are not responsible for configuring connectivity matters. If you're still responsible then this might be because you act in the [JEE roles](http://docs.oracle.com/javaee/6/tutorial/doc/bnaca.html) of a [application component provider](http://docs.oracle.com/javaee/6/tutorial/doc/bnaca.html#bnacd) and an [application deployer](http://docs.oracle.com/javaee/6/tutorial/doc/bnaca.html#bnaci) at the same time. This is OK, no worries, but: **Always keep in mind what role you're currently playing and what responsibilities it includes!**

##Let's write a connector
As this post is already quite long I will postpone the code to a later blog post.

In part 2 I will introduce a base implementation I wrote that is based on the [connectorz project](http://connectorz.adam-bien.com/). It will provide base classes for creating connectors that only use url, user and password as configuration parameters. With only a little effort you'll be able to create additional connectors that connect to remote systems.

So long... and please have a close look at your configuration files





