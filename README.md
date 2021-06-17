## [IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)


## Module: Model Driven Telemetry

## Version: 17.6



# Model-Driven Telemetry

Network data collection for today s high-density platforms and scale is becoming a tedious task for monitoring and troubleshooting. There is a need for operational data from different devices in the network to be collected in a centralized location so that cross-functional groups can collaboratively work to analyze and fix an issue.

**Model-driven Telemetry** (MDT) provides a mechanism to stream data from an MDT-capable device to a destination. It uses a new approach for network monitoring in which data is streamed from network devices continuously using a push model and provides near real-time access to operational statistics for monitoring data. Applications can subscribe to specific data items they need by using standards-based YANG data models over open protocols. Structured data is published at a defined cadence or on-change, based upon the subscription criteria and data type.

There are two main MDT Publication/Subscription models, Dial-in and Dial-out:

**Dial-in** is a dynamic model. An application based on this model has to open a session to the network device and send one or more subscriptions reusing the same session. The network device will send the publications to the application for as long as the session stays up. **NETCONF** and **gNMI** are the Dial-In telemetry interfaces.

**Dial-out** is a configured model. The subscriptions need to be statically configured on the network device using any of the available interfaces (CLI, APIs, etc.), and the device will open a session with the application. If the session goes down, the device will try to open a new session. **gRPC** is the Dial-Out telemetry interface.

![](imgs/1-pubsub.png)

In this lab, we cover the **gRPC Dial-out with DNS** telemetry that was released in IOS XE 17.6. The open source TIG stack will also be described that is used to collect and visualize the telemetry data.

- **Telegraf (Collection)** with the **cisco\_telemetry\_mdt** plugin that decodes the gRPC data to text
- **InfluxDB (Storage)**: an open-source time series database optimized for fast, high-availability storage and retrieval of time series data in fields such as operations monitoring, application metrics, Internet of Things sensor data, and real-time analytics. It provides a SQL-like language with built-in time functions for querying a data structure composed of measurements, series, and points
- **Grafana (GUI visualization)**: an open-source platform to build monitoring and analytics dashboards

Every LAB POD includes a full installation of all the software mentioned above.


# gRPC Dial-Out with FQDN

Insted of using IP address, now the DNS name can be used when configuring the telemetry receiver.

## Configure gRPC Dial-Out

There are two components to configur for the DNS based telemetry receiver. The **telemetry ietf sub** section remains similar to previous versions, however the **receiver-type** and **receiver name** CLI's are new. These are used to define the host and port that the telemetry subscriptions use.

```
conf t
no telemetry ietf subscription 1010
telemetry ietf subscription 1010
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
 source-address 10.1.1.5
 stream yang-push
 update-policy periodic 2000
 receiver-type protocol
 receiver name yangsuite

telemetry receiver protocol yangsuite
 host name yangsuite-telemetry.cisco.com 57500
 protocol grpc-tcp

```

Copy and paste the gRPC configuration into the C9300.



### Verify

The **show telemetry** CLI command can be used to see the **State** of the secure telemetry connection. It should be in the **Connected** state:

**show telemetry ietf subscription 1010 receiver**

```
C9300#show telemetry ietf subscription 1010 receiver
Telemetry subscription receivers detail:

  Subscription ID: 1010
  Name: yangsuite
  Connection: 0
  State: Connected
  Explanation:


C9300#
```

Use the show details CLI below to see the details of the subscription: **show telemetry ietf subscription 1010 detail**

```
C9300#show telemetry ietf subscription 1010 detail
Telemetry subscription detail:

  Subscription ID: 1010
  Type: Configured
  State: Valid
  Stream: yang-push
  Filter:
    Filter type: xpath
    XPath: /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
  Update policy:
    Update Trigger: periodic
    Period: 2000
  Encoding: encode-kvgpb
  Source VRF:
  Source Address: 10.1.1.5
  Receiver Type: protocol
  Notes:

  Named receivers:
    Name                              Last State Change  State                 Explanation
    ---------------------------------------------------------------------------------------------------------
    yangsuite                         06/15/21 19:55:03  Connected


C9300#
```



Proceed to the Grafana GUI, where the telemetry data is visualized
## Telegraf, Influx, Grafana (TIG)

![](imgs/4-mdt-solution.png)

Telegraf is the tool that receives and decodes the telemetry data that is sent from the IOS XE devices. It processes the data and sends it into the InfluxDB datastore, where Grafana can access it to create visualizations.

Telegraf runs inside the  "tig_mdt" Docker container. To connect to this container from the Ubuntu host, follow the steps below:

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

**Output Plugin:** This defines where the received data is sent to (outputs.influxdb), the database to use (telegraf), and the URL for InfluxDB ([http://127.0.0.1:8086](http://127.0.0.1:8086/))

**Outputs.file** : sends a copy of the data to the text file at /root/telegraf/telegraf.log

These configuration options are defined as per the README file in each respective input or output plugins. For more details of the cisco_telemetry_mdt plugin that is in use here, see the page at ["https://github.com/influxdata/telegraf/tree/master/plugins/inputs/cisco_telemetry_mdt"]("https://github.com/influxdata/telegraf/tree/master/plugins/inputs/cisco_telemetry_mdt")

Examining the output of the telegraf.log file shows the data coming in from the IOS XE device that matches the subscription we created and do ctrl+c to stop the output.

**# tail -F /tmp/telegraf-grpc.log**

![](imgs/7-cat_telegraf_grpc.png)

## The Influx Database InfluxDB 

InfluxDB is already installed and started within the same Docker container. Let's verify it's working correctly by connecting to the Docker contain where it is running.

Step 1. Verify InfluxDB is running with the command **ps xa | grep influx**

```
15 pts/0 Sl+ 1:45 /usr/bin/influxd -pidfile /var/run/influxdb/influxd.pid -config /etc/influxdb/influxdb.conf
```

Step 2. Verify the data stored on the Influx database using the command shown below:

```
root@43f8666d9ce0:~# influx
Connected to http://localhost:8086 version 1.7.7
InfluxDB shell version: 1.7.7
> show databases
name: databases
name
----
_internal
mdt_gnmi
mdt_grpc
mdt_grpc_tls
mdt_netconf
>
> use mdt_grpc
Using database mdt_grpc
> show measurements
name: measurements
name
----
Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
>
> SELECT COUNT("five_seconds") FROM "Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization"
name: Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
time count
---- -----
0    1134
>
```
The output above shows:

- a **telegraf** database as defined in the Telegraf config file, which holds that telemetry data
- one measurement defined as the YANG model used for the gRPC Dial-out subscription (Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization)
- number of publications received so far (33251).

![](imgs/8-influx.png)

# Grafana Dashboard

Grafana is an open-source platform to build monitoring and analytics dashboards that also runs within the Docker container. Navigating to the web-based user interface allows us to see the dashboard with the Model Driven Telemetry data.

Verify Grafana is running: with the following command: **ps xa | grep grafana**

```
44 ? Sl 0:32 /usr/sbin/grafana-server --pidfile=/var/run/grafana-server.pid --config=/etc/grafana/grafana.ini --packaging=deb cfg:default.paths.provisioning=/etc/grafana/provisioning cfg:default.paths.data=/var/lib/grafana cfg:default.paths.logs=/var/log/grafan cfg:default.paths.plugins=/var/lib/grafana/plugins**
```

## Visualize gRPC Dial-Out with Grafana

Open the Firefox browser and navigate to the Grafana tab or shortcut.

The **CPU Utilization** streaming telemetry data that was configured earlier is now visible in the pre-configured chart as seen below

This shows this telemetry data that was configured earlier in this lab using Grafana for visualization of the data.

![](./imgs/grafana-secure.png)



# Conclusion

This completes the gRPC Dial-Out with DNS section of the lab module. Refer to the previous lab guide, linked below, that covers gRPC telemetry in more detail, as well as Telegraf, InfluxDB, and Grafana - including the tig_mdt Docker container. This lab modules focus is on the gRPC-Dial Out + FQDN/DNS telemetry connection, while the previous lab guide covers each of the Model Driven Telemetry (MDT) interfaces (NETCONF, gRPC, gNMI) in greater detail.

[https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab-module-6-mdt/blob/master/README-172.md](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab-module-6-mdt/blob/master/README-172.md)
