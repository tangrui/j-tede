# Introduction #

A lot of Java developers complain that Java development environment feels clunky, slow. That  leeds to negative feelings, dissatisfaction and ultimately lost time and money.

But it doesn't have to be like that. I want to show that development environment doesn't have to be heavyweight and that you can develop in a lightweight container even if you plan to deploy into a heavyweight environment.

My definition of heavyweight environment: environment that uses multiple datasources, jms providers, distributed transactions or deploys into a portal. Environment that takes minutes to start.

Motivation: Most of the content comes from my own experience or is freely available on the net. This project just compiles documentation and knowledge in hope that it will help other developers.

# Details #

This document describes configuration of the latest Tomcat 6.x.

## JNDI ##

Tomcat has a built-in support for JNDI. Unfortunately it doesn't have (AFAIK) a nice admin UI for administering it. One will have to edit xml files in order to configure JNDI.


### URL references ###


## Setup transaction manager ##

In order to use JTA we have to setup a transaction manager. In this case we will use [bitronix](http://docs.codehaus.org/display/BTM/Home).

### Bitronix setup ###

Download latest bitronix release from http://docs.codehaus.org/display/BTM/Home
At the moment of writing latest bitronix release is 2.0.1

  * copy btm-2.0.1.jar to the ${TOMCAT\_HOME}/lib
  * from /lib directory in archive copy geronimo-jms\_1.1\_spec-1.0.1.jar to the ${TOMCAT\_HOME}/lib
  * from /lib directory in archive copy geronimo-jta\_1.1\_spec-1.1.jar to the ${TOMCAT\_HOME}/lib
  * from /integration directory copy btm-tomcat55-lifecycle.jar to the ${TOMCAT\_HOME}/lib


Inside ${TOMCAT\_HOME}/conf directory create btm-config.properties with following content:

```
bitronix.tm.serverId=tomcat-btm-node0
bitronix.tm.journal.disk.logPart1Filename=${btm.root}/work/btm1.tlog
bitronix.tm.journal.disk.logPart2Filename=${btm.root}/work/btm2.tlog
bitronix.tm.resource.configuration=${btm.root}/conf/resources.properties
```

Please notice that we use ${btm.root} as a placeholder. Runtime has to be able to resolve it.
If you start tomcat from command line or script you need to change runtime parameters as in this example (windows):

> set CATALINA\_OPTS=-Dbtm.root=%CATALINA\_HOME%
> -Dbitronix.tm.configuration=%CATALINA\_HOME%\conf\btm-config.properties

The example is borrowed from [How to use BTM as the transaction manager in Tomcat 6.x](http://docs.codehaus.org/display/BTM/Tomcat13).

If you use eclipse you will have to configure eclipse server configuration.

  * Create new Tomcat server in eclipse
  * Double click on the newly created server so that configuration editor opens
  * Click on the "Open launch configuration"


![http://wiki.j-tede.googlecode.com/hg/img/eclipse_server_config.png](http://wiki.j-tede.googlecode.com/hg/img/eclipse_server_config.png)

  * On the "Arguments" tab under "VM arguments" add:
    * -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=9004
    * -Dbtm.root=${catalina.home} -Dbitronix.tm.configuration=${catalina.home}/conf/btm-config.properties

![http://wiki.j-tede.googlecode.com/hg/img/eclipse_vm_arguments.png](http://wiki.j-tede.googlecode.com/hg/img/eclipse_vm_arguments.png)


You may have noticed that I have used placeholder ${catalina.home}.
You can define ${cataline.home} as java environment variable or by configuring eclipse string substitution. The value should point to the directory where tomcat is installed.
On the screenshot there are two additional references: ${tomcat\_jmx} and ${tomcat\_bitronix}.
You can define string substitution under "Window->Preferences"-> Run/Debug -> String substitution.

![http://wiki.j-tede.googlecode.com/hg/img/eclipse_string_substitution.png](http://wiki.j-tede.googlecode.com/hg/img/eclipse_string_substitution.png)



## Embedded database ##

### Derby ###

- see http://blog.quadrilateral.ca/quadrilateral/entry/using_embedded_derby_in_tomcat

### H2 ###

## Embedded JMS provider ##

Five years ago you could get away with a database and an interface to it (be it web interface or fat/rich client). Today enterprise applications seldom live in an isolation and they have to integrate with existing applications using more or less standard mechanisms. JMS is an API that allows us (developers) to decouple system through messages that they exchange. Trough JMS we can connect otherwise independent applications in a fast and reliable way. Of course it's not the only way - we could also use web services to transfer data - but JMS has it's purpose, and we need it in a Tomcat based enterprise development environment.

Couple of years ago there weren't many open source and high quality JMS implementations available - today you can choose between open source JMS implementations like ActiveMQ, HornetQ or OpenMQ.

### ActiveMQ/Bitronix setup ###

PREREQUISITE: Make sure you have completed Bitronix setup tasks.

  * Copy activemq-core-{version}.jar into the ${TOMCAT\_HOME}/lib directory.
  * Copy activemq-listener-{version}.jar into the ${TOMCAT\_HOME}/lib directory.

Create file resources.properties in the ${TOMCAT\_HOME}/conf directory and add to it following snippet:

```
# activemq instance datasource
resource.ds1.className=org.apache.activemq.ActiveMQXAConnectionFactory
resource.ds1.uniqueName=jms/myqcf
resource.ds1.maxPoolSize=5
resource.ds1.driverProperties.brokerURL=vm:(broker:(tcp://localhost:6000)?persistent=false)?marshal=false
```

This allows you to create Bitronix managed Connection Factory to an embedded in memory activemq message broker. You will be able to access the same message broker through port 6000. Feel free to adjust parameters by your needs.

If you need more than one connection factory just create it the same way. It doesn't matter if you repeat the same brokerURL - there will be just one activemq broker active in the end. For example add the following lines into resources.properties file:

```
resource.ds2.className=org.apache.activemq.ActiveMQXAConnectionFactory
resource.ds2.uniqueName=jms/myqcf2
resource.ds2.maxPoolSize=5
resource.ds2.driverProperties.brokerURL=vm:(broker:(tcp://localhost:6000)?persistent=false)?marshal=false
```


Edit Tomcat configuration file server.xml, and after Bitronix listener add another one (the one that shuts down activemq):

```
  <!-- bitronix tomcat lifecycle listener -->
  <Listener className="bitronix.tm.integration.tomcat55.BTMLifecycleListener" />
  <!-- activemq tomcat lifecycle listener -->
  <Listener className="com.google.jtede.tomcat.listeners.activemq.ActiveMQListener"/>
```

In the same file (server.xml) under GlobalNamingResources section add:

```
    <Resource name="jms/qcf" auth="Container" type="javax.jms.ConnectionFactory" factory="bitronix.tm.resource.ResourceObjectFactory" uniqueName="jms/qcf" />
    <Resource name="jms/qcf2" auth="Container" type="javax.jms.ConnectionFactory" factory="bitronix.tm.resource.ResourceObjectFactory" uniqueName="jms/qcf2" />

    <Resource auth="Container" description="input queue" factory="org.apache.activemq.jndi.JNDIReferenceFactory" name="jms/queue.in" physicalName="MYQUEUE.IN" type="org.apache.activemq.command.ActiveMQQueue"/>
```

Edit Tomcat configuration file configuration.xml and add to it:
```
 <Transaction factory="bitronix.tm.BitronixUserTransactionObjectFactory" />

 <ResourceLink global="jms/qcf" name="jms/QCF" type="javax.jms.ConnectionFactory" />
 <ResourceLink global="jms/queue.in" name="jms/queue.in" type="javax.jms.Queue" />
 <ResourceLink global="jms/qcf2" name="jms/QCF2" type="javax.jms.ConnectionFactory" />
```



This will allow you to reference queue connection factories and queues in the web.xml file:

```
        <!-- JMS  connection factory ref -->
        <resource-ref>
                <description>Connection Factory</description>
                <res-ref-name>jms/QCF</res-ref-name>
                <res-type>javax.jms.QueueConnectionFactory</res-type>
                <res-auth>Container</res-auth>
        </resource-ref>

        <!-- 2nd JMS  connection factory ref -->
        <resource-ref>
                <description>Connection Factory</description>
                <res-ref-name>jms/QCF2</res-ref-name>
                <res-type>javax.jms.QueueConnectionFactory</res-type>
                <res-auth>Container</res-auth>
        </resource-ref>

        <!-- Queue ref -->
        <resource-ref>
                <res-ref-name>jms/queue.in</res-ref-name>
                <res-type>javax.jms.Queue</res-type>
                <res-auth>Container</res-auth>
        </resource-ref>
```


## CommonJ ##

If you're IBM WebSphere or Oracle WebLogic user you probably know about CommonJ API. For others here's a short description taken from inactive JSR-236:


> Concurrency Utilities for Java EE provides a simple, standardized API for using
> concurrency from application components without compromising container integrity
> while still preserving the Java EE platform's fundamental benefits.


Basically it gives you an API that allows you to execute your code concurrently. Read: instead of creating threads on your own (which is NOT recommended in a JEE) you ask CommonJ API to execute code on your behalf. Spring JMS uses it to implement message driven POJOs.

You can read more at [IBM WebSphere Developer Technical Journal: Develop high performance J2EE threads with WebSphere Application Server](http://www.ibm.com/developerworks/websphere/techjournal/0606_johnson/0606_johnson.html)
This API was pushed by IBM and BEA (bought by Oracle), but it was never finished nor widely accepted.

Fortunately there is an open source CommonJ implementation available at http://commonj.myfoo.de.


### CommonJ setup ###

Download commonj from http://commonj.myfoo.de

  * Copy commonj-twm.jar into ${TOMCAT\_HOME}/lib directory.
  * Copy commonj-tomcat.jar into ${TOMCAT\_HOME}/lib directory.


Into the Tomcat's configuration file context.xml add:

```
  <Resource
    name="wm/ServiceWorkManager"
    auth="Container"
    type="commonj.work.WorkManager"
    factory="de.myfoo.commonj.work.FooWorkManagerFactory"
    minThreads="1"
    maxThreads="5" />
```

Into the web.xml add:

```
  <resource-ref>
    <res-ref-name>wm/ServiceWorkManager</res-ref-name>
    <res-type>commonj.work.WorkManager</res-type>
    <res-auth>Container</res-auth>
    <res-sharing-scope>Shareable</res-sharing-scope>
  </resource-ref>
```

Now, in the spring configuration you can use something like:

```
    <jee:jndi-lookup id="workManager" jndi-name="wm/ServiceWorkManager" />

    <bean id="workManagerTaskExecutor" 
      class="org.springframework.scheduling.commonj.WorkManagerTaskExecutor">
      <property name="workManager" ref="workManager" />
    </bean>

    <jms:listener-container connection-factory="myqcf"
      task-executor="workManagerTaskExecutor"
      destination-resolver="jndiDestinationResolver"
      acknowledge="transacted"
      transaction-manager="transactionManager">
      <jms:listener destination="jms/queue.in" ref="myMdb" />
    </jms:listener-container>

    <!-- NOTE: sessionTransacted -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
      <property name="connectionFactory" ref="myqcf" />
      <property name="sessionTransacted" value="true" />
    </bean>
```


# Relevant links #

  * [Article on Atomikos site  that describes integration with Tomcat6](http://www.atomikos.com/Documentation/Tomcat6Integration33)
  * [Article by Vinod Singh that describes integration of Atomikos and Tomcat](http://blog.vinodsingh.com/2009/12/jta-transactions-with-atomkios-in.html)
  * [Article by Vinod Singh that describes how to share datasource among several applications in Tomcat using Atomikos](http://blog.vinodsingh.com/2010/01/share-datasource-among-several.html)
  * [ActiveMQ - VM transport reference](http://activemq.apache.org/vm-transport-reference.html)
  * [ActiveMQ - Fuse documentation - Introduction to the VM protocol](http://fusesource.com/docs/broker/5.3/connectivity_guide/)
  * [CommonJ API implementation](http://commonj.myfoo.de)
  * [How to use BTM as the transaction manager in Tomcat 6.x](http://docs.codehaus.org/display/BTM/Tomcat13)
  * [How To Implement a Distributed CommonJ WorkManager](http://jonasboner.com/2006/09/14/how-to-implement-a-distributed-commonj-workmanager.html)
  * [Using embedded Derby in Tomcat](http://blog.quadrilateral.ca/quadrilateral/entry/using_embedded_derby_in_tomcat)
  * [How to use JBoss Transactions in Spring](http://ingenious-camel.blogspot.com/2012/01/how-to-use-jboss-transactions-in-spring.html)