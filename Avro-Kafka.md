
### What is Avro?

Avro is an open source data serialization system that helps with data exchange between systems, programming languages, and processing frameworks. <br>
Avro helps define a binary format for your data, as well as map it to the programming language of your choice. <br>

### Why Use Avro with Kafka?

Avro has a JSON like data model, but can be represented as either JSON or in a compact binary form. It comes with a very sophisticated schema description language that describes data.

We think Avro is the best choice for a number of reasons:

- It has a direct mapping to and from JSON
- It has a very compact format. The bulk of JSON, repeating every field name with every single record, is what makes JSON inefficient for high-volume usage.
- It is very fast.
- It has great bindings for a wide variety of programming languages so you can generate Java objects that make working with event data easier, but it does not require code generation so tools can be written generically for any data stream.
- It has a rich, extensible schema language defined in pure JSON
- It has the best notion of compatibility for evolving your data over time.
