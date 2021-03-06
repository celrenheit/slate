---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - go

toc_footers:
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

# includes:
#   - errors

search: true
---

# Introduction

Welcome to the Sandglass API documentation.

# Topics

## Creating a topic


> To create a topic:

```shell
sandctl topics create emails --num_partitions 6 --replication_factor 3
```

```go
client, err := sg.NewClient(
    sg.WithAddresses(":7170"),
)
if err != nil {
    return err
}

err = client.CreateTopic(context.Background(), &sgproto.CreateTopicParams{
  Name: "emails",
  NumPartitions: 6,
  ReplicationFactor: 3,
})
if err != nil {
    return err
}
```

> Make sure to replace `:7170` with the address of at least one node in the cluster

A topic has a name, a number of partitions and a replication factor.
Here is a table describing the required/optional params:

Parameter | Required | Description
--------- | ------- | -----------
name | true | The name of the topic, must not start with an underscore `_` and only be composed of alphanumeric characters
num partitions | true | The number of partitions
replication factor | true | The number of nodes to replicate each partition to
kind | false | The topic kind. Default: **Timer**
storage driver | false | The storage engine to use (RocksDB or Badger). Default: **RocksDB**

# Producing a message

## Now

```shell
sandctl produce emails '{"dest" : "hi@example.com"}'
```

```go
client, err := sg.NewClient(
    sg.WithAddresses(":7170"),
)
if err != nil {
    return err
}

err = client.Produce(context.Background(), "emails", "", &sgproto.Message{
    Value: []byte(`{"dest" : "hi@example.com"}`),
})
if err != nil {
    return err
}
```

When producing a new message to a topic we need to specify to which partition the message should be produced in.

However, an empty partition means choose a random one. This is for convenience and when there is not need to partition in a particular partition.

## In the future

```go
client, err := sg.NewClient(
    sg.WithAddresses(":7170"),
)
if err != nil {
    return err
}

// Now we produce a new message that will be consumed in one hour
inOneHour := time.Now().Add(1 * time.Hour)
gen := sandflake.NewFixedTimeGenerator(inOneHour)

msg := &sgproto.Message{
	Offset: gen.Next(),
	Value:  []byte("Hello"),
}

err := client.ProduceMessage(context.Background(), "emails", "", msg)
if err != nil {
	return err
}
```

The way to produce a message in future, is by setting a custom [sandflake id](https://github.com/celrenheit/sandflake) with the time that you want.

# Consuming


```shell
sandctl consume emails
```

```go
client, err := sg.NewClient(
    sg.WithAddresses(":7170"),
)
if err != nil {
    panic(err)
}
defer client.Close()

mux := sg.NewMux()
mux.SubscribeFunc("emails", func(msg *sgproto.Message) error {
    // handle message
    log.Printf("received: %s\n", string(msg.Value))
    return nil
})

m := &sg.MuxManager{
    Client:               c,
    Mux:                  mux,
    ReFetchSleepDuration: dur,
}

err = m.Start()
if err != nil {
    log.Fatal(err)
}
```


In Go, when using Handlers and Subscribe(Func) methods, each message received will launch a new goroutine.

<aside class="notice">
If the handler returns non-nil error the message will be <code>NACKed</code> and made available for redelivery. Otherwise, the message will be <code>ACK</code>.
</aside>
