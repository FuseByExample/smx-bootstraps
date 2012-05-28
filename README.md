This projects contains a set of OSGi bundles that bootstrap the use of ServiceMix 4. It is intended as a starting point for the creation of additional bundles, and as a guide to using SMX feaures.

The Ping Pong bootstrap is used to show inter-bundle request-response communication using ActiveMQ. 

1. A Pinger defines a Camel route which periodically triggers a message to be sent and prints out the response.
1. A Ponger listens for ping messages on a known queue, and responds by invoking an OSGi Blueprint service defined in an implementation bundle to generate the response. 

Project layout
==============
The Maven projects contained within are as follows:

* `smxb-features` - Contains an XML features file used to install the rest of the bundles.
* `smxb-pinger` - Contains a bundle that periodically sends a ping message over the embedded ActiveMQ instance. Also exposes a RESTful web service on port 9090 that allows you you ping the service manually.
* `smxb-ponger` - Listens on the queue and replies after invoking an OSGi Blueprint service to generate a response. Contains an additional NMR-based route as an alternative mechanism for chatting with the pinger in the same ServiceMix instance.
* `smxb-ponger-service` - Contains the implementation of that service.

There is also an additional parent project `camel-bundle` that simplifies the Maven project configuration.

Prerequisites
=============
Set up ServiceMix by downloading the latest 4.4.1+ version from [FuseSource](http://fusesource.com/products/enterprise-servicemix/). The installation guide can be reached from the Documentation tab on that page. 

Ensure that Maven is set up on your system. 

Installation
============
Download this project and run

	smx-bootstraps> mvn clean install

Start up ServiceMix

	$SERVICEMIX_HOME> bin/servicemix console 
	
	 ____                  _          __  __ _      
	/ ___|  ___ _ ____   _(_) ___ ___|  \/  (_)_  __
	\___ \ / _ \ '__\ \ / / |/ __/ _ \ |\/| | \ \/ /
	 ___) |  __/ |   \ V /| | (_|  __/ |  | | |>  < 
	|____/ \___|_|    \_/ |_|\___\___|_|  |_|_/_/\_\
	
	  Apache ServiceMix (4.4.1-fuse-01-06)
	
	Hit '<tab>' for a list of available commands
	and '[cmd] --help' for help on a specific command.

You may need to wait for a while as Servicemix downloads and starts the bundles it needs to run fully. Running the `list` command should show around 200+ bundles all Active.

	karaf@root> list
	...
	[ 220] [Active     ] [            ] [       ] [   50] camel-jetty (2.8.0.fuse-01-06)
	karaf@root> _

If `camel-jetty` does not appear in the bundle list, install it; it is required to enable the Pinger web service:

	karaf@root> features:install camel-jetty

Install the `smxb-features` features file. This gets pulled out from your local Maven repository, and defines which bundles you mean to install for the bootstrap project.

	karaf@root> features:addurl mvn:com.fusesource.examples/smxb-features/1.0-SNAPSHOT/xml/features
	
You can now check that the features defined in that file are available for installation:

	karaf@root> features:list | grep smxb
	[uninstalled] [1.0                 ] smxb-ping-pong                       smxb-features-1.0-SNAPSHOT       
	[uninstalled] [1.0                 ] smxb-ping                            smxb-features-1.0-SNAPSHOT       
	[uninstalled] [1.0                 ] smxb-pong                            smxb-features-1.0-SNAPSHOT
	
_NP: It's often a good idea to prefix all of your features and bundles with a known string, such as `smxb` in this case, so you can easily find them via the grep command in the various listings._

Install all of the necessary OSGi bundles by installing the `smxb-ping-pong` feature

	karaf@root> features:install smxb-ping-pong
	karaf@root> list | grep smxb
	[ 236] [Active     ] [Created     ] [       ] [   60] smxb-pinger (1.0.0.SNAPSHOT)
	[ 237] [Active     ] [Created     ] [       ] [   60] smxb-ponger (1.0.0.SNAPSHOT)
	[ 238] [Active     ] [Created     ] [       ] [   60] smxb-ponger-service (1.0.0.SNAPSHOT)

You can then start the bundles (the numbers may vary in your installation)

	karaf@root> start 236
	karaf@root> start 237
	karaf@root> start 238

The log should now contain output from your bundles

	karaf@root> log:display -n 5
	13:03:11,760 | INFO  | msConsumer[pings | pong                             | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Received ping: Ping at 2011-12-30 13:03:11
	13:03:11,772 | INFO  | tenerContainer-1 | toActiveMQ                       | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Received AMQ: [Pong from service to [Ping at 2011-12-30 13:03:11]]
	13:03:16,752 | INFO  | foo              | pingOnTimer                      | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Timer generated Ping at 2011-12-30 13:03:16
	13:03:16,759 | INFO  | msConsumer[pings | pong                             | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Received ping: Ping at 2011-12-30 13:03:16
	13:03:16,768 | INFO  | tenerContainer-1 | toActiveMQ                       | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Received AMQ: [Pong from service to [Ping at 2011-12-30 13:03:16]]

Check that the web service is running by hitting the following from your web browser:

	http://localhost:9090/ping?msg=Ping!!!

Congratulations. You have just deployed and run a bootstrapped integration project!

Next steps
==========
Working with bundles
--------------------
Change the Camel route defined in `smxb-pinger/target/classes/OSGI-INF/blueprint/blueprint.xml`.
Run 

	smx-bootstraps> mvn clean install

In ServiceMix, update the `smxb-pinger` bundle

	karaf@root> list | grep smxb-pinger
	[ 236] [Active     ] [Created     ] [       ] [   60] smxb-pinger (1.0.0.SNAPSHOT)
	karaf@root> update 236

Your changes should now be visible.

Configuration
-------------
The `smxb-pinger` and `smxb-ponger` projects make use of OSGi Blueprints config services. From the same `blueprints.xml`:

	<cm:property-placeholder persistent-id="com.fusesource.examples.pinger"
		update-strategy="reload">
		<cm:default-properties>
			<cm:property name="broker.url" value="failover:(tcp://127.0.0.1:61616)" />
		</cm:default-properties>
	</cm:property-placeholder>
	...
    <bean id="activemq" class="org.apache.activemq.camel.component.ActiveMQComponent">
        <property name="brokerURL" value="${broker.url}" />
    </bean>
    
You can dynamically change the default configuration by creating a properties file in the `$SERVICEMIX_HOME/etc` directory called `{persistent-id}.cfg`, so in this case `com.fusesource.examples.pinger.cfg`. This is a standard properties file, within which you can override any of the properties used in your `blueprints.xml`.

Adding the following line to the file and saving it will stop the bundles that listen on that `persistent-id` and start them again.

	broker.url=failover:(tcp://127.0.0.1:61616,tcp://127.0.0.1:61617)

This assumes that the `cm:property-placheholders` are configured with `update-strategy="reload"`, if not the bundles need to be stopped and started manually.

The log should show the affected Camel routes stopping and starting:

	13:24:58,181 | INFO  | Thread-51        | BlueprintCamelContext            | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Apache Camel 2.8.0-fuse-01-06 (CamelContext:smxb-pinger) is shutting down
	13:24:58,182 | INFO  | Thread-51        | DefaultShutdownStrategy          | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Starting to graceful shutdown 2 routes (timeout 300 seconds)
	13:24:58,183 | INFO  | 8 - ShutdownTask | DefaultShutdownStrategy          | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Route: pingOnTimer shutdown complete, was consuming from: Endpoint[timer://foo?period=5000]
	13:24:58,183 | INFO  | 8 - ShutdownTask | DefaultShutdownStrategy          | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Route: toActiveMQ shutdown complete, was consuming from: Endpoint[direct://toActiveMQ]
	13:24:58,183 | INFO  | Thread-51        | DefaultShutdownStrategy          | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Graceful shutdown of 2 routes completed in 0 seconds
	13:24:58,185 | INFO  | Thread-51        | TemporaryQueueReplyManager       | ?                                   ? | 93 - org.apache.camel.camel-jms - 2.8.0.fuse-01-06 | Stopping reply listener container on endpoint: Endpoint[activemq://pings]
	13:24:58,973 | INFO  | Thread-51        | DefaultInflightRepository        | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Shutting down with no inflight exchanges.
	13:24:58,974 | INFO  | Thread-51        | BlueprintCamelContext            | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Uptime: 39.091 seconds
	13:24:58,974 | INFO  | Thread-51        | BlueprintCamelContext            | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Apache Camel 2.8.0-fuse-01-06 (CamelContext: smxb-pinger) is shutdown in 0.793 seconds
	13:24:58,990 | INFO  | rint Extender: 3 | BlueprintCamelContext            | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | JMX enabled. Using ManagedManagementStrategy.
	13:24:58,993 | INFO  | rint Extender: 3 | BlueprintCamelContext            | ?                                   ? | 85 - org.apache.camel.camel-core - 2.8.0.fuse-01-06 | Apache Camel 2.8.0-fuse-01-06 (CamelContext: smxb-pinger) is starting

Using this mechanism you can externalise any environment-specific config. If you like, define some properties to echo a different pong message and have a go at modifying them externally.
