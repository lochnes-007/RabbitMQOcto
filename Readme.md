# RabbitMQ Tutorial

Notes for running RabbitMQ's Java tutorials on macOS, starting with the **Hello World** producer.

## What you need

- RabbitMQ server installed and running locally.
- JDK 21, which includes both the Java runtime (`java`) and compiler (`javac`). This project uses Homebrew for RabbitMQ and OpenJDK 21.
- The RabbitMQ Java client and SLF4J API JAR files, included in this folder.

RabbitMQ is the message broker: it receives messages from programs, holds them in queues, and makes them available to other programs. The server is separate from the Java client program.

## Start and check RabbitMQ

Start RabbitMQ in the background with Homebrew:

```bash
brew services start rabbitmq
```

The tutorial connects to `localhost` (this computer) using RabbitMQ's standard AMQP port, `5672`. To check that the server is responding, run:

```bash
rabbitmq-diagnostics ping
```

It should report `Ping succeeded`. If the tutorial's Java program cannot connect, check that the server is running and listening on port 5672.

## Install and check Java

Install JDK 21 with Homebrew:

```bash
brew install openjdk@21
```

On macOS, Homebrew may require linking the JDK so macOS can discover it. If `java -version` cannot find Java, run:

```bash
sudo ln -sfn "$(brew --prefix openjdk@21)/libexec/openjdk.jdk" /Library/Java/JavaVirtualMachines/openjdk-21.jdk
```

Open a new terminal and check that both the runtime and compiler are available:

```bash
java -version
javac -version
```

Both should report version 21.

## Compile and run `Send.java`

Run these commands from this project folder:

```bash
javac -cp ".:amqp-client-5.16.0.jar" Send.java
java -cp ".:amqp-client-5.16.0.jar:slf4j-api-1.7.36.jar" Send
```

`javac` compiles `Send.java` into `Send.class`. The `-cp` option sets the classpath: places Java should look for compiled classes and libraries. A dot (`.`) means the current folder, and colons (`:`) separate classpath entries on macOS.

The RabbitMQ client JAR contains classes the program imports to connect and communicate with RabbitMQ. The SLF4J API JAR contains the logging API used by the client. You may see a warning that SLF4J could not find a logging implementation. The API alone does not provide one, so log messages are suppressed; this warning does not prevent sending. A compatible implementation such as `slf4j-simple` can be added if you want client log output.

On success, the program prints:

```text
 [x] Sent 'Hello World!'
```

This means the program reached its confirmation print after publishing. To see the message, run the tutorial's receiving program; the print by itself does not mean a consumer received it.

## Run `Recv.java`

Compile and run the receiver in a second terminal from this project folder:

```bash
javac -cp ".:amqp-client-5.16.0.jar" Recv.java
java -cp ".:amqp-client-5.16.0.jar:slf4j-api-1.7.36.jar" Recv
```

Start the receiver before running `Send` if you want to watch it receive a message immediately. If `Send` ran first, its message remains in the durable queue and the receiver can read it later, provided no other consumer already took it. The receiver keeps running and prints `Waiting for messages`; stop it with `Ctrl+C`.

The sender and receiver must use the **same queue name** (`hello`) and compatible queue settings. The sender publishes to the `hello` queue, and the receiver listens to that queue. A different name would refer to a different queue, so the receiver would wait on the wrong mailbox. Both programs declare the queue with the same durable, exclusive, auto-delete, and quorum settings; RabbitMQ requires declarations of an existing queue to match.

## `Recv.java`, explained line by line

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
import java.nio.charset.StandardCharsets;
import java.util.Map;
```

These imports make the RabbitMQ connection, channel, and delivery callback classes available, along with UTF-8 decoding and the map used for queue options.

```java
public class Recv {
```

Declares the public class `Recv`. The source file is named `Recv.java` to match it.

```java
    private final static String QUEUE_NAME = "hello";
```

Defines the queue name as a constant. It must match the publisher's queue name so both programs use the same queue.

```java
    public static void main(String[] argv) throws Exception {
```

Declares the program's entry point. `argv` can contain command-line arguments, but this program does not use them. `throws Exception` lets connection or RabbitMQ errors propagate.

```java
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
```

Creates a connection factory and configures it to connect to RabbitMQ on this computer. With the default settings, the client uses port 5672 and the default guest credentials.

```java
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
```

Opens a network connection to the broker and creates a channel for RabbitMQ operations. The connection is like a phone call to the server; the channel is a work lane within that call. Unlike the sender's try-with-resources block, these resources are not automatically closed here. The program is intended to keep running as a consumer until it is stopped.

```java
        channel.queueDeclare(QUEUE_NAME, true, false, false,
                             Map.of("x-queue-type", "quorum"));
```

Declares the `hello` queue if it does not already exist, with durable set to `true`, exclusive set to `false`, auto-delete set to `false`, and queue type `quorum`. These settings must match the publisher's declaration. Declaring the queue on the receiving side ensures it exists before consumption begins.

```java
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
```

Prints a status message so you know the receiver is waiting. `Ctrl+C` in the terminal stops the program.

```java
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
```

Creates a callback, which is code RabbitMQ's client runs each time a message is delivered. `consumerTag` identifies this consumer; this example does not use it. `delivery` contains the received message and its metadata. The `->` syntax is a Java lambda expression.

```java
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
```

Gets the message body as bytes and decodes those bytes into a Java string using UTF-8, matching the encoding used by `Send`.

```java
            System.out.println(" [x] Received '" + message + "'");
        };
```

Prints the received text, then closes the callback definition.

```java
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
```

Starts consuming messages from `hello`. The second argument, `true`, enables automatic acknowledgement: RabbitMQ considers a message handled as soon as it delivers it to this program. `deliverCallback` handles each message. The final callback is for consumer cancellation; its body is empty because this example takes no action when cancellation happens. `basicConsume` registers the consumer, and the Java client keeps the connection active while it waits for deliveries.

```java
    }
}
```

The braces close `main` and the `Recv` class.

## `Send.java`, explained line by line

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.nio.charset.StandardCharsets;
import java.util.Map;
```

- The first three imports bring in RabbitMQ's channel, connection, and connection factory classes.
- `StandardCharsets` provides the UTF-8 character encoding used to turn the text into bytes.
- `Map` is used to provide queue options.

```java
public class Send {
```

Declares a public Java class named `Send`. The source file is named `Send.java` to match the public class name.

```java
    private final static String QUEUE_NAME = "hello";
```

Defines a constant queue name. `private` limits access to this class, `static` makes it belong to the class, and `final` means its value cannot be reassigned. The queue is named `hello`.

```java
    public static void main(String[] argv) throws Exception {
```

This is the program entry point. `public` lets Java invoke it, `static` means no `Send` object is needed, and `void` means it returns no value. `argv` holds optional command-line arguments (unused here). `throws Exception` allows errors to propagate to Java rather than handling them in this method.

```java
        ConnectionFactory factory = new ConnectionFactory();
```

Creates a RabbitMQ connection factory, which holds settings used to open a connection.

```java
        factory.setHost("localhost");
```

Sets RabbitMQ's host to this same computer. With the default client settings, the connection uses port 5672 and the default guest credentials.

```java
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
```

Opens a network connection to RabbitMQ, then creates a channel over that connection. A connection is like a phone call between the program and the server; a channel is like a conversation or work lane within that call. The channel carries operations such as declaring queues and publishing messages. Java's try-with-resources syntax automatically closes the channel and connection when the block ends, even if an error occurs.

```java
            channel.queueDeclare(QUEUE_NAME, true, false, false,
                                 Map.of("x-queue-type", "quorum"));
```

Declares the `hello` queue with these options:

1. `QUEUE_NAME`: the queue's name (`hello`).
2. `true`: durable, so the queue metadata survives a broker restart.
3. `false`: not exclusive to this connection.
4. `false`: do not automatically delete the queue when its last consumer disconnects.
5. `Map.of("x-queue-type", "quorum")`: make this a quorum queue.

If a queue with this name already exists, its settings must match this declaration or RabbitMQ reports an error.

```java
            String message = "Hello World!";
```

Stores the text that will be sent.

```java
            channel.basicPublish("", QUEUE_NAME, null,
                                 message.getBytes(StandardCharsets.UTF_8));
```

Publishes a message through the channel. The empty exchange name selects RabbitMQ's default exchange, which routes to a queue whose name matches the routing key. `QUEUE_NAME` is that routing key. `null` means no extra message properties are specified. The last argument converts the text to UTF-8 bytes for the message body.

```java
            System.out.println(" [x] Sent '" + message + "'");
```

Prints a confirmation in the terminal. It confirms the publishing call returned; it does not confirm that a consumer received the message.

```java
        }
    }
}
```

The braces end the try block, `main` method, and `Send` class. When the try block ends, Java automatically closes the channel and connection.

## Where to find the official tutorial examples

RabbitMQ's tutorial pages explain each concept, while the companion repository contains runnable source code in different languages:

- [Official tutorial pages](https://www.rabbitmq.com/tutorials)
- [Official tutorial code repository](https://github.com/rabbitmq/rabbitmq-tutorials)
- [Java tutorial examples](https://github.com/rabbitmq/rabbitmq-tutorials/tree/main/java)

