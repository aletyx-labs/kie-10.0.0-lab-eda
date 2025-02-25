# Lab: Event Driven Architecture 

You will interact with decisions and rule engine using Kafka topics, and will create from a couple of processes that will also interact with Kafka topics using the `Messages` nodes.

### Supporting Files

The initial project already contains the necessary dependencies to run Decisions and Rules in event driven mode, you can access it here https://github.com/aletyx-labs/kie-10.0.0-lab-eda .

### Instructions

#### Step 1: Clone the Repository

Clone the lab project from this repository, which contains all the necessary files and infrastructure for the lab.

#### Step 2: Import the Project

Import the project into VS Code and explore the provided content, including the POJOs, RuleUnits, DRL file and DMN.

#### Step 3: Start Kafdrop

To best interact with Kafka topics, to create new messages and inspect generated ones, we'll use Kafdrop (usign docker container). To start all you need is run the following in your terminal:

```bash
$ ./kafdrop.sh
```

It'll start Kafdrop and make the UI available in port 9000: localhost:9000

#### Step 4: Change the default topic name

We'll use a single instance of Kafka that is already up and running in `104.248.236.197:9092`, however to properly isolate the content of the Lab tests you have to change the sufix of the topics used to your id.

The name of the topic is defined in the application.properties file under src/main/resources. Replace the following lines:

```properties
...
mp.messaging.incoming.kogito_incoming_stream.topic=requests-alex
...
mp.messaging.outgoing.kogito_outgoing_stream.topic=responses-alex
```

to something like 

```properties
...
mp.messaging.incoming.kogito_incoming_stream.topic=requests-tim
...
mp.messaging.outgoing.kogito_outgoing_stream.topic=responses-tim
```

At this point you'll interact only with the topics mentioned above. ^

#### Step 5: Run in dev:mode and submit a Decision Cloud Event.

```bash
$ mvn clean quarkus:dev
```

Once application is running, go to Kafdrop and send the following message to your `request-[your_id]` topic.

Note: the key aspect here for kogito identify what it should do with such message is the Type ("DecisionRequest"), DMN Namespace ("kogitodmnmodelnamespace") and Model Name ("kogitodmnmodelname").

```json
{
    "specversion":"1.0",
    "id":"a89b61a2-5644-487a-8a86-144855c5dce8",
    "source":"SomeEventSource",
    "type":"DecisionRequest",
    "subject":"TheSubject",
    "kogitodmnmodelname":"Traffic Violation",
    "kogitodmnmodelnamespace":"https://github.com/kiegroup/drools/kie-dmn/_A4BCA8B8-CF08-433F-93B2-A2598F19ECFF",
    "kogitodmnfullresult":true,
    "data":{
       "Driver":{
          "Age":25,
          "Points":13
       },
       "Violation":{
          "Type":"speed",
          "Actual Speed":115,
          "Speed Limit":60
       }
    }
 }
```

If everything is working as expected, you can now open the `response-[your_id]` topic and see the result of the DMN file execution.


#### Step 6: Submit a RuleUnit Cloud Event.

Now we'll send a new message to the same `request-[your_id]` topic, but this time for the Rule Engine:

Note: the key aspect here for kogito identify what it should do with such message is the Type ("RulesRequest"), Unit Name ("kogitoruleunitid") and Query ("kogitoruleunitquery").

```json
{
  "specversion": "1.0",
  "id": "a89b61a2-5644-487a-8a86-144855c5dce8",
  "source": "SomeEventSource",
  "type": "RulesRequest",
  "subject": "TheSubject",
  "kogitoruleunitid": "ai.aletyx.workshop.eda.LoanUnit",
  "kogitoruleunitquery": "FindApproved",
  "data": {
    "maxAmount": 5000,
    "loanApplications": [
      {
        "id": "ABC10001",
        "amount": 2000,
        "deposit": 100,
        "applicant": {
          "age": 45,
          "name": "John"
        }
      },
      {
        "id": "ABC10002",
        "amount": 5000,
        "deposit": 100,
        "applicant": {
          "age": 25,
          "name": "Paul"
        }
      },
      {
        "id": "ABC10015",
        "amount": 1000,
        "deposit": 100,
        "applicant": {
          "age": 12,
          "name": "George"
        }
      }
    ]
  }
}
```

If everything is working as expected, you can now open the `response-[your_id]` topic and see the result of the above execution with the applications approved or denied.

#### Step 7: Let's create our first BPMN file

Before create our first BPMN file, we need to add to the pom.xml the following dependency, so that BPMN files are properly handled.

```xml
    <dependency>
      <groupId>org.jbpm</groupId>
      <artifactId>jbpm-with-drools-quarkus</artifactId>
    </dependency>
```

Create a new file name test.bpmn under src/main/resources and open it using the Apache KIE BPMN Editor.

Once file is created, create the process variables that we'll need.

| Name             | Data Type                               |
|-----------------|----------------------------------------|
| orderId        | String                                 |
| newOrderInfo    | String                                 |

#### Step 8: Create the process diagram for test.bpmn

1. Start Event
2. Script task name Process

     Script: kcontext.setVariable("newOrderInfo", "addedInfo+" + orderId);

3. End Message: 
        3.1. Message: `responses-[your-id]`
        3.2. Data Assignment

          Variables - Input Mapping:

          | Name   | Data Type | Source     |
          |--------|----------|------------|
          | event | String   | newOrderInfo    |


Now you can save and restart your Quarkus application. Once restart is complete, you can go to DevUI and start the process you just created.

Once you execute it, you should be able to see a message in the `responses-[your-id]` topic with the newOrderInfo content.


#### Step 9: Let's create another BPMN file, this time to listen to messages in a topic

Create a new file name event.bpmn under src/main/resources and open it using the Apache KIE BPMN Editor.

Once file is created, create the process variables that we'll need.

| Name             | Data Type                               |
|-----------------|----------------------------------------|
| orderId        | String                                 |

#### Step 10: Create the process diagram for event.bpmn


1. Start Message Event
    1.1. Message: `requests-[your-id]`
    1.2. Data Assignment

        Variables - Output Mapping:

        | Name   | Data Type | Source     |
        |--------|----------|------------|
        | event | String   | orderId    |

2. Script task name Process

     Script: System.out.println("Content Here:" + orderId);

3. End

Now you can save and restart your Quarkus application. Once restart, create the following message in Kafdrop for the `requests-[your-id]` topic:

```json
{
  "specversion": "1.0",
  "id": "order-123456",
  "source": "",
  "type": "requests-alex",
  "time": "2025-02-25T12:00:00Z",
  "data": "ORD-789012"
}
```

If everything is working as expected, you can see the message we print in the logs of the quarkus application.


#### Step 11: Collect events from the engine

You can optionally add the following depencency to your project. This dependency will send to Kafka `kogito-processinstances-events` a message for each change of state of the process instance.

```xml
    <dependency>
      <groupId>org.kie</groupId>
      <artifactId>kie-addons-quarkus-events-process</artifactId>
    </dependency>
```

#### Step 12: Disable rest endpoints

At this point we explored several different mechanisms to interact with Kafka, however if you check the DevUI, the swagger-ui will show that you also have REST endpoint for all your DMNs, RuleUnits and Processeses. You may not want to expose the same service with both protocols, for that you can disable the rest endpoint generation by adding the following property in your application.properties:

```properties
kogito.generate.rest=false
```

