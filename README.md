# raft-java
Raft implementation library for Java.

Referenced from [Raft paper](https://github.com/maemual/raft-zh_cn) and the Raft author's open-source implementation [LogCabin](https://github.com/logcabin/logcabin).

# Supported Features
* Leader election
* Log replication
* Snapshot
* Dynamic cluster membership changes

## Quick Start
To deploy a 3-instance Raft cluster on a local machine, execute the following script:

cd raft-java-example && sh deploy.sh

This script will deploy three instances example1, example2, example3 in the raft-java-example/env directory;
It will also create a client directory for testing the read and write functionality of the Raft cluster.

After successful deployment, test the write operation with the following script:
cd env/client
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world

To test the read operation, use the command:
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

# Usage
Below is an introduction on how to use the raft-java dependency library in your code to implement a distributed storage system.

## Configure Dependency (Not yet published to Maven central repository, needs to be manually installed locally)
```

<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Define Data Write and Read Interfaces
```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server-side Usage
1. Implement the StateMachine interface
```java
// // These three methods of the interface are mainly called by Raft internally
public interface StateMachine {
    /**
     * Create a snapshot of the data in the state machine, called periodically by each node locally
     * @param snapshotDir output directory for snapshot data
     */
    void writeSnapshot(String snapshotDir);
    /**
     * Read snapshot into the state machine, called when the node starts
     * @param snapshotDir snapshot data directory
     */
    void readSnapshot(String snapshotDir);
    /**
     * Apply data to the state machine
     * @param dataBytes data binary
     */
    void apply(byte[] dataBytes);
}
```

2. Implement data write and read interfaces
```
// ExampleService implementation class needs to include the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```
// Main logic for data writing
byte[] data = request.toByteArray();
// Synchronously write data to the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```
// Main logic for data reading, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);
```

3. Server startup logic
```
// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Application state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, for example:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register services for Raft nodes to call each other
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft services for Client to call
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register services provided by the application itself
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start RPCServer, initialize Raft node
server.start();
raftNode.init();
```
