AWS Mosquitto Broker
--------------------------------

[![](https://images.microbadger.com/badges/image/mantgambl/aws_mosquitto_broker.svg)](https://microbadger.com/images/mantgambl/aws_mosquitto_broker "Get your own image badge on microbadger.com")
[![](https://images.microbadger.com/badges/version/mantgambl/aws_mosquitto_broker.svg)](https://microbadger.com/images/mantgambl/aws_mosquitto_broker "Get your own version badge on microbadger.com")

Docker Image for AWS IOT connected Mosquitto broker.

![enter image description here](https://s3.amazonaws.com/aws-iot-blog-assets/how-to-bridge-mosquitto-mqtt-broker-to-aws-iot/1-overview.png)

## Step 1: Setup AWS Account

Go to [AWS](https://docs.aws.amazon.com/iot/latest/developerguide/iot-console-signin.html) and setup the account.

Navigate to `User` -> `My Security Credentials`, and obtain **Access Key ID** and **Access Key**.

## Step 2: Clone the Repository

Clone this repository to a location in your drive.

## Step 3: Install and Setup AWS CLI

Install AWS CLI from [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html).

Run `aws configure` in terminal and type in your Region, your Access ID and Keys, as followed:

```sh
aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: eu-central-1
Default output format [None]: json
```

## Step 3: Create an IAM policy for the bridge

Run the following command to create policy for the bridge:

```sh
aws iot create-policy --policy-name bridge --policy-document '{"Version": "2012-10-17","Statement": [{"Effect": "Allow","Action": "iot:*","Resource": "*"}]}'
```

## Step 4: Create Certificates

Go into the `aws_mosquitto_broker/config/certs` directory and run the following to create certificates:

```sh
cd aws_mosquitto_broker/config/certs

sudo aws iot create-keys-and-certificate --set-as-active --certificate-pem-outfile cert.crt --private-key-outfile private.key --public-key-outfile public.key --region eu-central-1
```

Then you can run the `aws iot list-certificates` to check the created certificates. Copy the ARN in the form of `arn:aws:iot:eu-central-1:0123456789:cert/xyzxyz`:

```sh
aws iot list-certificates
```

Attach the policy to your certificate. Replace the `{REPLACE_ARN_CERT}` with your copied ARN `arn:aws:iot:eu-central-1:0123456789:cert/xyzxyz`:

```sh
aws iot attach-principal-policy --policy-name bridge --principal {REPLACE_ARN_CERT}
```

Add read permissions to **private key**, **public key** and **client cert** (inside `certs` folder):

```sh
sudo chmod 644 private.key && sudo chmod 644 public.key && sudo chmod 644 cert.crt
```

Download the root Amazon CA certificate also in the `certs` directory:

```sh
sudo curl https://www.websecurity.digicert.com/content/dam/websitesecurity/digitalassets/desktop/pdfs/roots/VeriSign-Class%203-Public-Primary-Certification-Authority-G5.pem -o rootCA.pem
```


## Step 5: Edit mosquitto custom config file

Rename `awsbridge.conf.sample` to `awsbridge.conf`:

```sh
mv awsbridge.conf.sample awsbridge.conf
```

Edit `config/conf.d/awsbridge.conf` and follow the awsbridge.conf instructions:

```sh
nano config/conf.d/awsbridge.conf
```

**Note:** Run `aws iot describe-endpoint` to get the AWS IoT endpoint.

## Step 6:  Build Docker File

Go back to the root location `aws_mosquitto_broker` and run the following:

```sh
docker build -t aws_mqtt_broker .
```

**Note:** Make sure you have installed docker on your PC first.

## Step 7: Run Docker Image

```sh
docker run -ti -p 1883:1883 -p 9001:9001 --name mqtt aws_mqtt_broker
```

Console / Log output:

```sh
1493564060: mosquitto version 1.4.10 (build date 2016-10-26 14:35:35+0000) starting
1493564060: Config loaded from /mosquitto/config/mosquitto.conf.
1493564060: Opening ipv4 listen socket on port 1883.
1493564060: Opening ipv6 listen socket on port 1883.
1493564060: Bridge local.bridgeawsiot doing local SUBSCRIBE on topic localgateway_to_awsiot
1493564060: Bridge local.bridgeawsiot doing local SUBSCRIBE on topic both_directions
1493564060: Connecting bridge awsiot (a3uewmymwlcmar.iot.us-east-1.amazonaws.com:8883)
1493564060: Bridge bridgeawsiot sending CONNECT
1493564060: Received CONNACK on connection local.bridgeawsiot.
1493564060: Bridge local.bridgeawsiot sending SUBSCRIBE (Mid: 1, Topic: awsiot_to_localgateway, QoS: 1)
1493564060: Bridge local.bridgeawsiot sending UNSUBSCRIBE (Mid: 2, Topic: localgateway_to_awsiot)
1493564060: Bridge local.bridgeawsiot sending SUBSCRIBE (Mid: 3, Topic: both_directions, QoS: 1)
1493564060: Received SUBACK from local.bridgeawsiot
1493564061: Received UNSUBACK from local.bridgeawsiot
1493564061: Received SUBACK from local.bridgeawsiot
```


## Step 8: Test


### Publish from aws iot console

1.- From AWS Management Console go to `AWS IoT Services` -> `Test`

2.- Subscribe to topics mentioned in our config file `awsiot_to_localgateway`, `localgateway_to_awsiot` and `both_directions`.

3.- Publish to `awsiot_to_localgateway` topic (hello world).

4.- Review log or console output in our local broker for something like this:

`1493564128: Received PUBLISH from local.bridgeawsiot (d0, q0, r0, m0, 'awsiot_to_localgateway', ... (45 bytes)) `

**Note:** Make sure that it is selected the `eu-central-1` as the region.

### Publish from host

Flow: hostpc -> dockergateway -> aws

`mosquitto_pub -h localhost -p 1883 -q 1 -d -t localgateway_to_awsiot  -i clientid1 -m "{\"key\": \"helloFromLocalGateway\"}"`

### Publish from arduino UNO / Mega with Ethernet Shield

[Arduino Client](https://github.com/dnavarrom/aws_mqtt_arduino_uno)

## References:

[AWS Mosquitto Guide](https://aws.amazon.com/es/blogs/iot/how-to-bridge-mosquitto-mqtt-broker-to-aws-iot/)

[Docker Mosquitto Image](https://github.com/toke/docker-mosquitto)
