
# NiFi Site-to-Site Direct Streaming to Storm for Log Ingestion

## Index

* [Short Description](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#short-description)
* [Introduction](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#introduction)
* [Prerequisites](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#prerequisites)
* [Configuring and Creating Table in Hbase via Phoenix](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#configuring-and-creating-table-in-hbase-via-phoenix)
* [Configuring and Starting NiFi](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#configuring-and-starting-nifi)
* [Building a Flow in NiFi to fetch and parse nifi-app.log](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#building-a-flow-in-nifi-to-fetch-and-parse-nifi-applog)
* [Building Storm application jar with maven](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#building-storm-application-jar-with-maven)
* [Extending NiFi Flow to ingest data directly to Phoenix using PutSql processor](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#extending-nifi-flow-to-ingest-data-directly-to-phoenix-using-putsql-processor-work-in-progress)
* [References](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/README.md#references)

## Short Description

Sample Application for Log Ingestion with NiFi and Storm into Phoenix using NiFi Site-to-Site on HW Sandbox.

## Introduction

Using NiFi, data can be exposed in such a way that a receiver can pull from it by adding an Output Port to the root process group. 
For Storm, we will use this same mechanism - we will use the Site-to-Site protocol to pull data from NiFi's Output Ports. In this tutorial we learn to capture NiFi app log from the Sandbox and parse it using Java regex and ingest it to Phoenix via Storm or Directly using NiFi PutSql Processor.

## Prerequisites

1) Assuming you already have latest version of NiFi-1.x/HDF-2.x downloaded as zip file (HDF and HDP cannot be managed by Ambari on same nodes as of now) on to your HW Sandbox Version 2.5, else execute below after ssh connectivity to sandbox is established:

```
# cd /opt/
# wget http://public-repo-1.hortonworks.com.s3.amazonaws.com/HDF/centos6/2.x/updates/2.0.1.0/HDF-2.0.1.0-centos6-tars-tarball.tar.gz
# tar -xvf HDF-2.0.1.0-12.tar.gz
```
2) Storm, Zeppelin are Installed on your VM and started.

3) Hbase is Installed with phoeix Query Server.

4) Make sure Maven is installed, if not already, execute below steps:

```
# curl -o /etc/yum.repos.d/epel-apache-maven.repo https://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo
# yum -y install apache-maven
# mvn -version
```	
## Configuring and Creating Table in Hbase via Phoenix

1) Make sure Hbase components as well as phoenix query server is started.

2) Make sure Hbase is up and running and out of maintenance mode, below properties are set(if not set it and restart the services):
	
	- Enable Phoenix --> Enabled
	
	- Enable Authorization --> Off
3) Create Phoenix Table after connecting to phoenix shell (or via Zeppelin):
	
```
# /usr/hdp/current/phoenix-client/bin/sqlline.py sandbox.hortonworks.com:2181:/hbase-unsecure
```

4) Execute below in the Phoenix shell to create tables in Hbase:

```
CREATE TABLE NIFI_LOG( UUID VARCHAR NOT NULL, EVENT_DATE DATE, BULLETIN_LEVEL VARCHAR, EVENT_TYPE VARCHAR, CONTENT VARCHAR CONSTRAINT pk PRIMARY KEY(UUID));
CREATE TABLE NIFI_DIRECT( UUID VARCHAR NOT NULL, EVENT_DATE VARCHAR, BULLETIN_LEVEL VARCHAR, EVENT_TYPE VARCHAR, CONTENT VARCHAR CONSTRAINT pk PRIMARY KEY(UUID));

```
	
## Configuring and Starting NiFi

1) Open **nifi.properties** for updating configurations:

```
# vi /opt/HDF-2.0.1.0/nifi/conf/nifi.properties
```

2) Change NIFI http port to run on 9090 as default 8080 will conflict with Ambari web UI

```
# web properties #
nifi.web.http.port=9090
```

3) Configure NiFi instance to run site-to site by changing below configuration : add a port say 8055 and set "nifi.remote.input.secure" as "false"

```
# Site to Site properties #
nifi.remote.input.socket.port=8055
nifi.remote.input.secure=false
```

4) Now Start [Restart if already running for configuration change to take effect] NiFi on your Sandbox.

```
# /opt/HDF-2.0.1.0/nifi/bin/nifi.sh start
```
5) Make sure NiFi is up and running by connecting to its Web UI from your browser:

```
http://your-vm-ip:9090/nifi/ 
```
## Building a Flow in NiFi to fetch and parse nifi-app.log

1) Let us build a small flow on NiFi canvas to read app log generated by NiFi itself to feed to Storm:
	
2) Drop a  "**TailFile**" Processor to canvas to read lines added to "**/opt/HDF-2.0.1.0/nifi/logs/nifi-app.log**". Auto Terminate relationship Failure. 
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/TailFile.jpg)

3) Drop a  "**SplitText**" Processor to canvas to split the log file into separate lines. Auto terminate Original and Failure Relationship for now. Connect TailFile processor to SplitText Processor for Success Relationship.
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/SplitText.jpg)

4) Drop a  "**ExtractText**" Processor to canvas to extract portions of the log content to attributes as below:
	- BULLETIN_LEVEL:([A-Z]{4,5})
	- CONTENT:(^.*)
	- EVENT_DATE:([^,]*)
	- EVENT_TYPE:(?<=\[)(.*?)(?=\])
	
	Connect SplitText processor to ExtractText Processor for splits relationship.
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/ExtractText.jpg)

5) Drop an OutputPort to the canvas and Name it "**OUT**", Once added, connect "ExtractText" to the port for matched relationship. The Flow would look similar as below:
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/Storm-Flow.jpg)

6) Start the flow on NiFi and notice data is stuck in the connection before the output port "OUT"


## Building Storm application jar with maven

1) To begin with, lets clone the artifacts, feel free to inspect the dependencies and NiFiStormStreaming.java 

```
# cd /opt/
# git clone https://github.com/jobinthompu/NiFi-Storm-Integration.git
```

2) Feel free the inspect pom.xml to verify the dependencies.

```
# cd /opt/NiFi-Storm-Integration
# vi pom.xml
```
3) Lets rebuild Storm jar with artifacts (this might take several minutes).

```
# mvn package
```

4) Once the build is SUCCESSFUL, make sure the NiFiStormTopology-Uber.jar is generated in the target folder:

```
# ls -l /opt/NiFi-Storm-Integration/target/NiFiStormTopology-Uber.jar
```

5) Now let us go ahead and submit the topology in storm (make sure the NiFi flow created above is running before submitting topology).

```
# cd /opt/NiFi-Storm-Integration
# storm jar target/NiFiStormTopology-Uber.jar NiFi.NiFiStormStreaming &
```

6) Lets Go ahead and verify the topology is submitted on the Storm View in Ambari as well as Storm UI:

```
Ambari UI: http://your-vm-ip:8080
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/StormView.jpg)

```
Storm UI: http://your-vm-ip:8744/index.html
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/StormUI.jpg)

7) Lets Go back to the NiFi Web UI, if everything worked fine, the data which was pending on the port OUT will be gone as it was consumed by Storm.

8) Now Lets Connect to Phoenix and check out the data populated in tables, you can either use Phoenix sqlline command line or Zeppelin 

 - via phoenix sqlline
 
```
# /usr/hdp/current/phoenix-client/bin/sqlline.py localhost:2181:/hbase-unsecure 

SELECT EVENT_DATE,EVENT_TYPE,BULLETIN_LEVEL FROM NIFI_DIRECT WHERE BULLETIN_LEVEL='ERROR' ORDER BY EVENT_DATE
```

![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/sqlline.jpg)

 - via Zeppelin for better visualization 
 
```
Zeppelin UI: http://your-vm-ip:9995/
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/Zeppelin.jpg)

9) No you can Change the code as needed, re-built the jar and re-submit the topologies.


## Extending NiFi Flow to ingest data directly to Phoenix using PutSql processor (Work In Progress)

1) Lets go ahead and kill the storm topology from command-line (or from Ambari Storm-View or Storm UI) 

```
# storm kill NiFi-Storm-Phoenix
```
2) Log back to NiFi UI currently running the flow, and stop the entire flow.

3) Drop a RouteOnAttribute processor to canvas for Matched relation from ExtractText processor and configure it with below property and auto terminate unmatched relation.
```
DEBUG  : ${BULLETIN_LEVEL:equals('DEBUG')}
ERROR  : ${BULLETIN_LEVEL:equals('ERROR')}
INFO   : ${BULLETIN_LEVEL:equals('INFO')}	
WARN   : ${BULLETIN_LEVEL:equals('WARN')}
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/RouteOnAttribute.jpg)

4) Drop an AttributesToJSON processor to canvas with below configuration and connect RouteOnAttribute's DEBUG,ERROR,INFO,DEBUG relations to it.
```
Attributes List : uuid,EVENT_DATE,BULLETIN_LEVEL,EVENT_TYPE,CONTENT
Destination : flowfile-content
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/AttributesToJSON.jpg)

5) Create and enable DBCPConnectionPool with name "Phoenix-Storm" with below configuration:

```
Database Connection URL : jdbc:phoenix:sandbox.hortonworks.com:2181:/hbase-unsecure
Database Driver Class Name : org.apache.phoenix.jdbc.PhoenixDriver
Database Driver Location(s) : /usr/hdp/current/phoenix-client/phoenix-client.jar	
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/Phoenix-Storm.jpg)

6) Drop a ConvertJSONToSQL to canvas with below configuration, connect AttributesToJSON's success relation to it, auto terminate Failure relation for now after connecting to Phoenix-Storm DB Controller service.

![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/ConvertJSONToSQL.jpg)

7) Drop a ReplaceText processor canvas to update INSERT statements to UPSERT for Phoenix with below configuration, connect sql relation of ConvertJSONToSQL auto terminate original and Failure relation.

![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/ReplaceText.jpg)

8) Finally add a PutSQL processor with below configurations and connect it to ReplaceText's success relation and auto terminate all of its relations.

![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/PutSQL.jpg)

9) The final flow including both ingestion via Storm and direct to phoenix using PutSql is complete, it should look similar to below:

![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/Final_Flow.jpg)

10) Now go ahead and start the flow to ingest data to both Tables via storm and directly from NiFi.

11) Login back to Zeppelin to see if data is populated in the NIFI_DIRECT table.
```
%jdbc(phoenix)
SELECT EVENT_DATE,EVENT_TYPE,BULLETIN_LEVEL FROM NIFI_DIRECT WHERE BULLETIN_LEVEL='INFO' ORDER BY EVENT_DATE
```
![alt tag](https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/images/Zeppelin_Final.jpg)

* Too Lazy to create flow??? download flow [here] ( https://github.com/jobinthompu/NiFi-Storm-Integration/blob/master/resources/templates/Storm_NiFi_Phoenix.xml)

#### This completes the tutorial,  You have successfully:

* Installed and Configured HDF 2.0 on your HDP-2.5 Sandbox.
* Created a Data flow to pull logs and then to Parse it and mave it available on a Site-to-site enabled NiFi port.
* Created a Storm topology to consume data from NiFi via Site-to-Site and Ingest it to Hbase via Phoenix.
* (Directly Ingested Data to Phoenix with PutSQL Processor in NiFi with out using Storm)
* Viewed the Ingested data from Phoenix command line and Zeppelin


### References:

* [ bbende's - nifi-storm](https://github.com/apache/nifi/tree/master/nifi-external/nifi-storm-spout/src/main/java/org/apache/nifi/storm)


Thanks,

Jobin George
