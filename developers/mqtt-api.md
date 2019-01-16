---
description: How to access the aedifion.io MQTT API
---

# MQTT API

## Overview

MQTT is a lightweight publish/subscribe messaging protocol designed. Originally designed for machine to machine telemetry in low bandwidth environments \(M2M\), MQTT has nowadays become one of the main protocols for \(data collection in\) Internet of Things \(IoT\) deployments \[1\].

MQTT has multiple advantages over HTTP \[3\] and other protocols: 

* Due to its binary encoding and minimal packet overhead, MQTT is 20x faster than HTTP, uses 50x less traffic, and consumes 20% less energy than HTTP according to the direct performance comparison presented in \[4\]. 
* Contrary to HTTP's client/server architecture, MQTT's publish/subscribe pattern decouples data sources from data sinks through a third party, the MQTT broker. Decoupling means that sources never directly talk to sinks which implies that i\) they do not need to know each other, ii\) they do not need to run at the same time, and iii\) they do not require synchronization \[5\]. All this allows easily allows building flexible 1-to-many, many-to-1, and many-to-many data pipelines.
* MQTT has message filtering built-in, i.e., data sinks can subscribe to arbitrary subsets of the data collected from the sources specified through a hierarchical topic theme \[6\].

When you are building an application that streams data into or from the aedifion.io platform, MQTT is probably a better choice than the [HTTP API](api-documentation.md).

**Sources and further resources:**  
\[1\] Introduction to MQTT: [http://www.steves-internet-guide.com/mqtt/](http://www.steves-internet-guide.com/mqtt/)  
\[2\] Introduction to MQTT: [https://www.hivemq.com/blog/mqtt-essentials-part-1-introducing-mqtt/](https://www.hivemq.com/blog/mqtt-essentials-part-1-introducing-mqtt/)  
\[3\] MQTT vs. HTTP: [https://iotdunia.com/mqtt-and-http/](https://iotdunia.com/mqtt-and-http/)  
\[4\] MQTT vs. HTTP: [https://flespi.com/blog/http-vs-mqtt-performance-tests](https://flespi.com/blog/http-vs-mqtt-performance-tests)  
\[5\] MQTT publish/subscribe: [https://www.hivemq.com/blog/mqtt-essentials-part2-publish-subscribe/](https://www.hivemq.com/blog/mqtt-essentials-part2-publish-subscribe/)  
\[6\] MQTT topics: [https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/](https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/)

## The aedifion.io MQTT API

Part of the aedifion.io platform is a MQTT broker which serves as the single logical point of data ingress to the aedifion.io, e.g., all data collected in the field through the aedifion edge devices is ingested to aedifion.io through this MQTT broker and, in turn, can also be subscribed to. The MQTT broker is clustered, i.e., distributed over multiple independent servers, to ensure seamless scalability and high availability.

### MQTT Broker 

The aedifion.io MQTT broker is reachable at mqtt2.aedifion.io. The broker only accepts TLS 1.2 encrypted connections, i.e., all plain TCP connections are rejected. The brokers certificate can be viewed, e.g., by connecting to [https://mqtt2.aedifion.io](https://mqtt2.aedifion.io) from any browser.

![Server MQTT certificate of MQTT broker.](../.gitbook/assets/mqtt_certificate.png)

Your operating system \(OS\) will accept this certificate if the DST Root CA X3 certificate is installed as a trusted root certification authority \(CA\) in your OS \[1\]. Don't worry, this is probably the case and you don't have to do anything. You can test if the certificate is accepted by navigating to [https://mqtt2.aedifion.io](https://mqtt2.aedifion.io) - if your browser doesn't issue a warning, you're fine.

The broker accepts connections on two ports: 8884 and 9001. Port 8884 accepts _plain_ MQTT, i.e., connections that transport MQTT directly via TLS. This is the standard case and, if in doubt, this port is the right choice. Port 9001 accepts websockets connections, i.e., connections that transport the MQTT protocol within the websockets protocol which in turn is transported via TLS. MQTT via websockets is the right choice when you want to send/receive MQTT data directly from a web browser \[2,3\].

**Summary:**

* Url: **mqtt2.aedifion.io**
* Ports:
  * **8884** - MQTT over TLS 1.2 \(use in standalone clients\)
  * **9001** - MQTT over websockets over TLS 1.2 \(use in browsers\)
* Test the connection:
  * Point a browser to [https://mqtt2.aedifion.io](https://mqtt2.aedifion.io)

    No warnings should occur.

  * If you have openssl installed, open a command line and execute

    ```text
    openssl s_client -connect mqtt2.aedifion.io:8884 -servername mqtt2.aedifion.io
    ```

    Check that the TLS handshake is ok.

    * There will be warning `verify error:num=20:unable to get local issuer certificate` at the top which can be fixed by providing option `-CAfile` or `-CApath` and pointing to the right locations depending on your OS, e.g., `-CAfile /etc/ssl/cert.pem` on Mac OS.



**Sources and further resources:**  
\[1\] Let's enrypt's chain of trust: [https://letsencrypt.org/certificates/](https://letsencrypt.org/certificates/)  
\[2\] MQTT over websockets: [https://www.hivemq.com/blog/mqtt-essentials-special-mqtt-over-websockets/](https://www.hivemq.com/blog/mqtt-essentials-special-mqtt-over-websockets/)  
\[3\] MQTT over websockets: [http://www.steves-internet-guide.com/mqtt-websockets/](http://www.steves-internet-guide.com/mqtt-websockets/)

### Client Authentication and Authorization 

The MQTT broker only accepts connections from _authenticated_ clients and additionally _authorizes_ clients based on the topics that they read from or write to. Authentication and authorization is a separated two-step process.

#### **Authentication**

After having established a TLS connection, the MQTT client has to present login credentials \(`username` and `password`\) to the MQTT broker. Client credentials can be obtained in two ways:

* Credentials with unlimited validity are provided only on request by the aedifion staff.

  Please email us at support@aedifion.io.

* Temporary credentials with limited validity can be created through the aedifion.io [HTTP API](api-documentation.md) using the `POST /v2/project/{project_id}/mqttuser` endpoint.

On connect, clients also have to provide a `client_id`. A prefix for the `client_id` will be assigned by aedifion and you are free to choose any prefix you like. It is, however, important to note that a new connection with an existing `client_id` will disconnect the older connection on the same `client_id`. While you can just open multiple connections using the same login credentials, you must use a different `client_id` for each parallel connection. It is good practice to append a random integer to your `client_id` as part of your individual postfix.

#### **Authorization**

Once connected and authenticated, the client can publish or subscribe to one or multiple topics - but not without authorization. To subscribe, the client needs _read access_ to that topic. To publish, the client needs _write access_. Note that _write access_ implies _read access_.

Authorization is specified through a list of topics \(following exactly MQTT's topic syntax and semantics \[1,2\]\) where for each topic is specified whether the user has read or read/write access. Make sure to familiarize yourself with MQTT's topic structure, especially with topic hierarchy levels and the `#` wildcard \[1,2\].

### Topic hierarchy 

All MQTT topics on aedifion.io have a hierarchy that consists of two main parts, i.e., a fixed prefix and a variable postfix.

The prefix has two hierarchies and is assigned by aedifion:

```text
load-balancing-group/project-handle/
```

* The top level hierarchy, the `load-balancing-group`, is fixed and assigned by aedifion.

  It serves to separate different customers and projects and ensures that each customer and project is guaranteed separate and sufficient processing and communication resources.

* The second level hierarchy, the `project-handle`, is a fixed \(human-readable\) string assigned by aedifion that uniquely identifies your project.

  This hierarchy separates different projects on the same load balancing group.

As an aedifion.io customer, you receive authorization to this prefix, i.e., to the topic `load-balancing-group/project-handle/#`, i.e., you can publish and subscribe to "_anything below_" the project level of the topic hierarchy.

The postfix matching the `#` can generally have arbitrary length or structure as long as they are UTF-8 strings.

* If you've purchased an aedifion edge device, e.g., this device collects data from different datapoints on your building network and publishes them to the postfixes `datapoint_1`, ..., `datapoint_n`.

  Via MQTT, you thus have datapoint-level publish/subscribe access to the datapoints of your building.

  For efficiency reasons, the edge device uses short 4 to 12 characters long base62-encoded hash identifiers generated from the full datapoint names.

* If you ingest data yourself, you can publish to arbitrary postfixes since the `#` wildcard of your topic authorization matches any number of sublevels.

It is important to note two things about publishing your own data via MQTT:

1. The postfix is only used for routing messages on the broker, e.g., you can use it to group data for different subscribers.

   The postfix does _not_ determine which time series data is stored to.

   This is determined by the payload of your messages \(see below\).

2. aedifion does not prevent you from writing data to datapoints that are at the same time written by the aedifion edge device.

   If you have a `datapoint_A` on your local building network that is discovered by the edge device and you also write to  `datapoint_A` yourself, this data will be stored and intermingled in the same time-series.

### Payload format 

All messages you publish to or receive from the MQTT broker must adhere to strictly to the following format:

```text
RoomTemperature,location=office20.32,unit=C value=20.3 1465839830100400200
    |          ---------------------------- ---------- -------------------
    |                             |             |           |
    |                             |             |           |
+---------+----------------------------+-+-----------+-+---------+
|datapoint|,tag_set....................| |observation| |timestamp|
+---------+----------------------------+-+-----------+-+---------+
```



This is known as _Influx Line Protocol_ and specified in detail at \[1\]. We highlight only the most important points in the following:

* `datapoint` is an arbitrary non-empty UTF-8 string that identifies your datapoint.

  If it contains blank spaces or quotes, then you must quote the string and escape blanks as well as quotes using a backslash `\`, e.g., `"this\ is\ a\ \"datapoint\"\ with\ spaces and\ quotes"`.

  The reported `observation` will be stored on aedifion.io in a time series with exactly this name as you will see when logging in to the [frontend](https://www.aedifion.io).

* `tag_set` is separated by a `,` from the `datapoint id`.

  It is itself a comma-separted list of `key=value` pairs that are attached to this reported measurement. The `tag_set` can be empty.

* `observation` is the reported measurement and must have the form of `value=<float>` where `<float>` is parsable as a floating point number. `observation` must be separated from the \(potentially empty\) `tag_set` by a single blank character.
* `timestamp` is the timestamp of your reported observation in nanosecond-precision Unix time, i.e., the number of nanoseconds since January 1st, 1970, 00:00:00 UTC.

  It must be separated from the `observation` by a single blank.

  If your timestamp is in millisecond or microsecond precision you must append 6 or 3 zeros, respectively.

**Note:** Observations from messages that do not strictly adhere to this format will still be received from the MQTT broker but will not be stored in the aedifion.io platform.

**Sources and further resources:**  
\[1\] [https://docs.influxdata.com/influxdb/v1.6/write\_protocols/line\_protocol\_tutorial/](https://docs.influxdata.com/influxdb/v1.6/write_protocols/line_protocol_tutorial/)

### Fair use 

We as aedifion give our best to ensure seamless scalability and highest availability of our MQTT services. Since we give priority to a clean and simple user experience, we currently do not enforce any rate limits on the MQTT ingress and egress. Deliberately, this allows you to send bursts of data, e.g., to import a batch of historical data.

This being said, we will negotiate a quota with each customer that we consider the basis of fair use of our MQTT services. In favor of your user experience, this quota will be monitored but not strictly enforced. aedifion reserves the right to technically enforce the fair use quota on repeated violations without prior notice.

## Examples 

Explore our step-by-step tutorial on [streaming data from our MQTT broker via websockets](../tutorials/mqtt/websockets.md) directly into a webpage using client-side javascript.

aedifion provides examples for subscribing and publishing to the aedifion.io MQTT broker on request for the following languages and tool sets:

* [Eclipse Paho MQTT Python library](https://pypi.org/project/paho-mqtt/)
* [Mosquitto command line tools](https://mosquitto.org/download/)
* [MQTT.fx](https://mqttfx.jensd.de/)
* [Node-RED](https://nodered.org/)
* [Matlab](https://www.mathworks.com/help/thingspeak/mqtt-api.html)
* [Docker](https://www.docker.com/)
* ...
