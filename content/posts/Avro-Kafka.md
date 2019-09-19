+++
title = "Avro  with  Kafka Introduction"
date = "2019-09-12"
+++

Ref: https://www.confluent.io/blog/avro-kafka-data/

### What is Avro

Avro is an open source data serialization system that helps with data exchange between systems, programming languages, and processing frameworks. <\br>
Avro helps define a binary format for your data, as well as map it to the programming language of your choice. <\br>

### Why Use Avro with Kafka

Avro has a JSON like data model, but can be represented as either JSON or in a compact binary form. It comes with a very sophisticated schema description language that describes data.

We think Avro is the best choice for a number of reasons:

- It has a direct mapping to and from JSON
- It has a very compact format. The bulk of JSON, repeating every field name with every single record, is what makes JSON inefficient for high-volume usage.
- It is very fast.
- It has great bindings for a wide variety of programming languages so you can generate Java objects that make working with event data easier, but it does not require code generation so tools can be written generically for any data stream.
- It has a rich, extensible schema language defined in pure JSON
- It has the best notion of compatibility for evolving your data over time.

One of the critical features of Avro is the ability to define a schema for your data. For example an event that represents the sale of a product might look like this:

```json
{
  "time": 1424849130111,
  "customer_id": 1234,
  "product_id": 5678,
  "quantity": 3,
  "payment_type": "mastercard"
}
```

It might have a schema like this that defines these five fields:

```json
{
  "type": "record",
  "doc": "This event records the sale of a product",
  "name": "ProductSaleEvent",
  "fields": [
    { "name": "time", "type": "long", "doc": "The time of the purchase" },
    { "name": "customer_id", "type": "long", "doc": "The customer" },
    { "name": "product_id", "type": "long", "doc": "The product" },
    { "name": "quantity", "type": "int" },
    {
      "name": "payment",
      "type": {
        "type": "enum",
        "name": "payment_types",
        "symbols": ["cash", "mastercard", "visa"]
      },
      "doc": "The method of payment"
    }
  ]
}
```

A real event, of course, would probably have more fields and hopefully better doc strings, but this gives their flavor.

### Effective Avro

Here are some recommendations specific to Avro:

- Use enumerated values whenever possible instead of magic strings. Avro allows specifying the set of values that can be used in the schema as an enumeration. This avoids typos in data producer code making its way into the production data set that will be recorded for all time.
- Require documentation for all fields. Even seemingly obvious fields often have non-obvious details. Try to get them all written down in the schema so that anyone who needs to really understand the meaning of the field need not go any further.
- Avoid non-trivial union types and recursive types. These are Avro features that map poorly to most other systems. Since our goal is an intermediate format that maps well to other systems we want to avoid any overly advanced features.
- Enforce reasonable schema and field naming conventions. Since these schemas will map into Hadoop having common fields like customer_id named the same across events will be very helpful in making sure that joins between these are easy to do. A reasonable scheme might be something like PageViewEvent, OrderEvent, ApplicationBounceEvent, etc.
