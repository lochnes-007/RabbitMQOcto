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

The callback does not poll the queue or run by itself. Registering it with `basicConsume` tells the client to call it whenever a message is delivered. If a message is already waiting, it may be called soon after registration; otherwise it is called when a publisher sends a message. The callback is the receiver's message-handling code: RabbitMQ delivers the message to the client, and the client invokes the callback with the delivery.

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

Starts consuming messages from `hello`. The second argument, `true`, enables automatic acknowledgement: RabbitMQ considers a message handled as soon as it delivers it to the client, before the callback finishes processing. `deliverCallback` handles each message: it decodes and prints the body, then returns. The client remains registered and invokes it again for later deliveries. With automatic acknowledgement, if the callback fails or the program stops before processing finishes, RabbitMQ may already consider that message handled and will not requeue it.

The final callback is for consumer cancellation; its body is empty because this example takes no action when cancellation happens. For safer processing, automatic acknowledgement can be disabled (`false`) and the callback can acknowledge a message with `channel.basicAck(...)` only after processing succeeds. If the consumer stops before acknowledging, RabbitMQ can requeue that message for another delivery. The `basicQos(1)` prefetch setting is useful with manual acknowledgements: it limits this consumer to one unacknowledged message at a time, so RabbitMQ waits for an acknowledgement before sending the next one.

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

## Publish/subscribe with `EmitLog.java` and `ReceiveLogs.java`

This pair demonstrates a `fanout` exchange. Unlike `Send`/`Recv`, the publisher sends to an exchange rather than naming a queue. The exchange copies each published message to every queue currently bound to it. Each running `ReceiveLogs` process creates its own temporary queue, so each receiver gets a copy of messages published while it is subscribed. Because these queues are temporary, messages sent while no receiver is running are not kept for a later receiver.

Your `RMQOcto` variable can be used as the classpath. Set it in each terminal session if needed:

```bash
export RMQOcto=.:amqp-client-5.16.0.jar:slf4j-api-1.7.36.jar:slf4j-simple-1.7.36.jar
```

Compile the two programs:

```bash
javac -cp "$RMQOcto" EmitLog.java ReceiveLogs.java
```

Run `ReceiveLogs` in one terminal and leave it running:

```bash
java -cp "$RMQOcto" ReceiveLogs
```

In another terminal, publish a message:

```bash
java -cp "$RMQOcto" EmitLog Hello from RabbitMQ
```

The receiver should print the message. You can start multiple `ReceiveLogs` processes in separate terminals; each gets a copy of messages published while its temporary queue is bound. Stop a receiver with `Ctrl+C`.

### How `EmitLog.java` works

- `EXCHANGE_NAME = "logs"` gives the publisher and receiver a shared exchange name.
- `new ConnectionFactory()` and `setHost("localhost")` configure a connection to the local RabbitMQ server.
- The try-with-resources block opens a connection and channel and closes them automatically when publishing is done.
- `exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT)` declares a fanout exchange named `logs`. Declaring it on both sides is safe when the exchange settings match.
- `argv.length < 1 ? "info: Hello World!" : String.join(" ", argv)` uses a default message when no command-line text is provided; otherwise it joins all supplied words with spaces.
- `basicPublish(EXCHANGE_NAME, "", null, message.getBytes(StandardCharsets.UTF_8))` publishes UTF-8 message bytes to `logs`. The empty routing key is ignored by a fanout exchange; `null` means no extra message properties are set.
- The `println` reports that the publish call completed. It does not prove a receiver got the message.

### How `ReceiveLogs.java` works

- It creates a connection factory, sets the host to `localhost`, then opens a connection and channel. These remain open so the receiver can keep listening.
- `exchangeDeclare("logs", FANOUT)` ensures the fanout exchange exists.
- `queueDeclare().getQueue()` asks RabbitMQ to create a server-named, temporary queue. Each receiver gets its own queue.
- `queueBind(queueName, "logs", "")` binds that queue to the exchange. A fanout exchange ignores the routing key, so the empty string is suitable.
- The `DeliverCallback` runs for each delivered message. It decodes the body as UTF-8 and prints it.
- `basicConsume(queueName, true, deliverCallback, ...)` registers the callback with automatic acknowledgement. RabbitMQ considers a delivery handled as soon as it sends it to the client, before the callback finishes.
- The final empty callback handles consumer cancellation; it takes no action here. `Ctrl+C` ends the long-running receiver process.

## Route messages with `EmitLogDirect.java` and `ReceiveLogsDirect.java`

This pair demonstrates a `direct` exchange. A direct exchange sends a message to queues whose binding key exactly matches the message's routing key. Here the routing key is the log severity, such as `info`, `warning`, or `error`. Unlike the fanout example, a receiver can select which severities it wants by binding its queue to those keys.

Compile both programs from the project folder using your `RMQOcto` classpath:

```bash
javac -cp "$RMQOcto" EmitLogDirect.java ReceiveLogsDirect.java
```

Start a receiver in one terminal, subscribing it to `warning` and `error`:

```bash
java -cp "$RMQOcto" ReceiveLogsDirect warning error
```

In another terminal, send messages with different severities:

```bash
java -cp "$RMQOcto" EmitLogDirect info This is an informational message
java -cp "$RMQOcto" EmitLogDirect warning This is a warning
java -cp "$RMQOcto" EmitLogDirect error Something failed
```

The receiver prints the `warning` and `error` messages. It does not receive `info`, because its queue is not bound with the `info` routing key. Start another receiver with `info` to receive messages routed with that key. These receivers each create temporary queues, so start them before publishing if you want them to get the messages.

If no severity is provided, `EmitLogDirect` uses `info`. If the severity is given without message text, it sends `Hello World!`. For example, `java -cp "$RMQOcto" EmitLogDirect` publishes the default `info` message.

### How `EmitLogDirect.java` works

- `EXCHANGE_NAME = "direct_logs"` names the exchange shared with the receivers.
- The factory and `setHost("localhost")` configure a connection to the local RabbitMQ server. The try-with-resources block closes the connection and channel after publishing.
- `exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT)` declares a direct exchange. The exchange type determines that routing keys must match bindings exactly.
- `getSeverity(argv)` uses the first command-line argument as the routing key, or `info` if no argument was given.
- `getMessage(argv)` uses the remaining arguments as message text joined with spaces, or defaults to `Hello World!`.
- `basicPublish(EXCHANGE_NAME, severity, ...)` publishes to `direct_logs`, using the severity as the routing key. `null` means no extra properties are set; the message is encoded as UTF-8 bytes.
- The output line reports the severity and message the program attempted to publish.
- `joinStrings` combines the message arguments from the given starting index, putting a space between each one.

### How `ReceiveLogsDirect.java` works

- It connects to `localhost`, creates a channel, and declares the same `direct_logs` exchange with the same exchange type.
- `queueDeclare().getQueue()` creates a server-named temporary queue for this receiver.
- If no severities are provided, it prints the usage instructions and exits.
- The loop calls `queueBind` once per supplied severity. Each binding key selects messages with that exact routing key. Multiple bindings let one receiver get several severities.
- Its delivery callback decodes the body as UTF-8 and prints both the delivery's routing key and the message.
- `basicConsume(queueName, true, deliverCallback, ...)` registers the callback with automatic acknowledgement, so RabbitMQ considers each message handled as it delivers it to the client.
- The final empty callback does nothing on consumer cancellation. The receiver keeps listening until stopped with `Ctrl+C`.

## Match routing patterns with `EmitLogTopic.java` and `ReceiveLogsTopic.java`

This pair demonstrates a `topic` exchange. Topic routing keys are words separated by dots, for example `kern.critical` or `app.auth.error`. A queue binding can use a pattern: `*` matches exactly one word, while `#` matches zero or more words. RabbitMQ sends a message to each queue whose binding pattern matches the message's routing key.

Think of the exchange as a mail sorting room. The publisher writes a category label on each message (the routing key); a receiver queue has a sorting rule (the binding pattern). The topic exchange reads the label and copies the message into every queue whose rule matches. The publisher does not need to know which queues exist. For example, a publisher can send `app.auth.error` once, and separate queues can independently subscribe to `app.*.error`, `app.auth.*`, or `#`.

Routing keys are made of dot-separated words. The dots define the word boundaries; `*` and `#` are special only in binding patterns. For example, `app.auth.error` has three words: `app`, `auth`, and `error`.

| Binding pattern | Matches `kern.critical` | Matches `app.auth.error` | Why |
| --- | --- | --- | --- |
| `kern.*` | Yes | No | `*` matches exactly one word after `kern`. |
| `*.auth.*` | No | Yes | Each `*` matches one word, so this requires exactly three words with `auth` in the middle. |
| `app.#` | No | Yes | `#` matches zero or more words after `app`. |
| `#` | Yes | Yes | It matches any routing key, including one with several words. |
| `kern.#` | Yes | No | It matches routing keys starting with `kern`, with zero or more words after it. |

The exchange type determines how strict matching is. A `direct` exchange requires the routing key and binding key to be identical. A `topic` exchange allows the `*` and `#` patterns. A `fanout` exchange ignores the routing key and sends to all bound queues.

Compile both programs with your `RMQOcto` classpath:

```bash
javac -cp "$RMQOcto" EmitLogTopic.java ReceiveLogsTopic.java
```

Start a receiver in one terminal. This pattern subscribes to all routing keys with `kern` as the first word:

```bash
java -cp "$RMQOcto" ReceiveLogsTopic 'kern.*'
```

In another terminal, publish messages:

```bash
java -cp "$RMQOcto" EmitLogTopic kern.critical Kernel failure
java -cp "$RMQOcto" EmitLogTopic app.auth.error Login failed
```

The receiver gets `kern.critical` because it matches `kern.*`. It does not get `app.auth.error`. Try a broad binding such as `#` to receive every message, or `*.auth.*` to receive three-word routing keys whose middle word is `auth`:

```bash
java -cp "$RMQOcto" ReceiveLogsTopic '#'
java -cp "$RMQOcto" ReceiveLogsTopic '*.auth.*'
```

Quote wildcard patterns in the shell so the shell passes `*` and `#` to Java unchanged. Each receiver creates a temporary queue, so run it before publishing to receive those messages. If no routing key is given to `EmitLogTopic`, it defaults to `anonymous.info`; if no message text follows the routing key, it defaults to `Hello World!`.

Here is how the first publish command is interpreted:

```bash
java -cp "$RMQOcto" EmitLogTopic kern.critical Kernel failure
```

Java receives `argv[0]` as `kern.critical`, `argv[1]` as `Kernel`, and `argv[2]` as `failure`. The program uses the first argument as the routing key and joins the remaining arguments with spaces to form `Kernel failure`. The receiver's `kern.*` binding matches because `kern` matches the first word and `*` matches the one remaining word, `critical`.

For another example, `app.auth.error Login failed` has the routing key `app.auth.error` and message text `Login failed`. A receiver bound with `*.auth.*` gets it: the first `*` matches `app`, the literal `auth` matches the second word, and the last `*` matches `error`. The `kern.*` receiver does not get that message because its pattern does not match.

### How `EmitLogTopic.java` works

- `private static final String EXCHANGE_NAME = "topic_logs";` defines a constant name for the exchange. The receiver uses the same name so both programs refer to the same exchange.
- `new ConnectionFactory()` creates the connection settings object, and `setHost("localhost")` points it at the broker on this computer. The try-with-resources statement opens a connection and channel, then closes both when publishing finishes.
- `exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC)` ensures the exchange exists and is a topic exchange. If it already exists with a different type, RabbitMQ rejects the declaration.
- `getRouting(argv)` selects the routing key. With `EmitLogTopic app.auth.error Login failed`, it returns `app.auth.error`; with no arguments, it returns `anonymous.info`.
- `getMessage(argv)` takes all arguments after the routing key and joins them with spaces. In that example, it returns `Login failed`. If the command has no second argument, it returns `Hello World!`.
- `basicPublish(EXCHANGE_NAME, routingKey, null, ...)` publishes to the `topic_logs` exchange. Its second argument is the message's routing key, not a queue name. The topic exchange compares this key to each bound queue's pattern. `null` means no extra message properties are supplied.
- `message.getBytes("UTF-8")` converts the Java text into bytes for the message body, using UTF-8 so text can be represented consistently.
- `System.out.println(...)` displays the routing key and body. It shows what the publisher sent; it does not confirm that a queue matched or a receiver processed the message.
- `joinStrings(strings, " ", 1)` builds the body from the argument array starting at index 1. The loop appends each later word with a space, so `Login` and `failed` become `Login failed`.

### How `ReceiveLogsTopic.java` works

- `newConnection()` opens a connection to RabbitMQ, and `createChannel()` creates the channel used for exchange, queue, binding, and consume operations. These stay open while the receiver waits.
- `exchangeDeclare("topic_logs", TOPIC)` declares the same exchange with the same type as the publisher. Declaring it on both sides makes either program able to start first.
- `queueDeclare().getQueue()` asks RabbitMQ to create a queue and choose its name. With no durability or name options, this queue is temporary and exclusive to the connection. Think of it as a short-lived inbox created for this receiver; when the receiver disconnects, the inbox is removed.
- The `argv.length < 1` check requires at least one binding pattern. If you start the class with no patterns, it prints an example of the expected command and exits rather than waiting without any subscription.
- `for (String bindingKey : argv)` visits every pattern supplied at launch. `queueBind(queueName, EXCHANGE_NAME, bindingKey)` attaches the receiver's queue to the exchange using that pattern. For `ReceiveLogsTopic kern.* '*.auth.*'`, the queue has two rules and receives a message if either rule matches its routing key.
- `DeliverCallback` is the code the Java client runs for each delivery. It decodes the byte body as UTF-8 and prints the routing key from the message envelope alongside the message text. Seeing the routing key helps explain why the pattern matched.
- `basicConsume(queueName, true, deliverCallback, ...)` registers the callback to receive messages. The `true` enables automatic acknowledgement, so RabbitMQ treats each delivery as handled as soon as it sends it to this client, before the callback finishes.
- The final empty callback is for consumer cancellation and performs no action. The process remains active waiting for messages; press `Ctrl+C` to stop it and remove its temporary queue.

## Where to find the official tutorial examples

RabbitMQ's tutorial pages explain each concept, while the companion repository contains runnable source code in different languages:

- [Official tutorial pages](https://www.rabbitmq.com/tutorials)
- [Official tutorial code repository](https://github.com/rabbitmq/rabbitmq-tutorials)
- [Java tutorial examples](https://github.com/rabbitmq/rabbitmq-tutorials/tree/main/java)

