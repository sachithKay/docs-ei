# Detecting Repeatedly Redelivered Messages

In JMS 2.0, it is mandatory for JMS providers to set the `JMSXDeliveryCount` property, which allows an application that receive a message to determine how many times the message is redelivered.

If a message is being redelivered, it means that a previous attempt to deliver the message failed due to some reason. If a message is being redelivered multiple times, it can be because the message is *bad* in some way. When such a message is being redelivered over and over again, it wastes resources and prevents subsequent *good* messages from being processed.

When you work with WSO2 Micro Integrator, you can detect such repeatedly redelivered messages using the `JMSXDeliveryCount` property that is set in messages.

Being able to detect repeatedly redelivered messages is particularly useful because you can take necessary steps to handle such messages in a proper manner. For example, you can consume such a message and send it to a separate queue.

The following diagram illustrates how WSO2 Micro Integrator can be used to detect repeatedly redelivered messages, and store such messages in an internal message store.

![](attachments/119130322/119130323.png){width="500"}

To demonstrate the scenario illustrated above, let's configure the JMS inbound endpoint in WSO2 Micro Integrator using HornetQ as the message broker.

## Synapse configuration

The synapse configuration for this example scenario is as follows:

``` java tab="Registry Artifact"
<registry provider="org.wso2.carbon.mediation.registry.WSO2Registry">
    <parameter name="cachableDuration">15000</parameter>
</registry>
```

``` java tab="Task Manager"
<taskManager provider="org.wso2.carbon.mediation.ntask.NTaskTaskManager">
    <parameter name="cachableDuration">15000</parameter>
</taskManager>
```

``` java tab="Sequence (Request)"
<sequence name="request" onError="fault">
    <log level="full"/>
    <filter regex="1" source="get-property('default','jms.message.delivery.count')" xmlns:ns="http://org.apache.synapse/xsd">
        <then>
            <log>
                 <property name="DeliveryCounter" value="1"/>
            </log>
        </then>
         <else>
            <store messageStore="JMS-Redelivered-Store"/>
            <log>
                <property name="DeliveryCounter" value="more than 1"/>
            </log>
         </else>
    </filter>
    <drop/>
</sequence>
```

``` java tab="Sequence (Main)"
<sequence name="main">
    <log level="full"/>
    <drop/>
</sequence>
```

``` java tab="Sequence (Fault)"
<sequence name="fault">
    <log level="full">
        <property name="MESSAGE" value="Executing default &quot;fault&quot; sequence"/>
        <property expression="get-property('ERROR_CODE')" name="ERROR_CODE"/>
        <property expression="get-property('ERROR_MESSAGE')" name="ERROR_MESSAGE"/>
    </log>
    <drop/>
</sequence>
```

``` java tab="Message Store"
<messageStore name="JMS-Redelivered-Store"/>
```

``` java tab="Inbound Endpoint"
<inboundEndpoint name="jms_inbound" onError="fault" protocol="jms"sequence="request" suspend="false">
    <parameters>
       <parameter name="interval">1000</parameter>
       <parameter name="transport.jms.Destination">queue/mySampleQueue</parameter>
       <parameter name="transport.jms.CacheLevel">1</parameter>
       <parameter name="transport.jms.ConnectionFactoryJNDIName">QueueConnectionFactory</parameter>
       <parameter name="sequential">true</parameter>
       <parameter name="java.naming.factory.initial">org.jnp.interfaces.NamingContextFactory</parameter>
       <parameter name="java.naming.provider.url">jnp://localhost:1099</parameter>
       <parameter name="transport.jms.SessionAcknowledgement">AUTO_ACKNOWLEDGE</parameter>
       <parameter name="transport.jms.SessionTransacted">false</parameter>
    </parameters>
</inboundEndpoint>
```

See the descriptions of the above configurations:

<table>
  <tr>
    <th>Artifact</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>Inbound Endpoint</td>
    <td>
      This configuration creates an inbound endpoint to the JMS broker and has a simple sequence that logs the message status using the `JMSXDeliveryCount` value.
    </td>
  </tr>
  <tr>
    <td>Task Manager</td>
    <td>The task manager configuration...</td>
  </tr>
  <tr>
    <td>Registry Arfiact</td>
    <td>The registry artifact..</td>
  </tr>
</table>

## Running the Example

1. Configure the Micro Integrator (Publisher) with the broker.
2. Start the Broker.
3. Start WSO2 Integration Studio and create artifacts with the above configuration. You can copy the synapse configuration given above to the **Source View** of your proxy service.
4. Run the following java file (**SOAPPublisher.java**) to publish a message to the JMS queue:
    
    ``` java
    package JMSXDeliveryCount;
        
    import java.util.Properties;
    import java.util.logging.Logger;
        
    import javax.jms.ConnectionFactory;
    import javax.jms.Destination;
    import javax.jms.JMSContext;
    import javax.naming.Context;
    import javax.naming.InitialContext;
    import javax.naming.NamingException;
        
    public class SOAPPublisher {
                private static final Logger log = Logger.getLogger(SOAPPublisher.class.getName());
        
                // Set up all the default values
                private static final String param = "IBM";
        
                // with header for inbounds
                private static final String MESSAGE_WITH_HEADER =
                        "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\">\n" +
                                "   <soapenv:Header/>\n" +
                                "<soapenv:Body>\n" +
                                "<m:placeOrder xmlns:m=\"http://services.samples\">\n" +
                                "    <m:order>\n" +
                                "        <m:price>" +
                                getRandom(100, 0.9, true) +
                                "</m:price>\n" +
                                "        <m:quantity>" +
                                (int) getRandom(10000, 1.0, true) +
                                "</m:quantity>\n" +
                                "        <m:symbol>" +
                                param +
                                "</m:symbol>\n" +
                                "    </m:order>\n" +
                                "</m:placeOrder>" +
                                "   </soapenv:Body>\n" +
                                "</soapenv:Envelope>";
                private static final String DEFAULT_CONNECTION_FACTORY = "QueueConnectionFactory";
                private static final String DEFAULT_DESTINATION = "queue/mySampleQueue";
                private static final String INITIAL_CONTEXT_FACTORY = "org.jnp.interfaces.NamingContextFactory";
                private static final String PROVIDER_URL = "jnp://localhost:1099";
        
                public static void main(String[] args) {
        
                    Context namingContext = null;
        
                    try {
        
                        // Set up the namingContext for the JNDI lookup
                        final Properties env = new Properties();
                        env.put(Context.INITIAL_CONTEXT_FACTORY, INITIAL_CONTEXT_FACTORY);
                        env.put(Context.PROVIDER_URL, System.getProperty(Context.PROVIDER_URL, PROVIDER_URL));
                        namingContext = new InitialContext(env);
        
                        // Perform the JNDI lookups
                        String connectionFactoryString =
                                System.getProperty("connection.factory",
                                                   DEFAULT_CONNECTION_FACTORY);
                        log.info("Attempting to acquire connection factory \"" + connectionFactoryString + "\"");
                        ConnectionFactory connectionFactory =
                                (ConnectionFactory) namingContext.lookup(connectionFactoryString);
                        log.info("Found connection factory \"" + connectionFactoryString + "\" in JNDI");
        
                        String destinationString = System.getProperty("destination", DEFAULT_DESTINATION);
                        log.info("Attempting to acquire destination \"" + destinationString + "\"");
                        Destination destination = (Destination) namingContext.lookup(destinationString);
                        log.info("Found destination \"" + destinationString + "\" in JNDI");
        
                        // String content = System.getProperty("message.content",
                        // DEFAULT_MESSAGE);
                        String content = System.getProperty("message.content", MESSAGE_WITH_HEADER);
        
                        try (JMSContext context = connectionFactory.createContext()) {
                            log.info("Sending  message");
                            // Send the message
                            context.createProducer().send(destination, content);
                        }
        
                    } catch (NamingException e) {
                        log.severe(e.getMessage());
                    } finally {
                        if (namingContext != null) {
                            try {
                                namingContext.close();
                            } catch (NamingException e) {
                                log.severe(e.getMessage());
                            }
                        }
                    }
                }
        
        
                private static double getRandom(double base, double varience, boolean onlypositive) {
                    double rand = Math.random();
                    return (base + (rand > 0.5 ? 1 : -1) * varience * base * rand) *
                            (onlypositive ? 1 : rand > 0.5 ? 1 : -1);
                }
            }
    ```
When you analyze the output on the console, you will see an entry similar to the following:

``` java
    INFO - LogMediator To: , MessageID: ID:60868ca5-d174-11e5-b7de-f9743c9bcc9e, Direction: request, DeliveryCounter = 1
```
