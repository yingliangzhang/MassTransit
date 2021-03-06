# Consumers

A consumer is a class that may consume one or more message types. Each message type is defined by the `IConsumer<T>` interface, where `T` is the message type.

```cs
public class UpdateCustomerConsumer :
    IConsumer<UpdateCustomerAddress>
{
    public async Task Consume(ConsumeContext<UpdateCustomerAddress> context)
    {
        await Console.Out.WriteLineAsync($"Updating customer: {context.Message.CustomerId}");

        // update the customer address
    }
}
```

When a consumer is configured on a receive endpoint, and a message type consumed by the consumer is received, an instance of the consumer is created (using a consumer factory, which is a delegate, or a container-specific consumer factory from one of the supported [containers](/usage/containers/)). The message (wrapped in a `ConsumeContext`) is then delivered to the consumer via the `Consume` method.

The `Consume` method is asynchronous, and returns a Task. The task is awaited by MassTransit, during which time the message is unavailable to other receive endpoints. If the consume method completes successfully (a task status of RanToCompletion), the message is acknowledged and removed from the queue.

::: warning
If the consumer faults (such as throwing an exception, resulting in a task status of Faulted), or is somehow cancelled (TaskStatus of Canceled), the exception is propagated back up the pipeline where it can ultimately be retried or moved to an error queue.
:::

## Consumer

For a consumer to receive messages, the consumer must be connected to a receive endpoint. This is done during bus configuration, particularly within the configuration of a receive endpoint.

An example of connecting a consumer to a receive endpoint is shown below.

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");

    cfg.ReceiveEndpoint("customer_update_queue", e =>
    {
        e.Consumer<UpdateCustomerConsumer>();
    });
});
```

The example creates a bus, which connects to the RabbitMQ running on the local machine, using the default username and password (guest/guest). On that bus. a single receive endpoint is created with the name *customer_update_queue*. The consumer is connected using the simplest method, which accepts a consumer class with a default constructor.

::: tip NOTE
When a consumer is connected to a receive endpoint, the combined set of message types consumed by all of the consumers connected to the same receive endpoint are *subscribed* to the queue. The subscription method varies by broker, in the case of RabbitMQ exchange bindings are created for the message types to the exchange/queue for the receive endpoint.

These subscriptions are persistent, and remain in place after the process exits. This ensures that messages published or sent that would be delivered to one of the receive endpoint consumers are saved even is the process is terminated. When the process is started, messages waiting in the queue will be delivered to the consumer(s).
:::

### Consumer Factories

The above example connects the consumer using a default constructor consumer factory. There are several other consumer factories supported, as shown below.

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");

    cfg.ReceiveEndpoint("customer_update_queue", e =>
    {
        // an anonymous factory method
        e.Consumer(() => new YourConsumer());

        // an existing consumer factory for the consumer type
        e.Consumer(consumerFactory);

        // a type-based factory that returns an object (container friendly)
        e.Consumer(consumerType, type => Activator.CreateInstance(type));

        // an anonymous factory method, with some middleware goodness
        e.Consumer(() => new YourConsumer(), x =>
        {
            // add middleware to the consumer pipeline
            x.UseExecuteAsync(context => Console.Out.WriteLineAsync("Consumer created"));
        });
    });
});
```

### Connect Consumers

Once a bus has been configured, returning an `IBusControl` reference, the receive endpoints have been created and cannot be modified. The bus itself, however, provides a temporary (auto-delete) queue which can be used to receive messages. To connect a consumer to the bus temporary queue, a series of *Connect* methods can be used.

::: warning
Published messages will not be received by the temporary queue. Because the queue is temporary, when consumers are connected no bindings or subscriptions are created. This makes it very fast for transient consumers, and avoids thrashing the message broker with temporary bindings.
:::

The temporary queue is useful to receive request responses and faults (via the response/fault address header) and routing slip events (via an event subscription in the routing slip).

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");
});

busControl.Start();

ConnectHandle handle = busControl.ConnectConsumer<FaultConsumer>();

handle.Disconnect(); // disconnect the consumer from the bus pipeline
```

In addition to the `ConnectConsumer` method, methods for each consumer type are also included (`ConnectHandler`, `ConnectInstance`, `ConnectSaga`, and `ConnectStateMachineSaga`).

## Instance

While using a consumer instance per message is highly suggested, it is possible to connect an existing consumer instance which will be called for every message. The consumer *must* be thread-safe, as the ```Consume``` method will be called from multiple threads simultaneously. To connect an existing instance, see the example below.

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");

    cfg.ReceiveEndpoint("customer_update_queue", e =>
    {
        e.Instance(existingConsumer);
    });
});
```

### Undeliverable Messages

If the configuration of an endpoint changes, or if a message is mistakenly sent to an endpoint, it is possible that a message type is received that does not have any connected consumers. If this occurs, the message is moved to a *_skipped* queue (prefixed by the original queue name). The original message content is retained, and additional headers are added to indicate the host which moved the message.

## Handler

While creating a consumer is the preferred way to consume messages, it is also possible to create a simple message handler. By specifying a method, anonymous method, or lambda method, a message can be consumed on a receive endpoint.

To configure a simple message handler, refer to the example below.

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");

    cfg.ReceiveEndpoint("customer_update_queue", e =>
    {
        e.Handler<UpdateCustomerAddress>(context =>
            return Console.Out.WriteLineAsync($"Update customer address received: {context.Message.CustomerId}"));
    });
});
```

In this case, the method is called for each message received. No consumer is created, and no lifecycle management is performed.

## Observer

With the addition of the `IObserver` interface, the concept of an observer was added to the .NET framework. MassTransit supports the direct connection of observers to receive endpoints.

> Unfortunately, observers are not asynchronous. Because of this, it is not possible to play nice with the async support provided by the compiler when using an observer.

An observer is defined using the built-in `IObserver<T>` interface, as shown below.

```csharp
public class CustomerAddressUpdatedObserver :
    IObserver<ConsumeContext<CustomerAddressUpdated>>
{
    public void OnNext(ConsumeContext<CustomerAddressUpdated> context)
    {
        Console.WriteLine("Customer address was updated: {0}", context.Message.CustomerId);
    }

    public void OnError(Exception error)
    {
    }

    public void OnCompleted()
    {
    }
}
```

Once created, the observer is connected to the receive endpoint similar to a consumer instance.

```csharp
var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    cfg.Host("localhost");

    cfg.ReceiveEndpoint("customer_update_queue", e =>
    {
        e.Observer<CustomerAddressUpdatedObserver>();
    });
});
```