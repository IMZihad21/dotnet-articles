In larger applications, producers (publishing events from command handlers / services) and consumers (processing events in background workers) belong to different deployment units or have different scaling needs. KafkaFlow supports this cleanly by allowing separate cluster registrations or isolated configuration slices.

This article shows a minimal, production-grade pattern: **producer-only** in API / command-side services and **consumer-only** in background / worker projects — both reading the same configuration section but registering only what they need.

Configuration Model (shared)

```csharp
public class KafkaFlowConfiguration
{
    public string ServerUrl { get; set; } = string.Empty;

    public OrderConfig Orders { get; set; } = new();
}

public class OrderConfig
{
    public string TopicName { get; set; } = string.Empty;
    public string ConsumerGroup { get; set; } = string.Empty;
    public string ConsumerName { get; set; } = string.Empty;
}
```

Producer-Only Extension (use in API / command services)

```csharp
public static class KafkaFlowProducerExtensions
{
    public static IServiceCollection AddKafkaFlowProducer(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<KafkaFlowConfiguration>(
            configuration.GetSection("KafkaConfiguration"));

        var config = configuration
            .GetSection("KafkaConfiguration")
            .Get<KafkaFlowConfiguration>();

        services.AddKafka(kafka => kafka
            .AddCluster(cluster => cluster
                .WithBrokers([config.ServerUrl])
                .CreateTopicIfNotExists(config.Orders.TopicName, 10, 1)
                .AddProducer("order-producer", p => 
                    p.DefaultTopic(config.Orders.TopicName)))
        );

        return services;
    }
}
```

Consumer-Only Extension (use in worker / processor projects)

```csharp
public static class KafkaFlowConsumerExtensions
{
    public static IServiceCollection AddKafkaFlowConsumer(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<KafkaFlowConfiguration>(
            configuration.GetSection("KafkaConfiguration"));

        var config = configuration
            .GetSection("KafkaConfiguration")
            .Get<KafkaFlowConfiguration>();

        services.AddKafka(kafka => kafka
            .AddCluster(cluster => cluster
                .WithBrokers([config.ServerUrl])
                .CreateTopicIfNotExists(config.Orders.TopicName, 10, 1)
                .AddConsumer(consumer => consumer
                    .Topic(config.Orders.TopicName)
                    .WithGroupId(config.Orders.ConsumerGroup)
                    .WithName(config.Orders.ConsumerName)
                    .WithBufferSize(10)
                    .WithConsumerLagWorkerBalancer(maxWorkers: 100, minWorkers: 10, lagThreshold: 25)
                    .WithWorkerDistributionStrategy<FreeWorkerDistributionStrategy>()
                    .AddMiddlewares(middlewares => middlewares
                        .AddTypedHandlers(handlers => handlers
                            .AddHandler<OrderCreatedHandler>()
                            .AddHandler<OrderPaidHandler>()
                            .AddHandler<OrderShippedHandler>()))))
        );

        return services;
    }
}
```

Shared appsettings.json (or separate overrides per project)

```json
{
  "KafkaConfiguration": {
    "ServerUrl": "kafka-1:9092,kafka-2:9092",
    "Orders": {
      "TopicName": "orders",
      "ConsumerGroup": "order-processing",
      "ConsumerName": "order-events-handler"
    }
  }
}
```

Domain Events

```csharp
public record OrderCreated(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    DateTime CreatedAt);

public record OrderPaid(
    Guid OrderId,
    string PaymentId,
    DateTime PaidAt);

public record OrderShipped(
    Guid OrderId,
    string TrackingNumber,
    DateTime ShippedAt);
```

Typed Handlers (consumer side)

```csharp
public class OrderCreatedHandler : IMessageHandler<OrderCreated>
{
    private readonly IInventoryService _inventory;

    public OrderCreatedHandler(IInventoryService inventory)
    {
        _inventory = inventory;
    }

    public Task Handle(OrderCreated message, CancellationToken cancellationToken)
    {
        return _inventory.ReserveStockAsync(message.OrderId);
    }
}

public class OrderPaidHandler : IMessageHandler<OrderPaid>
{
    private readonly IShippingService _shipping;

    public OrderPaidHandler(IShippingService shipping)
    {
        _shipping = shipping;
    }

    public Task Handle(OrderPaid message, CancellationToken cancellationToken)
    {
        return _shipping.InitiateAsync(message.OrderId);
    }
}

public class OrderShippedHandler : IMessageHandler<OrderShipped>
{
    private readonly INotificationService _notification;

    public OrderShippedHandler(INotificationService notification)
    {
        _notification = notification;
    }

    public Task Handle(OrderShipped message, CancellationToken cancellationToken)
    {
        return _notification.SendOrderUpdateAsync(message.OrderId, "shipped");
    }
}
```

Producer Usage (command / API side)

```csharp
public class OrderEventPublisher
{
    private readonly IMessageProducer _producer;

    public OrderEventPublisher(IMessageProducer producer)
    {
        _producer = producer;
    }

    public Task PublishAsync<T>(T evt) where T : class
    {
        var message = new Message<string, T>
        {
            Key = evt switch
            {
                { OrderId: var id } => id.ToString(),
                _ => Guid.NewGuid().ToString()
            },
            Value = evt
        };

        return _producer.ProduceAsync(message);
    }
}
```

Registration Examples

API / Command project → `Program.cs`

```csharp
builder.Services.AddKafkaFlowProducer(builder.Configuration);
```

Worker / Processor project → `Program.cs`

```csharp
builder.Services.AddKafkaFlowConsumer(builder.Configuration);
```

Key Benefits of Separation

- Independent scaling — scale producers horizontally with API instances, consumers vertically or separately
- Different resource profiles — producers are lightweight, consumers can be memory/CPU heavy
- Clear boundaries — no accidental consumer registration in web apps
- Shared config — single source of truth for brokers/topics
- Topic auto-creation — safe from both sides (idempotent)

Extend with serializers, retry policies, or schema registry independently on each side.