---
layout: post
title: "JBoss EAP6: Upgrade to RESTEasy3.x"
description: ""
category: "JEE"
tags: [Jee, JBoss, JBossEAP, JAX-RS 2.0, REST, RESTEasy]
---
{% include JB/setup %}

When working with JBoss EAP 6.x you might reach the point, where the you would like to use the latest JAX-RS 2.0 features instead of the provided JAX-RS 1.1. In my case we hat to upgrade because of a bug in [RESTEasy 2](http://resteasy.jboss.org/) concerning sub-resource locators when using the proxy clients.

According to [the RESTEasy page](https://docs.jboss.org/resteasy/docs/3.0.2.Final/userguide/html_single/index.html#upgrading-eap61) you simply have
to unzip a file that contains the new modules for JBoss. This is a wonderful solution when working locally but it's a very bad solution for 
production servers where the modules are installed through RPM packages. In that case you shouldn't manually unzip some modules over the JBoss installation.

After several attempts of packing our own [RESTEasy](http://resteasy.jboss.org/) modules we found the solution for packing [RESTEasy](http://resteasy.jboss.org/) in the WAR file and leave the provided [RESTEasy 2](http://resteasy.jboss.org/) installation alone. After all, it's not that complicated. There are even blog post about that topic but some are a bit outdated when it 
comes to naming of modules and extensions.

So, here is what's needed to upgrade [JBoss EAP 6.2](https://www.redhat.com/products/jbossenterprisemiddleware/application-platform/) to [RESTEasy 3](http://resteasy.jboss.org/).

## pom.xml

As we provide our own version of RESTEasy 3 we have to pack them into our WAR file. You'll need the following dependencies:

{% highlight xml %}
<dependency>
  <groupId>org.jboss.resteasy</groupId>
  <artifactId>resteasy-jaxrs</artifactId>
  <version>3.0.8.Final</version>
</dependency>
<dependency>
  <groupId>org.jboss.resteasy</groupId>
  <artifactId>resteasy-servlet-initializer</artifactId>
  <version>3.0.8.Final</version>
</dependency>
<dependency>
  <groupId>org.jboss.resteasy</groupId>
  <artifactId>resteasy-jackson-provider</artifactId>
  <version>3.0.8.Final</version>
</dependency>
<dependency>
  <groupId>org.jboss.resteasy</groupId>
  <artifactId>resteasy-cdi</artifactId>
  <version>3.0.8.Final</version>
</dependency>
{% endhighlight %}

## jboss-deployment-structure.xml

We need to tell JBoss that we don't want the default JAX-RS 1.1 to be loaded but our own implementation. For this we have to:

- Disable the default JAX-RS subsystem
- Exclude the ``javaee.api`` module which contains all JEE6 apis
- Manually add all [JEE6-apis](http://www.oracle.com/technetwork/java/javaee/tech/javaee6technologies-1955512.html) except the ``javax.ws.rs.api`` api

The WEB-INF/jboss-deployment-structure.xml looks like this:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<jboss-deployment-structure xmlns="urn:jboss:deployment-structure:1.2">
    <deployment>
        <exclude-subsystems>
            <!-- Disable the default JAX-RS subsystem -->
            <subsystem name="jaxrs" />
        </exclude-subsystems>
 
        <exclusions>
            <!-- Exclude the ``javaee.api`` module which contains all JEE6 apis-->
            <module name="javaee.api" />
        </exclusions>
 
        <dependencies>
            <!-- Manually add all JEE6-apis except the javax.ws.rs.api api -->
            <module name="javax.activation.api" export="true" />
            <module name="javax.annotation.api" export="true" />
            <module name="javax.ejb.api" export="true" />
            <module name="javax.el.api" export="true" />
            <module name="javax.enterprise.api" export="true" />
            <module name="javax.enterprise.deploy.api" export="true" />
            <module name="javax.inject.api" export="true" />
            <module name="javax.interceptor.api" export="true" />
            <module name="javax.jms.api" export="true" />
            <module name="javax.jws.api" export="true" />
            <module name="javax.mail.api" export="true" />
            <module name="javax.management.j2ee.api" export="true" />
            <module name="javax.persistence.api" export="true" />
            <module name="javax.resource.api" export="true" />
            <module name="javax.rmi.api" export="true" />
            <module name="javax.security.auth.message.api"
                export="true" />
            <module name="javax.security.jacc.api" export="true" />
            <module name="javax.servlet.api" export="true" />
            <module name="javax.servlet.jsp.api" export="true" />
            <module name="javax.transaction.api" export="true" />
            <module name="javax.validation.api" export="true" />
            <!-- <module name="javax.ws.rs.api" export="true" services="export"/> -->
            <module name="javax.xml.bind.api" export="true" />
            <module name="javax.xml.registry.api" export="true" />
            <module name="javax.xml.soap.api" export="true" />
            <module name="javax.xml.ws.api" export="true" />
            <!-- This one always goes last. -->
            <module name="javax.api" export="true" />
        </dependencies>
    </deployment>
</jboss-deployment-structure>
{% endhighlight %}

That's it! almost...

## Almost there

If you put the JAX-RS annotations (like ``@Path``) directly on your resource implementations the work done above should be sufficient.

In my case it didn't work. In my project we have an api- and a implementation project. This because we want to use the interfaces on client side 
with the proxy client feature of RESTEasy. This way we don't have to care much about building REST urls and parameters. So, we have our interfaces which
are annotated with JAX-RS stuff. The implementation is completely JAX-RS free and only uses CDI/EJB annotations. In this constellation RESTEasy wasn't
able to find all classes involved. Looks like the automatic scanning (that used to work in RESTEasy 2) doesn't work anymore. I think this might be a bug but maybe we're missing something. 

As a workaround you can register all resource classes (the implementations) and exception mappers manually in your rest application class:

{% highlight java %}
@ApplicationPath("/rest")
public class MyApplication extends Application {
    @Override
    public Set<Class<?>> getClasses() {
        Set<Class<?>> impls = new HashSet<>();
        // resources
        impls.add(MyResource.class);
        ...
        // exception mappers
        impls.add(MyExceptionMapper.class);
        ...
        return impls;
    }
}
{% endhighlight %}

That did the trick!

I hope that others find this post useful. We wasted several days on this.

And please let me know if there's a solution for the class scanning issue. Thanks!