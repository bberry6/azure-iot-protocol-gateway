# Microsoft Azure IoT Protocol Gateway

Azure IoT protocol gateway is a framework that enables communication with Azure IoT Hub over the MQTT protocol. It is a pass-through component bridging traffic between connected IoT devices and IoT Hub. The protocol gateway can be deployed in Azure using Cloud Services worker roles. It can also be deployed in on-premises environments.

The Azure IoT protocol gateway includes an MQTT adapter to facilitate communication with devices over the MQTT v3.1.1 protocol. The MQTT adapter also demonstrates the programing model for building protocol adapters for other protocols. The Azure IoT protocol gateway programming model allows you to plug in custom components for specialized processing such as custom authentication, message transformations, compression/decompression, or encryption/decryption of traffic between the devices and IoT Hub. 

This document describes how to get started using the Azure IoT protocol gateway including general overview, deployment, and setup instructions.

For guidance on customizations and extensibility of the protocol gateway, please refer to the [Azure IoT Protocol Gateway - Developer Guide](https://github.com/Azure/azure-iot-protocol-gateway/blob/master/docs/DeveloperGuide.md).

## How to Build and Run

### Prerequisites
The protocol gateway requires an Azure IoT hub to connect to and an Azure Storage account used for persisting state. You can also use the Azure Storage Emulator for the console or cloud samples running locally.

To build the protocol gateway you need Microsoft Visual Studio 2013 (or later) and Microsoft Azure SDK 2.7 installed on your development machine. For deployment of the protocol gateway you need Microsoft Azure PowerShell installed.

### Running the Console Sample
The Azure IoT protocol gateway repository contains samples in the ‘/samples’ folder. The console app in this folder hosts a protocol gateway that connects to IoT Hub and starts listening for devices to connect (Note: by default the gateway listens on port 8883 for MQTT and uses port 5672 for AMQP traffic).
In order to run the console sample please follow these steps:

1. Provide the IoT hub connection string in the `IoTHubConnectionString` setting in the app settings configuration file samples\ProtocolGateway.Samples.Console\appSettings.config.user. You can use the connection string for the device policy in the `Shared access policies` settings of IoT Hub.
2. Run `build.cmd` or build the solution from Visual Studio. 
	Both the build.cmd and the solution file are in the root folder of the repository
3. Start the Microsoft Azure Storage Emulator
4. Run as administrator the compiled binary (samples\ProtocolGateway.Samples.Console\[Debug or Release]\Gateway.Samples.Console.exe)
	If you run this from Visual Studio, then Visual Studio needs to be run as administrator.
	Note: If the required network ports are not enabled for communication with IoT Hub, you might be prompted to open the firewall ports.

### Running an End-to-End Test
There is also an end-to-end test that exercises the following scenario:

- the test creates an instance of the protocol gateway or alternatively it can use an existing one
- the test simulates a device that connects to the protocol gateway (which is connected to an IoT hub)
- an app backend is simulated by using an Event Hub receiver to read messages from IoT Hub, and an IoT Hub service client to send messages to the device
- the device sends two messages to IoT Hub (one with QoS 0 and one with QoS 1) and those are received by the app backend (through the Event Hub receiver) and verified
- the app backend sends 3 messages (with QoS 0, 1 and 2) to the device and those are verified by device when received

Please follow these steps in order to run the end-to-end test for the protocol gateway:

1. Update test configuration file (`test\ProtocolGateway.Tests\appSettings.config.user`) providing the following settings:
	- `IoTHubConnectionString` - connection string for your IoT hub that will be used for the test. For this test specifically you need to provide the You can use the connection string for the device policy in the `Shared access policies` settings of IoT Hub.
		Note: It is recommended to use an IoT hub instance specifically created for the tests. The test will register a device in IoT Hub and will it for to exchange messages between the device and the simulated app back end.
	- `End2End.DeviceName` - identity of the device that will be used during the test. If the device name is not specified, a new device will be created for the test.
	- If you want to run the test against an already running protocol gateway, uncomment the `End2End.ServerAddress` setting in test configuration file and provide an IP-address of the protocol gateway you want to use. Otherwise, the test will start a new in-process protocol gateway. If you want to connect to a running protocol gateway on the same machine you can use 127.0.0.1 as the IP address.
	(Note: by default, the gateway listens on port 8883 for MQTT and uses port 5672 for AMQP traffic).
2. In Visual Studio, run the unit test `EndToEndTests.BasicFunctionalityTest` (in the ProtocolGateway.Tests project). This test is expected to pass if the setup has been successful.

### Deploying Cloud Sample
Please follow these steps to deploy the cloud sample in Azure Cloud Services worker roles:
	
1. Open the solution in Visual Studio.
2. Right-click on **samples/ProtocolGateway.Samples.Cloud** project and choose **Package**.
3. Open command line and set current directory to `<repo root>\samples`.
4. Execute the following command: `PowerShell -File deploy.ps1 <cloud service name> <storage account name> <location> <tls cert path> <tls cert password> <IoT Hub connection string> [-SubscriptionName <subscription name>] [-VmCount <VM count>]`
	- `<IoT Hub connection string>` can contain any credentials as the default Authentication Provider (`SasTokenAuthenticationProvider`) will override them with the ones provided by device upon connection.
	- Please note that based on your local configuraiton you might need to change the PowerShell execution policy, if you are receiving an execution error.  
5. You will be prompted to sign-in to your Azure account. It will be used for the deployment session.
6. Wait for the script execution to complete. 
7. To verify the successful deployment, perform an end-to-end test as described in [Running End-to-End Test](#running-end-to-end-test). Please note that you need to supply the public IP address of the Cloud Service in the `End2End.ServerAddress` setting.

### Diagnostics and Monitoring
#### Logging
The protocol gateway uses EventSource and Semantic Logging Application Block for logging (see https://msdn.microsoft.com/en-us/library/dn440729(v=pandp.60).aspx for details). By default, diagnostic events of all levels (including verbose) are logged to the console output in the Console sample and to `SLABLogsTable` Azure Storage Table in the Cloud sample.

In addition, the Cloud sample is also configured to collect Windows Event Log events and Crash Dumps. Please see https://msdn.microsoft.com/en-us/library/azure/dn186185.aspx for details on configuring Windows Azure Diagnostics.

#### Performance Counters
The protocol gateway emits the following performance counters under the “Azure IoT protocol gateway” performance counter category:
-	**Connections Established Total** – established connection count since last protocol gateway startup
-	**Connections Current** – current connection count
-	**Connections Established/sec** – rate of establishing new connections
-	**Failed Connections (due to Auth Issues)/sec** – rate of connection authentication failures
-	**Failed Connections (due to Operational Errors)/sec** – rate of connection failures due to other errors
-	**MQTT Packets Received/sec** – rate of MQTT packet arrival (from devices)
-	**MQTT Packets Sent/sec** – rate at which protocol gateway sends out MQTT packets (to devices)
-	**MQTT PUBLISH packets Received/sec** – rate of MQTT PUBLISH packet arrival (from devices)
-	**MQTT PUBLISH packets Sent/sec** – rate at which protocol gateway sends out MQTT PUBLISH packets (to devices)
-	**Messages Received/sec** – rate at which protocol gateway receives messages from IoT Hub (intended for forwarding to devices)
-	**Messages Sent/sec** – rate at which protocol gateway is sending messages to IoT Hub
-	**Messages Rejected/sec** – rate at which protocol gateway is rejecting messages received from IoT Hub (e.g. because of topic name generation errors, no subscription match, or reaching max allowed retransmission count)
-	**Outbound Message Processing Time, msec** – average time of processing outbound messages (from the time a message was received to the time the message was forwarded to a device or acknowledgement was completed (for messages with QoS > 0))
-	**Inbound Message Processing Time, msec** – average time of processing inbound message (from the time a message was received from a device to the time it was sent (and acknowledgement for QoS > 0) was completed.

For the Cloud sample, Windows Azure Diagnostics is configured to collect protocol gateway as well as the default (processor and memory) performance counters. See https://msdn.microsoft.com/en-us/library/azure/dn535595.aspx for details.

## Configuration Settings

The protocol gateway provides a number of configuration settings to help fine tune its behavior. Note that the default configuration provides a good starting point. Usually there is no need to adjust it, unless a specific behavior (like retransmission) is required. The Console sample derives the configuration settings from the appSettings.config.user file, while the Cloud sample derives them from the cloud service configuration.

- **IoTHubConnectionString**. REQUIRED Connection string to IoT Hub. Defines the connection parameters when connecting to IoT Hub on behalf of devices. By default, the protocol gateway will override the credentials from the connection string with the device credentials provided when a device connects to the gateway. No default value.
- **ConnectArrivalTimeout**. If specified, the time period after which an established TCP-connection will get closed, if a CONNECT packet has not been received. If not specified or is less or equal to 00:00:00, no timeout will be imposed on a CONNECT packet arrival. Default value: not set
- **MaxKeepAliveTimeout**. If specified, and device connects with KeepAlive=0 or a value that is higher than MaxKeepAliveTimeout, the connections keep alive timeout will be set to MaxKeepAliveTimeout. If not specified or is less or equal to 0, the KeepAlive value provided by the device will be used. Default value: not set
- **RetainProperty**. Protocol gateway will set a property with RetainProperty name and value of “True” on a message sent to IoT Hub, if the PUBLISH packet was marked with RETAIN=1. Default value: mqtt-retain
- **DupProperty**. Protocol gateway will set a property with DupProperty name and value of “True” on a message sent to IoT Hub, if the PUBLISH packet was marked with DUP=1. Default value: mqtt-dup
- **QoSProperty**. Indicates the name of the property on cloud-to-device messages that might be used to override the default QoS level for the device-bound message processing. For device to cloud messages, the name of the property that indicates the QoS level of a message when received from the device. Default value: mqtt-qos
- **DefaultPublishToClientQoS**. Default Quality of Service to be used when publishing a message to a device. Can be overridden by adding a property on the message in IoT Hub with the name defined by QoSProperty setting and value of the desired QoS level. Supported values are: 0, 1, 2. Default value: 1 (at least once)
- **MaxPendingsInboundMessages**. Maximum allowed number of received messages pending processing. Protocol gateway will stop reading data from the device connection once it reaches MaxPendingInboundMessages and will restart only after the number of pending messages becomes lower than MaxPendingInboundMessages. Default value: 16
- **MaxPendingOutboundMessages**. Maximum allowed number of published device-bound messages pending acknowledgement. Protocol gateway will stop receiving messages from IoT Hub once it reaches MaxPendingOutboundMessages and will restart only after the number of messages pending acknowledgement becomes lower than MaxPendingOutboundMessages. The number of messages is calculated as a sum of all unacknowledged messages: PUBLISH (QoS 1) pending PUBACK, PUBLISH (QoS 2) pending PUBREC, and PUBREL pending PUBCOMP. Default value: 1
- **MaxOutboundRetransmissionCount**. If specified, maximum number of attempts to deliver a device-bound message. Messages that reach the limit of allowed delivery attempts will be rejected. If not specified or is less or equal to 0, no maximum number of delivery attempts is imposed. Default value: -1
- **DeviceReceiveAckTimeout**. If specified, indicates timeout for acknowledgement. If an acknowledgement times out for any of the acknowledgement queues (PUBACK, PUBREC, PUBCOMP), the device connection will be put in retransmission mode until all outstanding acknowledgements are received. If not specified or is less of equal to 00:00:00, acknowledgement is awaited without a timeout. Default value: 00:00:00
- **MaxInboundMessageSize**. REQUIRED Maximum message size allowed for publishing from a device to the gateway. If a device publishes a bigger message, the protocol gateway will close the connection. The max supported value is 262144 (256 KB). Default value: 262144
- **BlobSessionStatePersistenceProvider.StorageConnectionString**. Azure Storage connection string to be used by the  BlobSessionStatePersistenceProvider to store MQTT session state data. Default value for Console sample and Cloud sample's Local configuration: `UseDevelopmentStorage=true`. For Cloud deployment the default value is a connection string to specified storage account.
- **BlobSessionStatePersistenceProvider.StorageContainerName**. Azure Storage Blob Container Name to be used by the BlobSessionStatePersistenceProvider to store MQTT session state data. Default value: mqtt-sessions
- **TableQos2StatePersistenceProvider.StorageConnectionString**. Azure Storage connection string to be used by TableQos2StatePersistenceProvider to store QoS 2 delivery information. Default value for Console sample and Cloud sample's Local configuration: `UseDevelopmentStorage=true`. For Cloud deployment the default value is a connection string to specified storage account.
- **TableQos2StatePersistenceProvider.StorageTableName**. Azure Storage Table name to be used by the TableQos2StatePersistenceProvider to store QoS 2 delivery information. Default value: mqttqos2