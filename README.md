# [Alert-System-Using-W5100s-EVB-Pico-SNS](https://maker.wiznet.io/gemma/projects/alert-system-using-w5100s-evb-pico-aws-sns/)

![cover](https://user-images.githubusercontent.com/87741718/188807547-9e21a36a-c00f-49c2-a39d-94d2d6bda796.png)

## 1. Concept
Alert system that operates when the value sent from the sensor exceeds the threshold.
Connect sensor to W5100S-EVB-Pico and receive sensor data from IoT core. If the sensor value exceeds the threshold, SNS is triggered through the Rule. You will be notified of the route you have set through SNS.
![image](https://user-images.githubusercontent.com/87741718/188807962-9d9f42f8-6635-4c33-bae1-23c1127c67ae.png)

## 2. How to make
### 2.1. Connect W5100S-EVB-Pico to AWS IoT Core
#### 2.1.1. Create Thing
Click 'Create things'
![image](https://user-images.githubusercontent.com/87741718/188810270-db968400-8f5b-4154-9b04-db43ebe8784e.png)

Select single thing
![image](https://user-images.githubusercontent.com/87741718/188810333-049a35ee-a046-4680-bbcf-1015efe8a1e9.png)

Set thing name
![image](https://user-images.githubusercontent.com/87741718/188810348-488c6a91-8a08-4e8f-840a-327267e02851.png)

Select Auto-generate option
![image](https://user-images.githubusercontent.com/87741718/188810363-d48d338d-cbb7-4175-a719-4cc988263054.png)

Create and connect policies for using IoT Core. The policy details are as follows. </br>
If you done [Attach policies to certificate] step, you can create thing. And you must save certificate files.
```JSON
{   
    "Version": "2012-10-17",   
    "Statement": [
         {       
            "Effect": "Allow",       
            "Action": "iot:Connect",
            "Resource": "*"     
        },
        {
            "Effect": "Allow",
            "Action": "iot:Publish",
            "Resource": "*"
        },
        {       
            "Effect": "Allow",
            "Action": "iot:Receive",
            "Resource": "*"
        },
        {       
            "Effect": "Allow",
            "Action": "iot:Subscribe",
            "Resource": "*"
        }
    ] 
}
```


#### 2.1.2. Publish MQTT Message
I used the SDK that connects the RP2040 to AWS, and you can use it by following the [github link](https://github.com/Wiznet/RP2040-HAT-AWS-C/tree/main/examples/aws_iot_mqtt). I changed the thing Name to the thing Name created above.
```C
/* AWS IoT */ 
#define MQTT_DOMAIN "account-specific-prefix-ats.iot.ap-northeast-2.amazonaws.com" 
#define MQTT_PUB_TOPIC "$aws/things/YOUR_THING_NAME/shadow/update" 
#define MQTT_SUB_TOPIC "$aws/things/YOUR_THING_NAME/shadow/update/accepted" 
#define MQTT_USERNAME NULL 
#define MQTT_PASSWORD NULL 
#define MQTT_CLIENT_ID "YOUR_THING_NAME"
```
Set up device certificate and keys. </br>
Put the authentication key saved when creating device in [mqtt_certificate.h] file.
```c
uint8_t mqtt_root_ca[] =
"-----BEGIN CERTIFICATE-----\r\n"
"...\r\n"
"-----END CERTIFICATE-----\r\n";

uint8_t mqtt_client_cert[] =
"-----BEGIN CERTIFICATE-----\r\n"
"...\r\n"
"-----END CERTIFICATE-----\r\n";

uint8_t mqtt_private_key[] =
"-----BEGIN RSA PRIVATE KEY-----\r\n"
"...\r\n"
"-----END RSA PRIVATE KEY-----\r\n";
```
Build SDK file, and put the permware in your device. </br>
The message payload currently sent by the W5100S-EVB-Pico is:
```JSON
{"temperature":23, "humidity":25}
```
![image](https://user-images.githubusercontent.com/87741718/188999762-def2adb2-3c13-4b0a-ae84-8db8d984bc16.png)

### 2.2. Setting AWS SNS
#### 2.2.1. Create Topic
![image](https://user-images.githubusercontent.com/87741718/188999924-35668681-80e0-4ed8-a1c2-0f4adc195359.png)

Select type as Standard, and set the topic' name. FIFO type support only SQS.
![image](https://user-images.githubusercontent.com/87741718/188999961-3debe6d4-ef24-4efc-bc3e-ce5b9622afbb.png)

#### 2.2.2. Create Subscriptions
In topic detail page, click 'create subscription'.
![image](https://user-images.githubusercontent.com/87741718/189000000-1e3b458d-2cb3-4056-b34f-e5cc2b85e122.png)

Subscription is alert's endpoint.
![image](https://user-images.githubusercontent.com/87741718/189000621-4f193c20-c23a-4ce3-8ccf-bbc22bf64745.png)

After subscription creation, a verify email is sent to the corresponding email address. You can only use it after confirming it in the email. Go to the mail and click Confirm subscription.
![image](https://user-images.githubusercontent.com/87741718/189000645-3af717a3-8727-4efc-a742-3d71c394ce22.png)

#### 2.2.3. Test SNS
In topic, Click 'Publish message'.
![image](https://user-images.githubusercontent.com/87741718/189000726-2d586ab9-a68b-4351-a1d1-a2d0d267d1f0.png)

Enter test message in message body and click 'publish message'
![image](https://user-images.githubusercontent.com/87741718/189000778-f96ef2c0-00e5-4da0-8c14-63ac5c804fb0.png)

You can check that the test message has been sent to the mail.
![image](https://user-images.githubusercontent.com/87741718/189000798-65ef1760-59fd-42f8-ba29-1fccd8de62ba.png)

### 2.3. Create IoT Core Rule
Click 'Create Rule' and, set the rule name.
![image](https://user-images.githubusercontent.com/87741718/189000888-de67d9b8-d522-45bf-bd96-b454897880e4.png)

SQL statement's structure is:
```SQL
SELECT  <Attribute> FROM <Topic Filter> WHERE <Condition>
```
It is set to trigger when the temperature value is greater than 1 (a specific value).
![image](https://user-images.githubusercontent.com/87741718/189000944-0ca65677-b372-4060-93fe-bb78e07fd25b.png)

In rule action, select SNS and then select the created topic.
![image](https://user-images.githubusercontent.com/87741718/189778694-df08a76b-f1b0-4e74-b02f-967786003c31.png)

### 2.4. Test
If you activate the rule created by IoT Core while the device is connected, you will receive a notification with a subscription set according to the sql statement.
![image](https://user-images.githubusercontent.com/87741718/189001019-dfbe75d5-2bc2-4e14-9bd8-2d93321bec29.png)




