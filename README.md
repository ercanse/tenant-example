# DSH tenant example

This example shows how tenants on the DSH platform should interact with the DSH
data streams. The particular points to pay attention to are:
- getting Kafka configuration from the kafka configuration service
- key and data envelopes for serialization/deserialization on public streams
  (the `stream.<something>` streams).
- partitioning for public streams.

## Kafka configuration

On startup, every DSH container that wants to interact with the data streams
needs to fetch its kafka configuration from the kafka configuration service
(pikachu.dsh.marathon.mesos).

**Fetching the Kafka configuration *must* happen on container startup.** It
depends on a limited-lifetime token that is configured in the container's
environment on startup. If you wait for the token to expire, there is no way
for the container to ever fetch its Kafka configuration.

The full flow for fetching the Kafka configuration is a rather involved
multi-step process. Documenting it here in its entirety will lead us too far,
but the good news is that this example contains a fully working shell script
that does all the work for you. You can copy it into your own containers and
invoke it from the entrypoint.

See the `get_signed_certificate.sh` script.

### Consumer groups

In order to consume data from Kafka, you need to set a consumer group for your
consumers. How you choose your consumer group has a profound impact on the 
operation of your program:
- if you choose a unique consumer group name, your program will see all data
  from all partitions of the topics you subscribe to.
- if you share a consumer group with other programs, each program will see
  a subset of the total data on the kafka topics you subscribe to. Kafka 
  balances the partitions among all members of a consumer group automatically.

#### Consumer group enforcement

The DSH platform enforces restrictions on the consumer groups each container
is allowed to join. When you retrieve the Kafka configuration from the Kafka
configuration service, it will include a list of consumer group names you are
allowed to use. To cater for the two options described above, the list is
split in two parts:
- **private** consumer groups are unique to this particular container, and
  can be used to consume all data on a Kafka topic.
- **shared** consumer groups are shared among all instances of this container,
  to allow you to spin up multiple instances and balance the load of a (set
  of) Kafka topics among them.

Note that the DSH platform does not currently offer provisions for sharing a 
consumer group among instances of different containers.

See [below](#configuration-data) for a description of how the consumer group
names are encoded in the `datastreams.properties` file.

#### Setting the consumer group

To set a specific consumer group, pass the `group.id` property to the Kafka
Consumer constructor.

### Configuration data

The configuration data fetched by the `get_signed_certificates.sh` script
consists of an SSL certificate to be used for connection to Kafka on the one
hand, and a Java properties file that contains the necessary security settings, 
a list of bootstrap servers and a description of the streams you have access to 
on the other hand.

On startup, load the properties from the `datastreams.properties` file
generated by the script, and pass them along to any Kafka consumer or producer
you create.

To learn more about the streams you have access to, you can inspect the
contents of the properties file. All configuration entries for a given stream
are prefixed with `datastream.<streamname>`. Note that the streamname includes
the stream type prefix, so concrete examples are `datastream.stream.foo` for a
public stream called "foo" and `datastream.internal.bar` for an internal stream
named "bar".

For each stream, the following configuration
values are defined:
- `datastream.<streamname>.cluster`: identifies the Kafka cluster carrying this
  data stream (right now, the only supported value is "tt")
- `datastream.<streamname>.partitioner`: partitioning scheme used on this
  stream (see below). Allowed values are: `"default-partitioner"`,
  `"topic-level-partitioner"` and `"topic-char-depth-partitioner"`.
- `datastream.<streamname>.canretain`: indicates whether this stream supports
  retained MQTT messages. This is only applicable to public streams. Allowed
  values are `"true"` and `"false"`
- `datastream.<streamname>.partitions`: number of Kafka partitions for the
  stream
- `datastream.<streamname>.replication`: Kafka replication factor for the
  stream
- `datastream.<streamname>.partitioningDepth`: the partitioning depth
  configuration parameter for the topic level and character depth partitioners.
  This value has no meaning for the default partitioner.
- `datastream.<streamname>.read`: the **regex pattern** you can use to
  subscribe to this stream on Kafka. You should always subscribe with a regex
  pattern, to ensure you subscribe to all Kafka topics that underly this data
  stream. If this configuration value is empty, it means you do not have read
  access to this stream.
- `datastream.<streamname>.write`: the Kafka topic name you can use to produce
  data on the stream. If this configuration value is empty, it means that you
  do not have write permissions for this stream.

In addition to the stream descriptions, the `datastreams.properties` file
defines the following helpful values:
- `consumerGroups.private`: a comma-separated list of "private" consumer group names
  your container is allowed to use. A private consumer group is one that is not 
  shared with any other container in the system, even other instances of the same
  container.
- `consumerGroups.shared`: a comma-separated list of "shared" consumer group names
  your container is allowed to use. A shared consumer group is one that is available
  for use in all instances of the same container.
- `envelope.streams`: a comma-separated list of public streams that do _NOT_
  yet have envelopes. Use this value to configure your (de)serializers to
  expect the right kind of data (enveloped or non-enveloped) on each topic.
  Note that the list contains only the middle part of the stream name, not the
  `stream` prefix or tenant name suffix.


## Envelopes on public data streams

Public data streams (the `stream.<something>` streams) have specific
requirements with respect to the structure of the key and payload values. In
essence, the "plain" key and payload are wrapped in envelope structures. The
envelope structures are defined as Protobuf messages. The definition can be
found in `src/main/proto/envelope.proto`.

**Note:** at this moment, the DSH platform is in a transition phase where the
use of envelopes on the public topics is gradually phased in. Right now,
envelopes MUST be used on all Kafka topics ending in `.dsh`, and MAY NOT be
used on any other topics. The example code deals with this transparently, and
allows you to write your application code as if envelopes are mandatory
everywhere.

**Note:** while the usage of envelopes on the public streams is strictly
regulated and enforced, you are free to choose whether you want to use
envelopes on the other stream types (scratch, internal). You are not forced to
do it, and also not forbidden to if you feel it is useful to do so.

### Envelope definition

The envelope definition can be found [here](doc/envelopes.md).

### Envelope usage

The Kafka serializers and deserializers for enveloped keys and data can be
found in the `KeyEnvelopeSerializer`, `KeyEnvelopeDeserializer`,
`DataEnvelopeSerializer` and `DataEnvelopeDeserializer` classes. As noted
above, these implementations deal automatically with enveloped (.dsh) and
non-enveloped (all other) topics. *If you reuse these implementations in your
own containers, take care to remove this logic when envelopes become mandatory
for all public data streams.*

#### Convenience methods

For your convenience, the serializer classes define some utility methods that
facilitate wrapping of plain keys and payloads in the envelopes:
- `KeyEnvelopeSerializer.setIdentifier()` is used to set the static identifier
  (tenant id and free-form publisher id) for every message produced by this container.
- `KeyEnvelopeSerializer.wrap()` wraps a plain string key in an envelope, with
  optional `qos` and `retain` settings (defaults are QoS 0 and non-retained).
- `DataEnvelopeSerializer.wrap()` wraps a plain byte array payload in a data
  envelope.

## Partitioning for public streams

If your container publishes data on a public stream, it needs to abide by the
stream contract, which defines (amongst others) the partitioning scheme for
this stream.

Currently, there are three supported partitioning schemes:
- topic level partitioning: the first `n` levels of the topic path (i.e. the
  plain string key) are hashed using the Murmur2 hashing scheme
- character level partitioning: the first `n` characters of the topic path are
  hashed using the Murmur2 hashing scheme
- default partitioning: Murmur2 hash of the entire topic path

As of now, the only partitioning scheme that is used in practice on the public
streams is the topic level partitioner. Therefore, that is the only partitioner
for which we have included an example implementation, in the
`TopicLevelPartitioner` class.

The `TopicLevelPartitioner` class will figure out the partitioning depth (the
aforementioned `n` parameter) from the configuration values you fetched from
the Kafka configuration service, and passed to your Kafka producer constructor.
