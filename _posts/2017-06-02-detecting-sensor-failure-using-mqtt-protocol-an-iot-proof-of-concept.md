---
layout: post
title:  "Detecting sensor failure using the MQTT protocol, an IoT Proof of Concept"
date:   2017-06-02 12:00:00 +0530
categories: iot, arduino
comments: false
---

## Motivation
IoT technology is beginning to witness wide scale adoption in the consumer as well as industrial environments. In a typical enterprise scenario, you may have a large number of inexpensive sensor nodes interacting with machinery and sending data to Gateway nodes. These Gateway nodes are responsible for aggregating sensor data before sending it to the back-end systems why may reside in the cloud. Detecting a failure in one or more of these inexpensive sensor nodes is critical in preempting catastrophic failures.

![Sensor Nodes with Gateways]({{ site.url }}/images/Node_Gateway.png)

_[Image Source][Nodegateway]_

## Breakdown
In the sections below, we will look at:

1. What is MQTT? 
2. Essential MQTT Concepts.
3. MQTT on the Arduino.
4. A simple Java client.

## 1) What is MQTT?
The FAQ page at mqtt.org provides an apt answer.

_"MQTT stands for MQ Telemetry Transport. It is a publish/subscribe, extremely simple and lightweight messaging protocol, designed for constrained devices and low-bandwidth, high-latency or unreliable networks. The design principles are to minimise network bandwidth and device resource requirements whilst also attempting to ensure reliability and some degree of assurance of delivery. These principles also turn out to make the protocol ideal of the emerging “machine-to-machine” (M2M) or “Internet of Things” world of connected devices, and for mobile applications where bandwidth and battery power are at a premium."_

In short, MQTT is a messaging protocol that's light weight, reliable and easy to implement. It was invented by IBM in 1999.

## 2) Essential MQTT Concepts
MQTT works over TCP/IP and uses a standard port 1883. An MQTT connection is always established between a client and a broker. No client can connect directly to another client.

#### Client

Any device from a small micro controller to a full fledge server, which is capable of running a MQTT library and can connect to a MQTT Broker, is considered a MQTT client. MQTT clients may be publishers or subscribers or both.

___In this PoC, the Arduino will be the publishing client and a Java program will act as a subscribing client.___

#### Broker (Gateway Node)

A MQTT Broker should typically be capable of handling thousands of concurrently connected MQTT clients. Brokers are responsible for receiving, filtering and sending messages to subscribed clients. Needless to say, brokers should be highly scalable, capable of integrating with back-end systems and resistant to failures. Mosquitto is an open source MQTT server, the Eclipse Paho project hosts an instance of the Mosquitto broker as a public sandbox.

___We will be using the freely available Eclipse Paho MQTT broker sandbox.___

#### Quality of Service (QoS)
MQTT is a reliable protocol and provides a QoS level agreement between the sender and the receiver of a message. The three levels are as follows.

- QoS 0 (At most once)
- QoS 1 (At least once)
- QoS 2 (Exactly once)

__QoS 0__ is often known as fire and forget, messages aren't acknowledged by the receiver nor stored by the sender for redelivery.

__QoS 1__ guarantees that each message will be delivered to the receiver, however the message may also be delivered more than once. Receivers return an acknowledgment ([PUBACK][PUBACK]) to the sender. Senders will store the message until it gets this acknowledgment. In the case of redelivery, a duplicate flag is set. This flag is ignored by the broker and client in the case of QoS 1.

__QoS 2__ guarantees that each message will be received only once. Once the received get a QoS 2 message, it sends back a "Publish Received" ([PUBREC][PUBREC]) packet and stores a reference to the packet identifier until it sends a "Publish Complete" ([PUBCOMP][PUBCOMP]) to the sender. This is important so that the message isn't processed more than once. 

Once the sender gets a PUBREC it can safely discard the initial publish, stores the PUBREC and responds with a "Publish release" ([PUBREL][PUBREL]). Once the receiver gets the PUBREL message, it will discard the stored state and respond with a PUBCOMP.

This two step flow makes QoS 2 the safest and slowest service level.


#### Retained Messages
A Retained Message is a normal MQTT message with the retained flag set to `true`. A broker will store a single retained message for each topic. Subscribing clients may match a topic using a pattern which may include wildcards. Retained messages are pushed to clients immediately after subscribing to a topic, they needn't wait until a new message is made available. 

It is important to note that a broker will store only the last retained message and the corresponding QoS for a topic.

#### Last Will Testament (LWT)
Last Will Testament (LWT) is an MQTT feature with is used to notify all clients which are subscribed to a topic that the publishing client has been disconnected abruptly. As per the [MQTT 3.1.1 specifications][mqttspec] the broker may distribute a LWT of a client in the following cases.

- A network or I/O error is detected by the server.
- The clients fails to communicate within the Keel Alive time.
- The client closes the network connection without sending the DISCONNECT package first.
- The server closes the network connection because of network error.

The LWT is set as part of the CONNECT message when a connection is established between the client and the broker.

___We will use the LWT feature along with Retained Messages and QoS 1 to notify subscribing clients when a publisher (in this case a sensor node) disconnects abruptly.___

### 3) MQTT on the Arduino
I followed [this][arduinomqtt] article by Nick O'Leary to install the MQTT library (PubSubClient) on the Arduino and to write some simple code.

The following snippet contains the initial setup required for the Ethernet client as well as defining the IP address to the MQTT broker, which is the Eclipse Paho Sandbox environment.

{% highlight c %}
// Update these with values suitable for your network.
// These values are used by the Ethernet shield
byte mac[]    = {  0xDE, 0xED, 0xBA, 0xFE, 0xFE, 0xEE };
IPAddress ip(192, 168, 0, 111);

// Eclipse Paho Mqtt broker sandbox url "iot.eclipse.org" resolved to IP address
IPAddress brokerServer(198, 41, 30, 241);
EthernetClient ethClient;
PubSubClient mqttClient(ethClient);
{% endhighlight %}

Next we will need our LWT parameters, namely the QoS, the will topic, the will message and the retained flag. As the retained flag is set to `true`, any new client subscribing to the status topic will immediately get the value.
{% highlight c %}
//LWT message constants
byte willQoS = 1;
const char* willTopic = "com/oliver/arduino/status";
const char* willMessage = "Arduino Offline";
boolean willRetain = true;
{% endhighlight %}

Next the `mqttClientConnect()` function will setup the connection and if successful, publish an "Online" status to the "com/oliver/arduino/status" topic with a retained set to `true`. Since this is the same as the LWT topic, clients will always get the last known good value. [Here][pubsubapi] is the link to the API documentation of the `connect()` function.
{% highlight c %}
void mqttClientConnect() {
  while(!mqttClient.connected()) {
    boolean connected = mqttClient.connect("arduinoClientId98", willTopic, willQoS, willRetain, willMessage);
    if(connected) {
      //Publish a status of Online on the will topic once the client is connected.
      //Retained is set to true so that new clients connecting will know the status immediately.
      mqttClient.publish(willTopic,"Arduino Online", true);
    } else {
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}
{% endhighlight %}

_Note: I am intentionally leaving out the `loop()` and `setup()` functions for brevity, I will make the entire code available on my [GitHub][githubme] repository._

### 4) A Simple Java Client
Below are code snippets of the Java client. It connects to the MQTT broker, and subscribes to the "com/oliver/arduino/status" topic.
Assuming the Arduino client has been successfully connected earlier, this subscribing client should receive the "Audrino Online" message. On abruptly disconnecting the Arduino client (I yanked out the Ethernet cable), it should received the "Arduino Offline" message. The Java code is written using the [Eclipse Paho MQTT library for Java][pahojava].

{% highlight java%}
//Initial Setup parameters
public static final String BROKER_URL = "tcp://iot.eclipse.org:1883";
public static final String CLIENT_ID = "subClientId98";
public static final String SUB_TOPIC_PATTERN = "com/oliver/ad/status";
private MqttClient mqttSubClient;
{% endhighlight %}

The `setCallback()` method takes a class to callback when events for the client arrives, I have provided a simple in-line implementation for the purposes of this PoC. The `messageArrived()` method prints the message out to standard output. 
{% highlight java%}
public void start() {
	try {
		mqttSubClient.setCallback(new MqttCallback() {
		 public void messageArrived(String topic, MqttMessage message) throws Exception {
		  System.out.println("Status: " + message.toString());
		 }

		 public void deliveryComplete(IMqttDeliveryToken token) {
		 }

		 public void connectionLost(Throwable cause) {
		 }
	  });
	 mqttSubClient.connect();
	 mqttSubClient.subscribe(SUB_TOPIC_PATTERN);

  } catch (MqttException e) {
    e.printStackTrace();
 }
}
{% endhighlight %}

### Conclusion
In this post, we have seen how to use the Last Will Testament and Retained message features of the MQTT protocol to detect abrupt disconnections of publishing clients.

### References
1. [Mqtt.org FAQs][mqttorg]
2. [MQTT Essentials][mqttessentials]
3. [PubSubClient API Documentationn][pubsubapi]




[mqttorg]:http://mqtt.org/faq
[mqttessentials]:http://www.hivemq.com/mqtt-essentials/
[PUBREC]:http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718048
[PUBREL]:http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718053
[PUBCOMP]:http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718058
[PUBACK]:http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718043
[mqttspec]:http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html
[Nodegateway]:https://cdn.thenewstack.io/media/2015/07/Node_Gateway.png
[arduinomqtt]:http://www.hivemq.com/blog/mqtt-client-library-encyclopedia-arduino-pubsubclient/
[pubsubapi]:http://pubsubclient.knolleary.net/api.html#connect2
[githubme]:https://github.com/oliversavio/
[pahojava]:https://eclipse.org/paho/clients/java/