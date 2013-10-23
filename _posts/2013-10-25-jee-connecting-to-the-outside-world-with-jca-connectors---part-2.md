---
layout: post
title: "JEE: Connecting to the outside world with JCA connectors - Part 2"
description: ""
category: JEE
tags: [JEE, JCA, Connector, Resource Adapter, Configuration]
---
{% include JB/setup %}

After philosophizing about application configuration in [Part 1]( ../../../10/01/jee-connecting-to-the-outside-world-with-resource-adapters) it's now time to get our hands dirty. 
What are we going to write an outbound resource adapter! The resource adapter is based on my GitHub project [outbound-connector](https://github.com/scheuchzer/outbound-connector) 
which provides some base classes that reduces the code for new simple resource adapters to almost nothing!

# echo-connector - A simple stupid resource adapter

We will now create a resource adaptor that echos a message. To show that the configuration of url, username and password is working, even when working with multiple
configurations of the resource adapter the echo resource adapter will add this information to the result. You might have noticed already that we're not going
to connect to a remote system at all for this demonstration.


## outbound-connector - The base implementation

When writing a resource adapter from scratch you won't come around some boilerplate code. It's that code that makes your resource look complex. When writing 
a simple resource this boilerplate doesn't do much at all. So, it can be placed in a few base classes that hide it from you. That's exactly what the 
[outbound-connector](https://github.com/scheuchzer/outbound-connector) project is doing!

The project comes with two main maven projects and some examples.

### remote-system-connector-api 

A resource adapter should always consist out of an API and a implementation project. The `remote-system-connector-api` contains two interfaces that APIs of 
new resource adapters can extend. The interfaces are:

{% highlight java %}
com.ja.rsc.api.Connection
com.ja.rsc.api.ConnectionFactory
{% endhighlight %}
  
These interfaces mainly aggregate some java interfaces so that you don't forget to include them :-) The API of our new resource adapter will extend these 
two interface and we're done with the API project.

### remote-system-connector

On the implementation side, the project `remote-system-connector` provides the base classes listed next. We will extend these for our new resource adapter.
    
{% highlight java %}
com.ja.rsc.AbstractAdapter
com.ja.rsc.AbstractConnectionFactory
com.ja.rsc.UrlBasedConnection
com.ja.rsc.UrlBasedManagedConnectionFactory.java
com.ja.rsc.UrlBasedManagedConnection.java
{% endhighlight %}
    
These classes will do the communication or integration with the container. Extending five classes for a new resource adapter looks like a lot of work, but relax, 
our classes will almost be empty. They simply must exist to fulfill the JCA contracts.

# Define the API for your resource adapter

We create a new maven project called `echo-connector-api`. As the only dependency we add the the `remote-system-connector-api` project
    
{% highlight xml %}
<dependency>
  <groupId>com.java-adventures.rsc</groupId>
  <artifactId>remote-system-connector-api</artifactId>
  <version>1.0.0</version>
</dependency>
{% endhighlight %}

# Interface: EchoConnection

We will provide one echo Method. We put it in a class called `EchoConnection`. The interface extends the `Connection` interface from the `remote-system-connector-api`. 

{% highlight java %}
package com.ja.rsc.echo.api;
import com.ja.rsc.api.Connection;
	
public interface EchoConnection extends Connection {
  EchoResponse echo(String text);
}
{% endhighlight %}

We use a `EchoRespone` object as return value. We use this object to store the current configuration properties of the connection. You won't do this in 
real life, of course.

{% highlight java %}
package com.ja.rsc.echo.api;

public class EchoResponse {

  private String text;
  private String url;
  private String username;
  private String password;
  
  // getters and setters omitted
}
{% endhighlight %}

# Interface: EchoConnectionFactory

The resource that we are going to inject into a web application or an EJB will be a connection factory. We define it in the interface 
`EchoConnectionFactory` that extends `ConnectionFactory` located in `remote-system-connector-api`. You will notice that we only have to define the generic 
connection type. No methods have to be defined.

{% highlight java %}
package com.ja.rsc.echo.api;
import com.ja.rsc.api.ConnectionFactory;

public interface EchoConnectionFactory extends ConnectionFactory<EchoConnection> {

}
{% endhighlight %}

That's it. The API for the echo-connector is already defined! If you're writing a resource adapter for a SOAP service this API project would be the perfect
location to generate the java code from the WSDL of the web service.

# Implement the resource adapter

The resource adapter gets implemented in a separate maven project `echo-connector`. Dependencies are

{% highlight xml %}
<dependency>
  <groupId>com.java-adventures.rsc</groupId>
  <artifactId>echo-connector-api</artifactId>
  <version>${project.version}</version>
</dependency>
<dependency>
  <groupId>com.java-adventures.rsc</groupId>
  <artifactId>remote-system-connector</artifactId>
  <version>1.0.0</version>
</dependency>
{% endhighlight %}
    
As mentioned above, we need to implement five classes. They all extends base classes found in `remote-system-connector`.

## Class: EchoResourceAdapter

{% highlight java %}
package com.ja.rsc.echo;
import javax.resource.spi.Connector;
import javax.resource.spi.TransactionSupport;
import com.ja.rsc.AbstractAdapter;

@Connector(
  reauthenticationSupport = false, 
  transactionSupport = 
  TransactionSupport.TransactionSupportLevel.NoTransaction)
public class EchoAdapter extends AbstractAdapter {

} 
{% endhighlight %}    
    
The base class handles the life cycle of the adapter. Out class is annotated as `@Connector`. Define the 
transaction behavior in the annotation. Our simple echo adapter doesn't support transactions at all. The use of annotations makes the use of a
`ra.xml` descriptor obsolete. 
    
## Class:  EchoManagedConnectionFactory


The `EchoManagedConnectionFactory` creates managed connections. Managed connections are used to implement the transaction behavior. We don't support transactions
in this example what makes implementation simple. Even if the following class looks massive, the class only contains two methods with only one statement actually.

{% highlight java %}
package com.ja.rsc.echo;
import java.io.Closeable;
import javax.resource.spi.ConnectionDefinition;
import javax.resource.spi.ConnectionManager;
import javax.resource.spi.ConnectionRequestInfo;
import javax.resource.spi.ManagedConnection;
import javax.resource.spi.ManagedConnectionFactory;
import com.ja.rsc.GenericManagedConnectionFactory;
import com.ja.rsc.UrlBasedManagedConnection;
import com.ja.rsc.UrlBasedManagedConnectionFactory;
import com.ja.rsc.UrlConnectionConfiguration;
import com.ja.rsc.echo.api.EchoConnection;
import com.ja.rsc.echo.api.EchoConnectionFactory;

@ConnectionDefinition(
      connectionFactory = EchoConnectionFactory.class, 
      connectionFactoryImpl = InMemoryEchoConnectionFactory.class, 
      connection = EchoConnection.class, 
      connectionImpl = InMemoryEchoConnection.class)
public class EchoManagedConnectionFactory extends
  UrlBasedManagedConnectionFactory<UrlConnectionConfiguration> {

  public EchoManagedConnectionFactory() {
    super(new UrlConnectionConfiguration());
  }

  @Override
  protected Object createConnectionFactory(
    GenericManagedConnectionFactory mcf, ConnectionManager cm) {
    return new InMemoryEchoConnectionFactory(mcf, cm);
  }

  @Override
  protected ManagedConnection createManagedConnection(
    UrlConnectionConfiguration connectionConfig,
    ManagedConnectionFactory mcf,
    ConnectionRequestInfo connectionRequestInfo) {

    return new UrlBasedManagedConnection<UrlConnectionConfiguration, InMemoryEchoConnection>(
      connectionConfig, mcf, connectionRequestInfo) {

      @Override
      protected InMemoryEchoConnection createConnection(
        UrlConnectionConfiguration connectionConfiguration,
        ManagedConnectionFactory mcf,
        ConnectionRequestInfo connectionRequestInfo,
        Closeable managedConnection) {
          return new InMemoryEchoConnection(connectionConfiguration, managedConnection);
      }
    };
  }
}
{% endhighlight %}

The `@ConnectionDefinition` provides information about your resource adapter. It wires the connection interfaces to the connection classes.

## Class: InMemoryEchoConnectionFactory

This class is named `InMemory` because our resource adapter will do its work purely in the memory and won't do any remote calls.

{% highlight java %}
package com.ja.rsc.echo;
import javax.resource.spi.ConnectionManager;
import javax.resource.spi.ManagedConnectionFactory;
import com.ja.rsc.AbstractConnectionFactory;
import com.ja.rsc.echo.api.EchoConnection;
import com.ja.rsc.echo.api.EchoConnectionFactory;

public class InMemoryEchoConnectionFactory extends
	AbstractConnectionFactory<EchoConnection> implements
	EchoConnectionFactory {

  public InMemoryEchoConnectionFactory(ManagedConnectionFactory mcf, ConnectionManager cm) {
    super(mcf, cm);
  }
}
{% endhighlight %}
    
This class allocates connection from the connection pool. Luckily this has been consolidated in the base class `AbstractConnectionFactory` that we extend here.
Therefore we don't have to implement any code besides a constructor.    


# Class: InMemoryEchoConnection

Now things are starting to get interesting. This class is where you place the actual business code of the resource adapter. The base class `UrlBasedConnection` 
provides access to our properties url, username and password. Nice, isn't it. They're just there waiting for you :-) You could use them to open a connection
to the remote system. For the sake of demonstration we store the configuration values in the `EchoResponse`.

{% highlight java %}
package com.ja.rsc.echo;
import java.io.Closeable;
import com.ja.rsc.UrlBasedConnection;
import com.ja.rsc.UrlConnectionConfiguration;
import com.ja.rsc.echo.api.EchoConnection;
import com.ja.rsc.echo.api.EchoResponse;

public class InMemoryEchoConnection extends
    UrlBasedConnection<UrlConnectionConfiguration> implements
    EchoConnection {

  public InMemoryEchoConnection(
      UrlConnectionConfiguration connectionConfiguration,
      Closeable closeable) {
      super(connectionConfiguration, closeable);
  }

  @Override
  public EchoResponse echo(String text) {
      EchoResponse response = new EchoResponse();
      response.setText(text);
      response.setUrl(getUrl());
      response.setUsername(getUsername());
      response.setPassword(getPassword());
      return response;
  }
}
{% endhighlight %}
     
With this the implementation of the echo resource adapter is done.

# Configure an echo connection

The code of the resource adapter can now be compiled and deployed. The package will be of type `RAR` and can be deployed like an application in a JEE container.
After the deployment connection pools and connection resources have to be defined inside your JEE container. How to do this depends on the JEE container. I will only 
describe the configuration steps for Glassfish. In the examples there is are scripts that will do it through command line tools for Glassfish and JBoss/Wildly.

## Configure a Connector Connection Pool

1.  After deploying the RAR file open the [Glassfish admin console](http:/localhost:4848), 
    go to `Resources`-> `Connectors` -> `Connector Connection Pools` and click `New`.
2.  Select a pool name and select the `echo-connector` resource adapter in the drop down and 
    click `Next` <br/> 
    ![Creating a connection pool]({{ site.url }}/assets/posts/jee-connecting-to-the-outside-world-with-jca-connectors/create-pool.png)
3.  On this page you can configure the connection pool as you known it from database 
    connection. Minimum and maximum connections and so on. At the bottom of the page we find 
    three properties that we accessed in the `EchoConnection` before. Add some values. <br/> 
    ![Creating a connection pool]({{ site.url }}/assets/posts/jee-connecting-to-the-outside-world-with-jca-connectors/create-pool2.png)
4.  Click `Finish`

> Please note that the configuration parameters for the connectivity is no longer part of your application. It's completely separated! It's up to 
> the operator or the deployer to configure it. The application developer doesn't has to know production password anymore what might be a good thing!

## Configure a Connector Resource

The resource we configure here is what we are going to inject into a web application or ejb.

1.   go to `Resources`-> `Connectors` -> `Connector Resource` and click `New`.
2.   Define a JDNI name and select the connection pool you created before. The JNDI name is going to be used later in `@Resource` annotations.<br/>
     ![Creating a resource]({{ site.url }}/assets/posts/jee-connecting-to-the-outside-world-with-jca-connectors/create-resource.png)
3.   Click `OK`

The echo resource adapter is now configured and can be used inside web applications and ejbs.

# Using the resource adapter

You can now access the echo connection by injecting a `EchoConnectionFactory` that provides access to
the `EchoConnection`. The name attribute of the `@Resource` annotation is the JNDI name we specified before. You can inject the `EchoConnectionFactory` 
into servlets or ejbs for example.

{% highlight java %}
@Resource(name = "jca/echo")
private EchoConnectionFactory echo;
}
{% endhighlight %} 

Connections always have to be closed after use so place your code inside a `try-with-resource` or close it in a `finally` block.

{% highlight java %}
try (EchoConnection connection = echo.getConnection()) {
	EchoResponse response = connection.echo(text);
	return response.toString();
} catch (Exception e) {
	throw new WebApplicationException(e, Status.INTERNAL_SERVER_ERROR);
}
{% endhighlight %} 

# The complete example

The working sample code with echo connector and demo REST application can be found on [GitHub](https://github.com/scheuchzer/outbound-connector) in 
the [examples](https://github.com/scheuchzer/outbound-connector/tree/master/examples/urlbased/echo) directory.

There is a (un)deploy script that works with Glassfish and JBoss/Wildfly. They will deploy resource adapter and demo application and configure
two connections with different configuration parameters. The different connections can be accessed through the demo application by going to the urls

*   <http://localhost:8080/demo-app/rest/echo/foo>
*   <http://localhost:8080/demo-app/rest/echo2/bar>

If you want to test in your console then call the scripts `testEcho.sh` and `testEcho2.sh`:

{% highlight bash %}
$ ./deploy.sh glassfish

$ ./testEcho.sh 
text=HelloWorld,
url=http://url1,
username=username1,
password=password1

$ ./testEcho2.sh 
text=HelloWorld,
url=http://url2,
username=username2,
password=password2
{% endhighlight %} 

# Conclusion

We are done. What have we reached?

* Connectivity configuration is separated from the application
  * Less application configuration, maybe even no application at all
  * Same WAR/EAR/RAR for all (testing) environment
* Configuration similar to datasources
* Usage similar to datasources
* Resource adapter and application can be release independently
* Faster builds

The application doesn't know with which echo implementation it's talking. The implementation can be replaced by the deployer or the 
operator without touching the application or the application configuration at all! That's is just perfect for testing environments for example
where certain peripheral systems might not be available for every stage.

I thinks it's worth investing some training in working with resource adapters even though it's not that obvious at the beginning.  
