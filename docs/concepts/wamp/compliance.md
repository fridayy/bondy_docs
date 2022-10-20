# Compliance
> Bondy tries to be as compliant as possible with the WAMP Specification. However, because the specification was not designed from the ground up to consider a distributed router there are some requirements we cannot (and/or we do not want to) be compliant with.

The following two sections go into a detail compliance analysis listing what WAMP features Bondy implement and those cases in which Bondy does not comply with the Specification. In those particular cases you will also find an explanation of the rationale and the roadmap behind them.

## Basic Profile

### General Features
|Feature|Status|Extensions|Roadmap|
|---|---|---|---|
|Web Socket transport|:heavy_check_mark:|||
|Batched Web Socket transport|:x:||Yes|
|Raw Socket transport|:heavy_check_mark:|||
|Transport and Session Lifetime|:heavy_check_mark:<br>Each session establishes a new transport connection|||
|Close session and connection on protocol errors|:heavy_check_mark:|||
|Serializations: Support JSON and Msgpack|:heavy_check_mark:| BERT, Erlang (subset)|CBOR, Flatbuffers|
|Validation of custom attribute keys using regex|:heavy_check_mark:|||
|No Polymorphism, avoid empty arguments and keyword arguments|:heavy_check_mark:|||
|Agent Identification|:heavy_check_mark:|||

### Pub/Sub Features

|Feature|Status|Extensions|Roadmap|
|---|---|---|---|
|Ordering Guarantees| Partial, see [NC1](#nc1)||Yes|
|Subscription (and Subscription ID) to be shared amongst subscribers to the same topic.|:x:<br>see [NC2](#nc2)||No|

### Routed RPC Features

## Advanced Profile

## Authentication
|Feature|Status|Extensions|Roadmap|
|---|---|---|---|
|Challenge-response authentication|:heavy_check_mark:||
|Ticket authentication|:heavy_check_mark:|Allows [Single Sign-on](/concepts/single_sign_on) to multiple realms|
|Cookie authentication|:x:||No|

## Non-compliance Cases

### NC1

<tabs cache-lifetime="1000">
<tab name="Requirement">

**Pub/Sub: Ordering Guarantees**

The ordering guarantees are as follows:

If _Subscriber A_ is subscribed to both **Topic 1** and **Topic 2**, and Publisher B first publishes an **Event 1** to **Topic 1** and then an **Event 2** to **Topic 2**, then _Subscriber A_ will first receive **Event 1** and then **Event 2**. This also holds if **Topic 1** and **Topic 2** are identical.

In other words, WAMP guarantees ordering of events between any given pair of Publisher and Subscriber.

Further, if _Subscriber A_ subscribes to **Topic 1**, the `SUBSCRIBED` message will be sent by the Broker to _Subscriber A_ before any `EVENT` message for **Topic 1**.

There is no guarantee regarding the order of return for multiple subsequent subscribe requests. A subscribe request might require the Broker to do a time-consuming lookup in some database, whereas another subscribe request second might be permissible immediately.

</tab>
<tab name="Implementation">
A Bondy node will deliver the events in order locally i.e. when Pulisher and Subscriber are on the same node. However, when it comes to publishing events to a remote subscriber, Bondy tries its best to deliver them in the order required by the protocol, but this is not currently guaranteed.
</tab>
<tab name="Rationale">
Bondy was designed as a distributed router with continuous availability as its main goal. Bondy uses an eventually consistent model avoiding coordination between nodes at all cost.

</tab>
<tab name="Roadmap">
Bondy plans to implement stronger delivery order guarantees to comply with the protocol requirements and also enable new features, such as guaranteed message delivery.
</tab>
</tabs>

### NC2

<tabs cache-lifetime="1000">
<tab name="Requirement">

**Pub/Sub: Subscription (and Subscription ID) to be shared amongst subscribers to the same topic**

The WAMP protocol requires a subscription to be shared amogst subscribers to the same topic.

A subscription is created when a client sends a subscription request for a topic where there are currently no other subscribers. It is deleted when the last subscriber cancels its subscriptions, or its session is disconnected.

A subscriber receives a subscription ID as the result of a successful subscription request. When an second subscriber issues a subscription request for the same topic, then it receives the **same** subscription ID.

</tab>
<tab name="Implementation">
A subscription is created when a client sends a subscription request for a topic when itself does not have an existing subscription for the same topic. It is deleted when the subscriber cancels the subscriptions, or its session is disconnected.

A subscriber receives a subscription ID as the result of a successful subscription request. When an second subscriber issues a subscription request for the same topic, then it receives a **different** subscription ID.
</tab>
<tab name="Rationale">
Bondy was designed as a distributed router with continuous availability as its main goal. Bondy uses an eventually consistent model avoiding coordination between nodes at all cost.

Sharing a subscription between two or more subscribers in at least two cluster nodes will require coordination (consensus) and thus it is not support it.
</tab>
<tab name="Roadmap">
No changes planned.
</tab>
</tabs>