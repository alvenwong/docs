This file briefly introduce the background and how [Basic Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) reaches consensus in distributed systems.
# Background
## Consensus
Refer to [Wikipedia](https://en.wikipedia.org/wiki/Consensus_(computer_science)), 
the consensus problem requires agreement among a number of processes for a single data value. 
As some of the processes may fail or be unreliable, consensus protocols must be fault tolerant or resilient. 
The processes must somehow put forth their candidate values, communicate with one another, 
and agree on a single consensus value.
The distributed system should have a consistent view of all the values at any replica.

## An example
The figure illustrates what consensus is and why it is necessary.

In a distributed system, such as file backup, there are multiple proposers asking to logging the values at almost the same time.
Given Proposers A, B, C, D and their values are *add*, *jmp*, *cmp* and *sub* respectively. 
All the values need to be recorded into logs in replicas in the same order, otherwise it will incur inconsistency in the system. <p>
In this scenario, consensus refers to that one value is accepted by all replicas in one round. 
Generally, it requires multiple rounds for the replicas to accept all the values in the same order in Basic Paxos.
Without consensus, due to the complexity of the network, 
it is likely that values from different proposers arrive at replicas in random order.
For example, the proposing order of the four values is exactly *add*, *jmp*, *cmp* and *sub*. 
However, their arriving order at replica 1 is *add*, *cmp*, *jmp* and  while at replica 2 could be *sub*, *cmp*, *jmp* and *add*.

## Strawman solutions
An intuitive approach to provide consensus in a distributed system is to pick up a acceptor, 
who serves to deside the order of values update.
The figure shows the overview of this strawman solution. 

Instead of directly broadcasting values to all replicas, proposers first send the values to the acceptor.
It ranks the values with a certain algorithm and then forwards them in a order, ensuring the same order of values logged in all replicas.
However, this acceptor is likely to go down unexpectedly at any time.
As a result, multiple primaries are necessary, as shown in the following figure.

# Paxos
## Three entities
### Proposers
Receive requests (values) from clients and try to convince acceptors to accept their proposed values. <p>
### Acceptors
Accept certain proposed values from proposers and let proposers know if something else was accepted. 
A response from an acceptor represents a vote for a particular proposal. <p>
### Learners
Announce the outcome. <p>

In practice, a single node may run proposer, acceptor, and learner roles. 
It is common for Paxos to coexist with the service that requires consensus (e.g., distributed storage) on a set of replicated servers, 
with each server taking on all three roles rather than using separate servers dedicated to Paxos. 
For the sake of discussing the protocol, however, we consider these to be independent entities.

## Variables
### Global variable



## Algorithm
