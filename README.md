## [IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)


## Module: Model Driven Telemetry

## Topics Covered


# Model-Driven Telemetry

Network data collection for today s high-density platforms and scale is becoming a tedious task for monitoring and troubleshooting. There is a need for operational data from different devices in the network to be collected in a centralized location, so that cross-functional groups can collaboratively work to analyze and fix an issue.

**Model-driven Telemetry** (MDT) provides a mechanism to stream data from an MDT-capable device to a destination. It uses a new approach for network monitoring in which data is streamed from network devices continuously using a push model and provides near real-time access to operational statistics for monitoring data. Applications can subscribe to specific data items they need, by using standards-based YANG data models over open protocols. Structured data is published at a defined cadence or on-change, based upon the subscription criteria and data type.

There are two main MDT Publication/Subscription models, Dial-in and Dial-out:

**Dial-in** is a dynamic model. An application based on this model has to open a session to the network device and send one or more subscriptions reusing the same session. The network device will send the publications to the application for as long as the session stays up. **NETCONF** and **gNMI** are the Dial-In telemetry interfaces. 

**Dial-out** is a configured model. The subscriptions need to be statically configured on the network device using any of the available interfaces (CLI, APIs, etc.) and the device will open a session with the application. If the session goes down, the device will try to open a new session. **gRPC** is the Dial-Out telemetry interface.

![](imgs/1-pubsub.png)

In this lab we cover the **gRPC Dial-out** telemetry that was released in IOS XE 16.10 along with the open source software stack for collection and visualization:

- **Telegraf (Collection)** with the **cisco\_telemetry\_mdt** plugin that decodes the gRPC data to text
- **InfluxDB (Storage)**: an open-source time series database optimized for fast, high-availability storage and retrieval of time series data in fields such as operations monitoring, application metrics, Internet of Things sensor data, and real-time analytics. It provides a SQL-like language with built-in time functions for querying a data structure composed of measurements, series, and points
- **Grafana (GUI visualization)**: an open-source platform to build monitoring and analytics dashboards

Every LAB POD includes a full installation of all the above-mentioned software. 

The focus of this lab is on gRPC with TLS security certificates: 

![](./imgs/intro_grpc_tls.png)


# gRPC + TLS

The gRPC MDT interface was introduced in IOS XE 16.10 and in 17.2 support for TLS security encapsulation was added. This means that the gRPC telemetry data can be sent in un-encrypted in plain text, or encrypted using TLS certificates as will be explored further below.

Before looking at the gRPC-TLS feature configuration, the required TLS certificates need to be installed into IOS XE. See the previous verious of the lab guide for those details, it's a manual process that is error prone. This time the certificates can be loaded using the **gNOI cert.proto certificate management API** as detailed in the DayN/gNOI/cert.proto module. Refer to that module for details of the gNOI cert.proto.

## Load the certificates from the Ubuntu into the C9300 with the gnoi_cert tooling

From the Ubuntu linux shell enter the **gnmi-ssl** directory and then go into the **certs** folder - these are the certs to load into the gNMI API via gNOI cert.proto's bootstrapping feature. If DayN/gNOI/cert.proto module was already completed then this step has already been completed.

**cd ~/gnmi_ssl/certs**

**/home/auto/gnoi_cert -target_addr c9300:9339 -op provision  -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id gnxi-cert -state BC -country CA -ca ./rootCA.pem  -key ./rootCA.key**

```
/home/auto/gnoi_cert -target_addr c9300:9339 -op provision  -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id gnxi-cert -state BC -country CA -ca ./rootCA.pem  -key ./rootCA.key
```

### Load the grpc-dial-out-tls TLS certificate with gNOI

Follow the similar proceedure as above to use the gnoi_cert tooling to install the **grpc-dial-out-tls** certificate into the IOS XE truststore.

**cd ~/gnmi_ssl/certs-grpc-tls**

**/home/auto/gnoi_cert -target_addr c9300:9339 -op provision  -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id grpc-dial-out-tls -state BC -country CA -ca ./myca.cert  -key ./myca.key**

```
cd gnmi_ssl/certs
/home/auto/gnoi_cert -target_addr c9300:9339 -op provision  -target_name c9300 -alsologtostderr -organization "jcohoe org" -ip_address 10.1.1.5 -time_out=10s -min_key_size=2048 -cert_id grpc-dial-out-tls -state BC -country CA -ca ./myca.cert  -key ./myca.key
```

Use the **show crypto pki trustpoint | i Tr** CLI to see details of the newly installed certificates. The gnoi_cert "GET" operation can also be used to see the certificate details.

```
C9300#show crypto pki trustpoints | i Tr
Trustpoint CISCO_IDEVID_SUDI_LEGACY:
Trustpoint CISCO_IDEVID_SUDI_LEGACY0:
Trustpoint CISCO_IDEVID_SUDI:
Trustpoint CISCO_IDEVID_SUDI0:
Trustpoint SLA-TrustPoint:
Trustpoint TP-self-signed-107381648:
Trustpoint gnxi-cert4:
Trustpoint grpc-dial-out-tls:
```

Continue to the next section to enable gRPC-TLS using this newly installed certificate.

## Configure gRPC-TLS

The main difference when creating a gRPC-TLS configuration is to specify the **grpc-tls profile** as part of the receiver IP address configuration, specifically:

receiver ip address 10.1.1.3 57500 **protocol grpc-tls profile grpc-dial-out-tls**

An completed exammple gRPC-TLS configuration is shown below

```
conf t
no telemetry ietf subscription 201
telemetry ietf subscription 201
encoding encode-kvgpb
filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
source-address 10.1.1.5
stream yang-push
update-policy periodic 6000
receiver ip address 10.1.1.3 57501 protocol grpc-tls profile grpc-dial-out-tls
end
```

Copy and paste the gRPC-TLS configuration into the C9300:



### Verify 

The **show telemetry** CLI command can be used to see the **State** of the secure telemetry connection. It should be in the **Connected** state:

**show telemetry ietf subscription 201 receiver**

```
C9300#show telemetry ietf subscription 201 receiver
Telemetry subscription receivers detail:

  Subscription ID: 201
  Address: 10.1.1.3
  Port: 57501
  Protocol: grpc-tls
  Profile: grpc-dial-out-tls
  Connection: 25
  State: Connected
  Explanation:
```

Proceed to the Grafana GUI , where the telemetry data is visualized

## Visualize gRPC-TLS with Grafana

Open the Firefox browser and navigate to the Grafana tab or shortcut. 

The window on the **LEFT** indicates the non-secured telemetry connection, that has been sending data the whole time.

The window on the **RIGHT** indicates the secured telemetry connection, that has just come up as there is no historical mesurements for this counter.

![](./imgs/grafana-secure.png)


## Explore telegraf-grpc-tls.conf

The telegraf configuration file that is receiving the telemetry data on port 57502 is within the **tig_mdt** Docker container at **/root/telegraf/**

The related TLS certifcates have been preinstalled into the Docker container's Telegraf configuration, located at **/root/telegraf/ssl** 

Explore the 

![](./imgs/telegraf-grpc-tls.png)

Additional gRPC and Model Driven Telemetry configuration examples can be found on the Github page at [https://github.com/jeremycohoe/cisco-ios-xe-mdt](https://github.com/jeremycohoe/cisco-ios-xe-mdt)

# Conclusion

This completes the gRPC-TLS section of the lab module. 

The next section covers gRPC telemetry in more detail, as well as Telegraf, InfluxDB, and Grafana - including the tig_mdt Docker container. Do not complete this part of the lab module as it is out of scope for this VT.


# STOP HERE #

```

























```

# The next sections are out of scope for this VT

```




















```

# Ok continue... have fun

```








































```







# gRPC Dial-Out Configured Subscriptions - Deep Dive

Lets continue by checking the subscriptions configured on the Catalyst 9300.

Open a SSH connection to the Catalyst 9300 switch

Check the subscription configured on the device using the following IOS XE CLI

**C9300# show run | sec telemetry**

![](imgs/3-showrunsectel.png)

Lets analyze the main parts of the subscription configuration:

- telemetry ietf subscription 101 (subscription ID)
- encoding encode-kvgpb (Key-Value pair encoding)
- filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds (xpath)
- update-policy periodic 500 (period in 1/100 seconds, 5 secs here)
- receiver ip address 10.1.1.3 57500 protocol grpc-tcp (receivers IP, port and protocol)

This telemetry configuration has already been applied to the switch. However, if it needs to be re-applied the following can be used to easily copy/paste:

```
conf t
telemetry ietf subscription 101
encoding encode-kvgpb
filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
source-address 10.1.1.5
stream yang-push
update-policy periodic 500
receiver ip address 10.1.1.3 57500 protocol grpc-tcp

```

Step 3. Verify the configured subscription using the following **telemetry** IOS XE CLIs

**c9300# sh telemetry ietf subscription all**

```

Telemetry subscription brief

ID               Type        State       Filter type
-----------------------------------------------------
101              Configured  Valid       xpath

```


**c9300# sh telemetry ietf subscription 101 detail**

```
Telemetry subscription detail:

Subscription ID: 101
Type: Configured
State: Valid
Stream: yang-push
Filter:
Filter type: xpath
XPath: /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
Update policy:
Update Trigger: periodic
Period: 500
Encoding: encode-kvgpb
Source VRF:
Source Address: 10.1.1.5
Notes:

Receivers:
Address          Port             Protocol         Protocol Profil
------------------------------------------------------------------
10.1.1.3         57500            grpc-tcp

```

**c9300# sh telemetry ietf subscription 101 receiver**

```
Telemetry subscription receivers detail:

Subscription ID: 101
Address: 10.1.1.3
Port: 57500
Protocol: grpc-tcp
Profile:
State: Connected
Explanation:
```


The State should report **Connected**.

If that state does not show Connected, for example, if it is the  "Connecting " state, then simple remove and re-add the telemetry configuration before continuing with the next steps and troubleshooting:

```
conf t
no telemetry ietf subscription 101
telemetry ietf subscription 101
encoding encode-kvgpb
filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
source-address 10.1.1.5
stream yang-push
update-policy periodic 500
receiver ip address 10.1.1.3 57500 protocol grpc-tcp
```

Note: If the state does not show  "Connected" then ensure the Docker container with the Telegraf receiver is running correctly. Follow the next steps to confirm status of each component.


## Telegraf, Influx, Grafana (TIG)

![](imgs/4-mdt-solution.png)

Telegraf is the tool that receives and decodes the telemetry data that is sent from the IOS XE devices. It processes the data and sends it into the InfluxDB datastore, where Grafana can access it in order to create visualizations.

Telegraf runs inside the  "tig_mdt" Docker container. To connect to this container from the Ubuntu host follow the steps below:

```
auto@automation:~$ docker ps
```

![](imgs/5-docker_ps.png)

```
auto@automation:~$ docker exec -it tig_mdt /bin/bash

 <You are now within the Docker container>

# cd /root/telegraf
# ls
```

There is one file for each telemetry interface: **NETCONF**, **gRPC**, and **gNMI**. Review each file to understand which. YANG data is being collected by which interface.

```
# cat telegraf-grpc.conf
# cat telegraf-gnmi.conf
# cat telegraf-netconf.conf
```

![](imgs/6-docker_exec_cat_grpc.png)

Inside the Docker container navigate to the telegraf directory and review the configuration file and log by tailing the log file with the command **tail -F /tmp/telegraf-grpc.log** 

The **telegraf-grpc.conf** configuration file shows us the following:

**gRPC Dial-Out Telemetry Input:** This defines the telegraf plugin (cisco\_telemetry\_mdt) that is being used to receive the data, as well as the port (57500)

**Output Plugin:** This defines where the received data is sent to (outputs.influxdb) the database to use (telegraf) and the URL for InfluxDB ([http://127.0.0.1:8086](http://127.0.0.1:8086/))

**Outputs.file** : sends a copy of the data to the text file at /root/telegraf/telegraf.log

These configuration options are defined as per the README file in each of the respective input or output plugins. For more details of the cisco_telemetry_mdt plugin that is in use here, see the page at ["https://github.com/influxdata/telegraf/tree/master/plugins/inputs/cisco_telemetry_mdt"]("https://github.com/influxdata/telegraf/tree/master/plugins/inputs/cisco_telemetry_mdt")

Examining the output of the telegraf.log file shows the data coming in from the IOS XE device that matches the subscription we created and do ctrl+c to stop the output

**# tail -F /tmp/telegraf.log**

![](imgs/7-cat_telegraf_grpc.png)


# Grafana Dashboard

Grafana is an open-source platform to build monitoring and analytics dashboards that also runs within the Docker container. Navigating to the web based user interface allows us to see the dashboard with the Model Driven Telemetry data

Access the interface Grafana at [http://10.1.1.3:3000](http://10.1.1.3:3000/)

You should see the following dashboard after logging in with the credentials provided

![](imgs/9-grafana-grpc.png)


# Conclusion

This module has shown how to configure the gRPC Dial Out configured telemetry feature on IOS XE. Using the Docker container with the open-source Telegraf + InfluxDB + Grafana stack you were able to receive, store, and visualize the telemetry information.
	
