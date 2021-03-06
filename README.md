# Bravo

## Introduction

Bravo is a convenient state reader and writer library leveraging the Flink’s
batch processing capabilities. It supports processing and writing Flink streaming savepoints.
At the moment it only supports processing RocksDB savepoints but this can be extended in the future for other state backends and checkpoint types.

Our goal is to cover a few basic features:
 - Converting keyed states to Flink DataSets for processing and analytics
 - Reading/Writing non-keyed operators states
 - Bootstrap keyed states from Flink DataSets and create new valid savepoints
 - Transform existing savepoints by replacing/changing some states

Some example use-cases:
 - Point-in-time state analytics across all operators and keys
 - Bootstrap state of a streaming job from external resources such as reading from database/filesystem
 - Validate and potentially repair corrupted state of a streaming job
 - Change max parallelism of a job

## Disclaimer

This is more of a proof of concept implementation, not necessarily something production ready.

Who am I to tell you what code to run in prod, I have to agree, but please double check the code you are planning to use :)

## Reading states

### Reading and processing keyed states

The `KeyedStateReader` provides DataSet input format that understands RocksDB savepoints and can extract state rows from it. The input format creates input splits by operator subtask of the savepoint at the moment but we can change this to split by keygroups directly.

```java
// First we start by taking a savepoint of our running job...
// Now it's time to load the metadata
Savepoint savepoint = StateMetadataUtils.loadSavepoint(savepointPath);

ExecutionEnvironment env = ExecutionEnvironment.getEnvironment();

// We create a KeyedStateReader for accessing the state of the operator CountPerKey
KeyedStateReader reader = new KeyedStateReader(savepoint, "CountPerKey", env);

// The reader now has access to all keyed states of the CountPerKey
// We are going to read one specific value state named Count
// The DataSet contains the key-state tuples from our state
DataSet<Tuple2<Integer, Integer>> countState = reader.readValueStates(
		ValueStateReader.forStateKVPairs("Count", new TypeHint<Tuple2<Integer, Integer>>() {}));

// We can now work with the countState dataset and analyize it however we want :)
```

### Accessing non-keyed states

There is also a convenient reader for non-keyed operator states that will restore that state in-memory. This assumes that the machine that we run the code has enough memory to do this. (This is mostly a safe assumption with the current operator state design)

```java
ManagedOperatorStateReader opStateReader = new ManagedOperatorStateReader(savepoint, "opUid");

// We restore the OperatorStateBackend in memory
OperatorStateBackend stateBackend = opStateReader.createOperatorStateBackendFromSnapshot(0);

// Now we can access the state just like from the function
stateBackend.getListState(...)
stateBackend.getBroadcastState(...)

```
## Creating new savepoints

### StateTransformer

As the name suggests the `StateTransformer` class provides utilities to change (replace/transform) the state contained in a savepoint to create a new valid savepoint from where the job can continue processing.

Let's continue our reading example by modifying the state of the all the users, then creating a new valid savepoint.
```java
DataSet<Tuple2<Integer, Integer>> countState = //see above example

// We want to change our state based on some external data...
DataSet<Tuple2<Integer, Integer>> countsToAdd = environment.fromElements(
        Tuple2.of(0, 100), Tuple2.of(3, 1000),
        Tuple2.of(1, 100), Tuple2.of(2, 1000));

// These are the new count states we want to put back in the state
DataSet<Tuple2<Integer, Integer>> newCounts = countState
        .join(countsToAdd)
        .where(0)
        .equalTo(0)
		.map(new SumValues());

// We create a statetransformer that will store new checkpoint state under the newCheckpointDir base directory
StateTransformer stateBuilder = new StateTransformer(savepoint, newCheckpointDir);

// As a first step we have to serialize our Tuple K-V state with the provided utility
DataSet<KeyedStateRow> newStateRows = stateBuilder.createKeyedStateRows("CountPerKey", "Count", newCounts);

// In order not to lose the other value states in the "CountPerKey" operator we have to get the untouched rows from the reader
stateBuilder.replaceKeyedState("CountPerKey", newStateRows.union(reader.getUnparsedStateRows()));

// Last thing we do is create a new meta file that points to a valid savepoint
Path newSavepointPath = stateBuilder.writeSavepointMetadata();
```

We can also use the StateTransformer to transform and replace non-keyed states:

```java
stateBuilder.transformManagedOperatorState(String uid, BiConsumer<Integer, OperatorStateBackend> transformer);
```

Modifications made to the state in the consumer are then stored back to the new snapshot.

## Contact us!

The original King authors of this code:
 - David Artiga (david.artiga@king.com)
 - Gyula Fora (gyula.fora@king.com)

With any questions or suggestions feel free to reach out to us in email anytime!
