In this article, I'd like to show you how to setup JBoss AS7 in domain mode and enable clustering so we could get HA and session replication among the nodes. It's a step to step guide so you can follow the instructions in this article and build the sandbox by yourself :-)

## Preparation & Scenario

### Preparation

We need to prepare two hosts (or virtual hosts) to do the experiment. We will use these two hosts as following:

* Make sure that they are in same local network
* Make sure that they can access each other via different TCP/UDP ports(better turn off firewall and disable SELinux during the experiment or they will cause network problems).

### Scenario

Here are some details on what we are going to do:

* Let's call one host as 'master', the other one as 'slave'.
* Both master and slave will run AS7, and master will run as domain controller, slave will under the domain management of master.
* Apache httpd will be run on master, and in httpd we will enable the mod\_cluster module. The as7 on master and slave will form a cluster and discovered by httpd.
* We will deploy a demo project into domain, and verify that the project is deployed into both master and slave by domain controller. Thus we could see that domain management provide us a single point to manage the deployments across multiple hosts in a single domain.
* We will access the cluster URL and verify that httpd has distributed the request to one of the as7 host. So we could see the cluster is working properly.
* We will try to make a request on cluster, and if the request is forwarded to master as7, we then kill the as7 process on master. After that we will go on requesting cluster and we should see the request is forwarded to slave, but the session is not lost. Our goal is to verify the HA is working and sessions are replicated.
* After previous step finished, we reconnect the master as7 by restarting it. We should see the master as7 is registered back into cluster, also we should see slave as7 sees master as7 as domain controller again and connect to it.

Please don't worry if you cannot digest so many details currently. Let's move on and you will get the points step by step.

## Download JBoss AS7

First we should download the AS7 from website:

	http://www.jboss.org/jbossas/downloads/

The version I downloaded is 7.1.0.CR1b, please don't use the version prior to this one, or you will meet this bug when running in clustering mode:

	https://issues.jboss.org/browse/AS7-2738  

After download finished, I got the zip file:

	jboss-as-7.1.0.CR1b.zip

Then I unzipped the package to master and try to make a test run:

	unzip jboss-as-7.1.0.CR1b.zip
	cd jboss-as-7.1.0.CR1b/bin
	./domain.sh

If everything ok we should see AS7 successfully startup in domain mode:

	jboss-as-7.1.0.CR1b/bin$ ./domain.sh 
	=========================================================================

	  JBoss Bootstrap Environment

	  JBOSS_HOME: /Users/weli/Downloads/jboss-as-7.1.0.CR1b

	  JAVA: /Library/Java/Home/bin/java

	  JAVA_OPTS: -Xms64m -Xmx512m -XX:MaxPermSize=256m -Djava.net.preferIPv4Stack=true -Dorg.jboss.resolver.warning=true -Dsun.rmi.dgc.client.gcInterval=3600000 -Dsun.rmi.dgc.server.gcInterval=3600000 -Djboss.modules.system.pkgs=org.jboss.byteman -Djava.awt.headless=true

	=========================================================================
	...
	
	[Server:server-two] 16:59:55,870 INFO  [org.jboss.as] (Controller Boot Thread) JBoss AS 7.1.0.CR1b "Flux Capacitor" started in 2499ms - Started 148 of 214 services (64 services are passive or on-demand)

Now exit as7 and let's repeat the same steps on slave host. Finally we get AS7 run on both master and slave, then we could move on to next step.

## Domain Configuration

### Interface config on master

In this section we'll setup both master and slave for them to run in domain mode. And we will configure master to be the domain controller.

First open the host.xml in master as7 for editing:

	vi domain/configuration/host.xml

The default settings for interface in this file is:

	<interfaces>
	    <interface name="management">
	        <inet-address value="${jboss.bind.address.management:127.0.0.1}"/>
	    </interface>
	    <interface name="public">
	       <inet-address value="${jboss.bind.address:127.0.0.1}"/>
	    </interface>
	</interfaces>

We need to change the address to public so slave could connect to master. My master's ip address is 10.211.55.7, so I change the config to:

	<interfaces>
	    <interface name="management">
	        <inet-address value="${jboss.bind.address.management:10.211.55.7}"/>
	    </interface>
	    <interface name="public">
	       <inet-address value="${jboss.bind.address:10.211.55.7}"/>
	    </interface>
	</interfaces>

### Interface config on slave

Now we will setup interfaces on slave. First we need to remove the domain.xml from slave, because slave will not act as domain controller and it will under the management of master. I just rename the domain.xml so it won't be processed by as7:

	mv domain/configuration/domain.xml domain/configuration/domain.xml.move

Then let's edit host.xml. Similar to the steps on master, open host.xml first:

	vi domain/configuration/host.xml


The configuration we'll use on slave is a little bit different, because we need to let slave as7 connect to master as7. First we need to set the hostname. We change the name property from:

	<host name="master" xmlns="urn:jboss:domain:1.1">

to:

	<host name="slave" xmlns="urn:jboss:domain:1.1">


Then we need to modify domain-controller section so slave as7 can connect to master's management port:

	<domain-controller>  
	   <remote host="10.211.55.7" port="9999"/>  
	</domain-controller>

As we know, 10.211.55.7 is the ip address of master.

Finally, we also need to configure interfaces section and expose the management ports to public address:

	<interfaces>
	    <interface name="management">
	        <inet-address value="${jboss.bind.address.management:10.0.1.2}"/>
	    </interface>
	    <interface name="public">
	       <inet-address value="${jboss.bind.address:10.0.1.2}"/>
	    </interface>
	</interfaces>

10.0.1.2 is the ip address of the slave.

---

Note: I suggest to turn off firewall and disable SELinux on both master and slave during testing to ensure master and slave can communicate with each other correctly.

---

### Security Configuration

If you start as7 on both master and slave now, you will see the slave as7 cannot be started with following error:

	[Host Controller] 20:31:24,575 ERROR [org.jboss.remoting.remote] (Remoting "endpoint" read-1) JBREM000200: Remote connection failed: javax.security.sasl.SaslException: Authentication failed: all available authentication mechanisms failed
	[Host Controller] 20:31:24,579 WARN  [org.jboss.as.host.controller] (Controller Boot Thread) JBAS010900: Could not connect to remote domain controller 10.211.55.7:9999
	[Host Controller] 20:31:24,582 ERROR [org.jboss.as.host.controller] (Controller Boot Thread) JBAS010901: Could not connect to master. Aborting. Error was: java.lang.IllegalStateException: JBAS010942: Unable to connect due to authentication failure.

Because we haven't properly set up the authentication between master and slave. Now let's work on it:

#### Master

In bin directory there is a script called add-user.sh, we'll use it to add new users to the properties file used for domain management authentication:

	./add-user.sh   

	Enter the details of the new user to add.
	Realm (ManagementRealm) : 
	Username : admin
	Password : 123123
	Re-enter Password : 123123
	The username 'admin' is easy to guess
	Are you sure you want to add user 'admin' yes/no? yes
	About to add user 'admin' for realm 'ManagementRealm'
	Is this correct yes/no? yes
	Added user 'admin' to file '/home/weli/projs/jboss-as-7.1.0.CR1b/standalone/configuration/mgmt-users.properties'
	Added user 'admin' to file '/home/weli/projs/jboss-as-7.1.0.CR1b/domain/configuration/mgmt-users.properties'

As shown above, we have created a user named 'admin' and its password is '123123'. Then we add another user called 'slave':

	./add-user.sh   

	Enter the details of the new user to add.
	Realm (ManagementRealm) : 
	Username : slave
	Password : 123123
	Re-enter Password : 123123
	About to add user 'slave' for realm 'ManagementRealm'
	Is this correct yes/no? yes
	Added user 'slave' to file '/home/weli/projs/jboss-as-7.1.0.CR1b/standalone/configuration/mgmt-users.properties'
	Added user 'slave' to file '/home/weli/projs/jboss-as-7.1.0.CR1b/domain/configuration/mgmt-users.properties'

We will use this user for slave as7 host to connect to master.

#### Slave

In slave we need to configure host.xml for authentication. We should change the security-realms section as following:

	<security-realms>
	   <security-realm name="ManagementRealm">  
	       <server-identities>  
	           <secret value="MTIzMTIz="/>  
	       </server-identities>  
	       <authentication>  
	           <properties path="mgmt-users.properties" relative-to="jboss.domain.config.dir"/>  
	       </authentication>  
	   </security-realm>  
	</security-realms>

We've added server-identities into security-realm, which is used for authentication host when slave tries to connect to master. Because the slave's host name is set as 'slave', so we should use the 'slave' user's password on master. In secret value property we have "MTIzMTIz=", which is the base64 code for '123123'. 

Then in domain controller section we also need to add security-realm property:

    <domain-controller>
       <remote host="10.211.55.7" port="9999" security-realm="ManagementRealm"/>
    </domain-controller>

So the slave host could use the authentication information we provided in 'ManagementRealm'.

#### Dry Run

Now everything is set for the two hosts to run in domain mode. Let's start them by running domain.sh on both hosts. If everything goes fine, we could see from the log on master:

	[Host Controller] 21:30:52,042 INFO  [org.jboss.as.domain] (management-handler-threads - 1) JBAS010918: Registered remote slave host slave





![alt text][id]

[id]: /path/to/img.jpg "Title"
