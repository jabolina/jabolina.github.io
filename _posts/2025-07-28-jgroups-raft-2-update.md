---
layout: post
title: JGroups Raft 2 Update
date: 2025-07-28 22:41 +0000
description: Development update of JGroups Raft 2.0.0.
categories: [jgroups, raft]
tags: [raft]
author: jab
---

Hello, everyone!

This post is a development update of JGroups Raft 2. I have a previous post here where I share the ideas we have in mind for the next major release. This post continues where I left off, it includes:

- Refinement in the API to submit commands.
- Annotation-focused API.
- Automatic snapshot.

Let’s get into it.

## State machine definition

Previously, the user could provide only the concrete state machine implementation, and with the extra burden we discussed in the following sections, the library would invoke the correct method.
Although this is a straightforward configuration, it moves the effort to other areas that become brittle.

In the current version, the user must provide the state machine API in an interface and the local concrete implementation.
For example:

```java
// User information is utilizing ProtoStream annotations for serialization.
@Proto
public record UserInformation(String name, int age) { }

// This is the state machine definition.
// It is only an interface with the proper annotations.
@JGroupsRaftStateMachine
public interface StateMachineApi {
    @StateMachineRead(id = 1)
    UserInformation get(String key);

    @StateMachineWrite(id = 2)
    void put(String key, UserInformation value);
}

// The concrete implementation.
// This implementation is local to the node running.
public static final class ReplicatedHashMap implements StateMachineApi {
    /**
     * We can utilize a simple hash map, no need for concurrent.
     */
    @StateMachineField(order = 0)
    Map<String, UserInformation> data = new HashMap<>();

    @Override
    public UserInformation get(String key) {
        return data.get(key);
    }

    @Override
    public void put(String key, UserInformation value) {
        data.put(key, value);
    }
}
```

The user creates the state machine definition with the interface and annotations, and an implementation local to the node.
The local implementation doesn’t need to be thread-safe, but it must always guarantee determinism when handling operations.

Currently, the state machine must have the `@JGroupsRaftStateMachine` annotation and identify the methods with the proper annotations.
The return methods can be synchronous or asynchronous, and will be handled properly by the library.

You can also notice the `@StateMachineField` (not the best naming) annotation in the concrete implementation.
This annotation is utilized during snapshot to flush and read from disk.
The requirement is that the field can not be final, since it can be overwritten when loading the snapshot.

The other change is how these values are provided to the JGroups Raft builder. Here is the builder for our example:

```java
ReplicatedHashMap stateMachine = new ReplicatedHashMap();
JGroupsRaft<StateMachineApi> raft = JGroupsRaft.builder(stateMachine, StateMachineApi.class)
        // The JGroups configuration file to use when creating the JChannel instance.
        .withJGroupsConfig("test-raft.xml")

        // The name of the cluster the nodes we'll connect to.
        .withClusterName("replicated-hash-map")

        // Register the serialization context so ProtoStream can serialize the objects.
        .registerSerializationContextInitializer(new ReplicatedHashMapSerializationContextImpl())

        // We use a custom configuration to set the raft ID and members.
        .configureRaft()
            .withRaftId("example")
            .withMembers(Collections.singletonList("example"))
            .withLogClass(InMemoryLog.class)
            .and()

        // And finally, we build the JGroupsRaft instance.
        .build();
```

The configuration hasn’t changed much since the last post.
The changes here are the construction; in addition to an instance of the state machine, it requires the API definition.

Another inclusion is the builder to configure the `RAFT` protocol at the JGroups level.
The goal is to also include the configuration for leader election.

## Command submission

Initially, we had the idea of an interface to use as a factory to create read and write commands.
Something like this:

```java
private static final ReadCommand<String, UserInformation> GET_USER_INFO_REQUEST =
        ReadCommand.create(1, 1, String.class, UserInformation.class);

private static final WriteCommand<IdToUserInformation, Void> PUT_USER_INFO_REQUEST =
        WriteCommand.create(2, 1, IdToUserInformation.class);

// Now submit the commands.
String userId = "1234";

// Submit a read request to the state machine to retrieve the user.
UserInformation v = raft.submit(GET_USER_INFO_REQUEST, userId, 10, TimeUnit.SECONDS);

// Now, we can update the user information for the given ID.
UserInformation ui = new UserInformation("user", 42);
raft.submit(PUT_USER_INFO_REQUEST, new IdToUserInformation(userId, ui), 10, TimeUnit.SECONDS);
```

We noticed this still has some boilerplate to create the commands, and quite a few drawbacks.
To name a few: it needs the ID in the factory to match the ID in the annotation, and it only handles zero/single parameter methods.
To improve the experience, we replace this API with a lambda/proxy client around the user’s interface.
It becomes something like:

```java
String userId = "1234";

// Submit a read request to retrieve the user.
UserInformation v = raft.read(api -> api.get(userId));

// Now, update the user information.
// We are not restricted to a single argument anymore.
raft.write(api -> api.put(userId, new UserInformation("user", 42)));
```

> Only the API invocation is submitted through Raft, and **not** the lambda.
{: .prompt-warning }

In addition to these invocations with lambda, the JGroups Raft interface also provides methods with custom options to change the invocation behavior.
For example:

```java
JGroupsRaftReadCommandOptions options = JGroupsRaftReadCommandOptions.options()
    .linearizable(false)
    .build();
raft.read(api -> api.get(userId), options);
```

In this example, the read operation is not linearizable; it can be executed locally without going through Raft.
This allows customization per invocation, with custom properties for write operations.
And lastly, instead of invoking with custom options per method, you can create the proxy client with a fixed set of options:

```java
// Creates the read-only proxy client with a fixed set of options.
JGroupsRaftReadCommandOptions options = JGroupsRaftReadCommandOptions.options()
    .linearizable(false)
    .build();
StateMachineApi readOnlyApi = raft.readOnly(options);
// Read-only client. Utilizing the methods with write annotations will throw an exception.
readOnlyApi.get(userId);
```

## Automatic snapshot

We use the `@StateMachineField` to identify a field holding some state in the state machine in the `ReplicatedHashMap` implementation.
This will automatically pick up the fields and serialize them in the provided order parameter.
You can define multiple fields simultaneously, each with a different serialization order.

The main requirement, which also fits to serialize any data class, is to have a marshaller registered.
You can utilize [ProtoStream](https://github.com/infinispan/protostream) to quickly create your marshalers.
Otherwise, you can provide a custom marshaller in the JGroups Raft builder.
Always keep backwards compatibility in mind, as a new marshaller could read data written by an older version.

## Conclusion

This blog explains the current state of development for the JGroups Raft 2 API.
We believe the form shouldn’t change much now, which opens the possibility for a beta release.
The next steps will involve internal improvements in the RAFT protocol level.

If you want to check the full code for the above example, you can find it here:
[jgroups-extras/jgroups-raft#ReplicatedHashMapExample.java](https://github.com/jabolina/jgroups-raft/blob/07381212b2fcdd846c8e4015efe314aa403c1ae3/src/test/java/org/jgroups/tests/examples/ReplicatedHashMapExample.java)

Also, I’ve adapted a small demo from JGroups to utilize Raft.
This is the total order.
It creates a matrix on each node and submits a random mathematical operation in total order to apply to a single cell.
The full code is available here: [jgroups-extras/jgroups-raft#TotalOrder.java](https://github.com/jabolina/jgroups-raft/blob/07381212b2fcdd846c8e4015efe314aa403c1ae3/src/test/java/org/jgroups/demo/TotalOrder.java)

We can create a JBang script to run it after a beta release.
As always, thanks for reading, and any feedback is welcome!
