---
layout: post
title: JGroups Raft 2.0.x Roadmap
date: 2025-05-01 13:37 +0000
description: Expected features and enhancements for JGroups Raft 2.0.0.
categories: [jgroups, raft]
tags: [raft]
author: jab
---

Hello, everyone!

In this post, I'll share the roadmap for the release of JGroups Raft 2.0.0.Final.
This compromises many enhancements around user-facing APIs and internal updates for the core algorithm.
Let's get started; the post might be a little longer.

## New API

The current API revolves around the `RaftHandle` class with an optional `StateMachine` implementation.
To correctly implement the `StateMachine`, the user must extend the interface, add methods to serialize the objects, submit through `RAFT`/`Settable`, deserialize the payload, and apply the command.
All of this while still requiring backward compatibility with the operations.
These steps create a lot of churn and boilerplate to have even a simple example running.

In the roadmap for 2.0.0, we want to streamline this process with a new API, focusing on simplifying usage, reducing the boilerplate, and turning the focus to the `StateMachine` implementation only.
Users don't need to worry about serialization or manually submitting commands.

Below is a code sample to configure the backing JGroups channel and get a new instance of the `JGroupsRaft` interface.
This interface is the point of contact with everything internally.

```java
// First, creates a new instance of the state machine.
// The new API revolves around the state machine type and commands.
ReplicatedHashMap stateMachine = new ReplicatedHashMap();

// Now, creates an instance of JGroupsRaft to use the new API.
// To start configuration, we use the builder pattern, which requires the state machine instance.
JGroupsRaft<ReplicatedHashMap> raft = JGroupsRaft.builder(stateMachine)
        // The JGroups configuration file to use when creating the JChannel instance.
        .withJGroupsConfig("test-raft.xml")

        // The name of the cluster the nodes we'll connect to.
        .withClusterName("replicated-hash-map")

        // Register the serialization context so ProtoStream can serialize the objects.
        .registerSerializationContextInitializer(new ReplicatedHashMapSerializationContextImpl())

        // We use a custom configuration to set the raft ID and members.
        // This is utilized by JGroups when creating the stack to replace the properties.
        .withConfiguration(Configuration.create()
                .with("raft_id", "example")
                .with("raft_members", "example")
                .build())
        .build();
```

Now, the `StateMachine` implementation is required to construct the instance.
We utilize a builder pattern that creates an instance of `JGroupsRaft`.
The builder has methods to configure the backing JChannel, custom serialization, monitoring, etc.
The `JGroupsRaft` interface has the submit method to replicate commands to the state machine.

```java
// Submit read operations to the state machine.
<I, O> CompletionStage<O> submit(ReadCommand<I, O> command, I input, ReadCommandOptions options);

// Submit write operations to the state machine.
<I, O> CompletionStage<O> submit(WriteCommand<I, O> command, I input, WriteCommandOptions options);
```

The method receives a `JGroupsRaftCommand` with input of type `I` and returns an object of type `O`.
The method also receives the input to the command.
With this method, the command is replicated with Raft and will be applied to the corresponding method in the state machine implementation.
These commands also receive optional arguments to change their behavior, e.g., allowing dirty reads for read commands.
There are the respective methods with synchronous invocations and without the custom options.

### State machine and submitting commands

The new API has a different approach to implementing the state machine.
This approach utilizes an annotation to mark the methods that accept a command to apply to the state machine.
The commands have an identifier and a version number to aid in maintenance and backward compatibility.
Below is an example of a state machine (`ReplicatedHashMap`) that stores and reads values from a `Map<String, UserInformation>`.

```java
public static final class ReplicatedHashMap implements StateMachine {
    private final Map<String, UserInformation> data = new HashMap<>();

    @ReplicatedMethod(id = 1, version = 1)
    public UserInformation get(String key) {
        return data.get(key);
    }

    @ReplicatedMethod(id = 2, version = 1)
    public void put(IdToUserInformation mapping) {
        data.put(mapping.id, mapping.userInfo);
    }

    @Override
    public void readContentFrom(DataInput in) { }

    @Override
    public void writeContentTo(DataOutput out) { }
}
```

The methods have the `ReplicatedMethod` annotation with the identifier and version of the accepted commands.
We provide the factory to create the commands.
These commands can be created statically since they don't have mutable state.
Below is an example of utilizing the factory to build the commands to access the state machine:

```java
// Defines a read command to retrieve the UserInformation from the state machine.
// This command has ID of 1 and version 1. It receives a String as input and returns a UserInformation object.
private static final ReadCommand<String, UserInformation> GET_USER_INFO_REQUEST =
        ReadCommand.create(1, 1, String.class, UserInformation.class);

// Defines a write command to store the UserInformation in the state machine.
// This command has ID of 2 and version 1. It receives an IdToUserInformation object as input and returns Void.
// The IdToUserInformation is mapping of the ID for the user. This is needed because the input is always a SINGLE object.
private static final WriteCommand<IdToUserInformation, Void> PUT_USER_INFO_REQUEST =
        WriteCommand.create(2, 1, IdToUserInformation.class);
```

Below is an example of how to submit the commands utilizing the `JGroupsRaft` interface created earlier:

```java
String userId = "1234";

// Submit a read request to the state machine to retrieve the user.
raft.submit(GET_USER_INFO_REQUEST, userId, 10, TimeUnit.SECONDS);

// Now, we can update the user information for the given ID.
UserInformation ui = new UserInformation("user", 42);
raft.submit(PUT_USER_INFO_REQUEST, new IdToUserInformation(userId, ui), 10, TimeUnit.SECONDS);
```

With this new API the boilerplate, which was quite error-prone, is no longer needed.
We are utilizing annotations but would welcome any feedback on alternatives that seem more ergonomic.

#### Questions

Additional internal details I thought of implementing is an internal state machine registry flushed to disk.
This registry is utilized to validate that a state machine is compatible with a previous version.
I still haven't thought deeply about this and not sure if it will actually happen.

And clearly about the automatic serialization happening which I am using [ProtoStream](https://github.com/infinispan/protostream).
The builder for the `JGroupsRaft` interface has methods to register marshallers of custom classes.

Here is [an example](https://github.com/jabolina/jgroups-raft/blob/v2/new-api/src/test/java/org/jgroups/tests/examples/ReplicatedHashMapExample.java) using the new API.

### Cluster management

We provide a dedicated administration API.
This API handles tasks for operators to configure and manage the Raft cluster.
These operations include adding or removing members, and triggering leader election.
The API name will contain "Admin" or "Maintenance" to be explicit that such operations should be utilized on specific occasions and with care.

All in all, this is just moving the API in `RaftHandle` to something more specific.
I'll ensure this has extensive documentation about how and when to use it.

### Cluster monitoring

In the same line as something explicit for management, we want to make people's lives easier when dealing with a Raft cluster.
Systems that require consensus are usually critical, and as such, they require extensive monitoring to ensure everything is working as it should.
We want to expose different APIs to help with monitoring.

Currently, the metrics we expose with JMX are in the `RAFT` and election implementations.
We want to expand the data and make it more easily accessible.
The final goal is to include:

* **Election metrics**: who's the leader, when it was elected, how many elections since the application started, last leader change, how long it took.
* **Performance metrics**: the latency to process an application command and latency to replicate internal commands. These are broken in percentiles.
* **Internal metrics**: these are metrics related to how Raft is working. They include the total log entries, entries committed, entries uncommitted, and delay of each node in the cluster.

These numbers should help create dashboards for monitoring.
Also, these values are not exposed following any specific metric format, they are the raw data.
The goal is for anyone with a monitoring system to consume these values as necessary.

Additionally, we want to provide an interface for health checks.
Therefore, any application utilizing a probe could easily retrieve values for health and readiness.
For example, the local node is healthy because the channel is created and waiting for connections, but it is not ready yet because there is no quorum.

Lastly, we'll start publishing events for meaningful operations.
These events should include leader elections, cluster changes, membership changes, snapshots of the state machine, etc.
We'll provide a disabled implementation and another exposing these events to JFR.
The goal is for the application to provide the implementation to export these events elsewhere.

## Extras

This section includes noteworthy miscellaneous additions.

### Read-only operations

Distinguish between read-only operations internally to avoid creating an entry in the log. This implementation follows the dissertation 6.4. 
We also want to expose in the API for read-only operations to completely avoid the Raft algorithm an perform local dirty reads. This is configurable per command invocation.

In the API we show earlier there is a clear distinction of read and write operations.
Our goal is to avoid creating a log entry for read operations.
We want to implement the suggestion in the Raft dissertation in Section 6.4.
We had a [GitHub issue](https://github.com/jgroups-extras/jgroups-raft/issues/18) for this optimization for some time and thought it would be the time to implement it.

With the new API, this makes it more flexible to change between quorum and dirty reads.

### Learner nodes

We want to introduce learner nodes.
This is also mentioned in Ongaro's dissertation in Section 4.2.1 about the availability gap.
The goal of learner nodes is to create nodes that do not count toward quorum.
These nodes allow the cluster to expand and could create hot replicas to step in during maintenance, for example.

## Conclusion

Our goal with version 2.0.0 for JGroups Raft is to improve the developer and operator experience in building and running a system.
We are proposing a new API to handle these problems.

I would love to gather feedback on this.
As always, none of this is set in stone.
We are blogging about this to evolve the project with suggestions from the community.
