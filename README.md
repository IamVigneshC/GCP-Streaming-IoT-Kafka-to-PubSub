# Streaming-IoT-Kafka-to-PubSub

• Launch a Kafka instance and use it to communicate with Pub/Sub

• Configure a Kafka connector to integrate with Pub/Sub  

• Setup topics and subscriptions for message communication  

• Perform basic testing of both Kafka and Pub/Sub services  

• Connect IoT Core to Pub/Sub



Activate Google Cloud Shell
Cloud Shell Terminal

You can list the active account name with this command:

` gcloud auth list `

Output:

Credentialed accounts:
 - <myaccount>@<mydomain>.com (active)

`gcloud config list project`

Output:

[core]
project = <project_ID>

## Architecture:

![Image of CloudBuild](https://github.com/IamVigneshC/GCP-Streaming-IoT-Kafka-to-PubSub/blob/main/Communication.png)

## Introduction

With the announcement of the Google Cloud Confluent managed Kafka offering, it has never been easier to use Google Cloud's great data tools with Kafka. You can use the Apache Beam Kafka.io connector to go straight into Dataflow, but this may not always be the right solution.

Whether Kafka is provisioned in the Cloud or on premise, you might want to push to a subset of Pub/Sub topics. Why? For the flexibility of having Pub/Sub as your Google Cloud event notifier. Then you could not only choreograph Dataflow jobs, but also use topics to trigger Cloud Functions.

So how do you exchange messages between Kafka and Pub/Sub? This is where the Pub/Sub Kafka Connector comes in handy. 

Tip: Here we use a virtual machine with a single instance of Kafka. This Kafka instance connects to Pub/Sub and exchanges event messages between the two services.

In the real world, Kafka would likely be run in a cluster.


![Image of CloudBuild](https://github.com/IamVigneshC/GCP-Streaming-IoT-Kafka-to-PubSub/blob/main/Architecture.png)

## 1. Configure the Kafka VM instance
In the Cloud Console, go to Navigation Menu > Compute Engine and open an SSH shell to the Kafka VM named kafka-1-vm. (This is SSH Window A.)

Export the path to the Java Virtual Machine for the Kafka VM.

`export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64`

In the SSH window, set an environment variable to the project identifier.

`export PROJECT_ID=[PROJECT_ID]`

Copy the Kafka connector jar file from the storage bucket to the Kafka VM.

`gsutil cp gs://cloud-training/gsp285/binary/cps-kafka-connector.jar .`

Create the destination sub-directory for the Kafka connector:

`sudo mkdir -p /opt/kafka/connectors`

Move the downloaded jar file to the directory created for the Kafka application:

`sudo mv ./cps-kafka-connector.jar /opt/kafka/connectors/`

Update the java connector file permissions to be executable:

`sudo chmod +x /opt/kafka/connectors/cps-kafka-connector.jar`

Change the current directory to /opt/kafka/config:

`cd /opt/kafka/config`

Using an editor, create cps-sink-connector.properties:

`sudo nano cps-sink-connector.properties`

Add the following content, replacing PROJECT_ID with your Project ID. To close Nano press Ctrl+X. Be sure to leave the empty last line.

name=CPSSinkConnector
connector.class=com.google.pubsub.kafka.sink.CloudPubSubSinkConnector
tasks.max=50
topics=to-pubsub
cps.topic=from-kafka
cps.project=PROJECT_ID


Using an editor, create another file named cps-source-connector.properties:

`sudo nano cps-source-connector.properties`

Add the following content, replacing PROJECT_ID with your Project ID. Be sure to leave the empty last line.

name=CPSSourceConnector
connector.class=com.google.pubsub.kafka.source.CloudPubSubSourceConnector
tasks.max=50
kafka.topic=from-pubsub
cps.subscription=to-kafka-sub
cps.project=PROJECT_ID


The Kafka instance is now configured to use the connector. Leave this SSH connection to the Kafka VM instance open, so you can finish the configuration and run the application later.

## 2. Pub/Sub Topic and Subscription setup
In a Cloud Shell window, set an environment variable to the project identifier.

`export PROJECT_ID=[PROJECT_ID]`

Configure Pub/Sub topics to communicate with Kafka:

`gcloud pubsub topics create to-kafka from-kafka`

Create a subscription for the to-kafka topic:

`gcloud pubsub subscriptions create to-kafka-sub --topic=to-kafka --topic-project=$PROJECT_ID`

Pub/Sub is now configured with two topics. A subscription has also been created on the to-kafka topic using the PROJECT_ID variable.

This configuration allows messages to be consumed by Pub/Sub. Go look at Pub/Sub in the Cloud Console.

Now create a subscription for traffic published from Kafka:

gcloud pubsub subscriptions create from-kafka --topic=from-kafka --topic-project=$PROJECT_ID

## 3. Start the Kafka VM application instance
Now you will set up Kafka topics interacting with Pub/Sub.

Return to the Kafka VM instance (SSH Window A) and submit the following command:

`cd /usr/local/kafka/bin`

Run the following commands to start Zookeeper and the base Kafka server.

`sudo /usr/local/kafka/bin/zookeeper-server-start.sh -daemon /usr/local/kafka/config/zookeeper.properties`

`sudo /usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties`


Create a topic that will exchange information to Pub/Sub:

`./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 10 --topic to-pubsub`

Create a topic that will receive messages from Pub/Sub:

`./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 10 --topic from-pubsub`

Move back to the user's home directory:

`cd ~`

Use an editor to create a new file named run-connector.sh and add these contents to it:

`sudo nano run-connector.sh`

#!/bin/bash
/usr/local/kafka/bin/connect-standalone.sh /opt/kafka/config/connect-standalone.properties \
/opt/kafka/config/cps-sink-connector.properties \
/opt/kafka/config/cps-source-connector.properties


Update the file permissions to allow it to be executed from the command line:

`sudo chmod +x ./run-connector.sh`

Start the connect service:

`./run-connector.sh`

The Kafka service is now be running on the VM. Leave this session open so that any errors can be seen.

## 4. Data exchange between Kafka and Pub/Sub
Test Kafka to Pub/Sub (producer/consumer) communication by opening a new SSH window where the Kafka commands will be run.

Open a new SSH connection to the Kafka VM, this is SSH Window B. Enter the following commands to initiate a Kafka console:

`export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64`

`cd /usr/local/kafka/bin`

`./kafka-console-producer.sh --broker-list localhost:9092 --topic to-pubsub`


From the Kafka console at the > prompt, enter the following data elements and press Return/Enter after each.

`{"message":"Hello"}`

`{"message":"From Kafka"}`

Press Ctrl+C to terminate the command entry:

Return to the Cloud Shell and issue the command below to see the information entered in Kafka:

`gcloud pubsub subscriptions pull from-kafka --auto-ack --limit=10`

Note: You may need to run this command a couple of times to see results.


Kafka to Pub/Sub messaging is configured and working as expected.

Test Pub/Sub to Kafka
In SSH Window B, enter the following command:

`cd /usr/local/kafka/bin`

`./kafka-console-consumer.sh --bootstrap-server=localhost:9092 --value-deserializer=org.apache.kafka.common.serialization.StringDeserializer --topic from-pubsub`


Return to the Cloud Shell, publish a message to be consumed by Kafka:

`gcloud pubsub topics publish to-kafka --attribute=data=HelloFromGoogleCloud`

Check SSH Window B for the Kafka VM example output:

`{"message":"","data":"HelloFromGoogleCloud"}`

Pub/Sub to Kafka connectivity is configured and working as expected.

Ctrl+C to stop this process.

## 5. Pub/Sub to Kafka testing
Your architecture for testing Pub/Sub to Kafka is as illustrated below:

![Image of CloudBuild](https://github.com/IamVigneshC/GCP-Streaming-IoT-Kafka-to-PubSub/blob/main/PubSub%20to%20Kafka%20testing.png)

Note: Ensure that a Kafka instance is actually running in the background - there should still be an open window showing the output from the instance.

If the application instance is not currently running, open a new SSH connection to Kafka, change to the user's home directory cd ~, and run the commands export `JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 and ./run-connector.sh` at the command line.

In the consumer/producer SSH session (SSH Window B) enter the following command:

`cd /usr/local/kafka/bin`

`./kafka-console-consumer.sh --bootstrap-server=localhost:9092 --value-deserializer=org.apache.kafka.common.serialization.StringDeserializer --topic from-pubsub`


In the Cloud Shell you'll create some example content. Use an editor and make a text file named movies.txt, and add the following contents to it:

`sudo nano movies.txt`

Deadpool 2
Avengers Infinity Wars
Jurassic World Fallen Kingdom
MI6 Fallout
Black Panther
The Incredibles
Three Billboards Outside of Ebbing Missouri
A Quiet Place
Thoroughbreds
Super Trooper 2

Enter the following script to publish your movie messages to the Kafka consumer:

`while read i; do gcloud pubsub topics publish to-kafka --attribute=data="$i"; done < movies.txt`

From the command above, a stream of messages should be observable in the Kafka consumer window.

In this example you sent a stream of information between two services. As the example demonstrates, exchanging information once configured is fairly straightforward.

### Kafka to Pub/Sub testing
Our architecture for testing Kafka to Pub/Sub is illustrated below:

![Image of CloudBuild](https://github.com/IamVigneshC/GCP-Streaming-IoT-Kafka-to-PubSub/blob/main/Kafka%20to%20PubSub%20testing.png)

Return to SSH Window B, press Ctrl+C to terminate the command from the prior step. Use an editor to create a text file named tv.json and add the following contents:

`sudo nano tv.json`

{"message":"Archer"}
{"message":"Ozark"}
{"message":"Star Trek Discovery"}
{"message":"Westworld"}
{"message":"The Magicians"}
{"message":"Legion"}
{"message":"Cloak and Dagger"}
{"message":"The Good Place"}
{"message":"Silicon Valley"}
{"message":"Mr Robot"}
{"message":"Rick and Morty"}
{"message":"Mindhunter"}

Now use the following script to publish your TV messages to the Pub/Sub consumer:

`cd /usr/local/kafka/bin`

`./kafka-console-producer.sh --broker-list localhost:9092 --topic to-pubsub < tv.json`


In the Cloud Shell window, run the following command to view the messages that have been published from Kafka:

`gcloud pubsub subscriptions pull from-kafka --auto-ack --limit=10`

In this example you have sent a stream of information between two services. When passing information via Kafka, the message content is formatted as JSON.

## 6. IOT simulator - IoT core
Extending your architecture allows the opportunity to explore further integration. In this section the IoT core service will be used to demonstrate connectivity of IoT devices, as illustrated below:

![Image of CloudBuild](https://github.com/IamVigneshC/GCP-Streaming-IoT-Kafka-to-PubSub/blob/main/IOT%20simulator%20-%20IoT%20core.png)

In the Cloud Console, go to Navigation Menu > Compute Engine and open an SSH shell to the iot-device-simulator instance.

SSH into the instance and Clone a git repository to gain access to the lab specific tools:

Clone a git repository to gain access to lab specific code:

`git clone http://github.com/GoogleCloudPlatform/training-data-analyst`

Add an environment variable for the current project id - replace [PROJECT_ID] with the Google Cloud Project ID:

`export PROJECT_ID=[PROJECT_ID]`

Add an environment variable for the region - replace [MY_REGION] with the iot-device-simulator VM Google Cloud region (such as “us-central1”):

`export MY_REGION=[MY_REGION]`

Create a device registry named iotlab-registry:

gcloud beta iot registries create iotlab-registry \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --event-notification-config=topic=projects/$PROJECT_ID/topics/to-kafka

Change the working directory to the iotlab directory:

`cd $HOME/training-data-analyst/quests/iotlab/`

Create a cryptographic key pair that will allow IoT devices to connect to Pub/Sub:

openssl req -x509 -newkey rsa:2048 -keyout rsa_private.pem \
    -nodes -out rsa_cert.pem -subj "/CN=unused"

The simulated devices to be created provide temperature readings from around the world. In the example you will setup an IoT device for Buenos Aires and read values from it into Pub/Sub.

Create a simulated device for Buenos Aires based on the current project settings:

gcloud beta iot devices create temp-sensor-buenos-aires \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=iotlab-registry \
  --public-key path=rsa_cert.pem,type=rs256

Download the CA root certificates from pki.google.com:

wget https://pki.google.com/roots.pem

Note: Before the IoT device simulator is started, make sure that a:

Background Kafka instance is running
Kafka consumer instance is ready to accept messages
In the SSH session for iot-device-simulator, run the following code to begin generating temperature readings to be consumed by Pub/Sub.

python3 cloudiot_mqtt_example_json.py \
   --project_id=$PROJECT_ID \
   --cloud_region=$MY_REGION \
   --registry_id=iotlab-registry \
   --device_id=temp-sensor-buenos-aires \
   --private_key_file=rsa_private.pem \
   --message_type=event \
   --algorithm=RS256

In the consumer/producer SSH session (SSH Window B) enter the following command:

`cd /usr/local/kafka/bin`

`./kafka-console-consumer.sh --bootstrap-server=localhost:9092 --value-deserializer=org.apache.kafka.common.serialization.StringDeserializer --topic from-pubsub`


Once the python command is running, it will send a stream of messages via PubSub to the Kafka instances:

The iot-device-simulator VM is displaying the list of temperatures.

SSH Window B (Kafka consumer) is receiving the inbound message traffic.

At this point, the architecture has been extended to include IoT Core. The example provides a simulated approach that can be further extended to include real devices.
