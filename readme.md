# Huntmaster

The universal puzzlehunt tracking system.

## About

Huntmaster is a system for tracking teams progress through a puzzlehunt in real time. It
servers two main purposes:

1.  Gather statistics about the game progress
2.  Provide instant situation awareness on demand

To that end, it needs to perform the following functions

*  Event collection from multiple sources (SMS, web/mobile)
*  Event logging for future processing
*  Event response for feedback
*  Message broadcast to all or a subset of teams
*  Real-time & historical data visualization

### Goals

Huntsmaster has several goals it's trying to achieve:

*  **Reliabilty** during the game - it needs to collect messages no matter the circumstances
*  Versatility - it should support any number of message transports, with SMS and (mobile) web
   out of the box.
*  Flexibility - it should be able to model any type of puzzlehunt with any rules and hint/help
   system, including non-linear games and automatic timed events.
*  Resilience - ability to adapt to changing conditions and human error. In particular, ability
   to reconstruct a full event set from a partial event log and an incoming message log after a
   change of game configuration.
*  Idempotency - it should never log the same event or deliver the same message twice. Subsequent
   attempts to do the same thing should be ignored.
*  Minimal resource consumption in view-only mode - when the game is over, it should need a
   very minimal infrastructure to provide the statistics and visualisation. Ideally it
   should only require a web server capable of serving static files for viewing, making the
   running costs negligeable.

## Installation

TODO

## Deployment

TODO

## Getting started

To get started with Huntmaster, you need to undertand it's puzzlehunt model and architecture.

### Game and progress model

Huntmaster models a puzzlehunt game as a directed graph consisting of four kind of vertices:
*checkpoints*, *puzzles*, *soulutions* and *hints*. Each checkpoint can have multiple puzzles,
each puzzle can have multiple solutions and hints and each solution can lead to multiple
checkpoints. Each vertex in the graph can have a (*code*, *action*) pair, used in the event
messages to look it up. Typically, puzzles and hints are labeled, sometimes checkpoints
and solutions can be labeled as well.

Teams progress through the game is modeled on a game graph induced state machine. Where
the game graph has an edge (e.g. puzzle -> solution) the team graph has a state
(e.g. 'solving *puzzle*' or 'in transit').

A team can be in multiple states at the same time. The state doesn't have
a physical manifestation in the system, it's always reconstructed from the team's event log
when checking for allowed transitions. Allowed transitions are all reachable from the team's
current states (some event messages may not have been sent).

### Event message structure

The event messages are the basic UI of Huntmaster and they all share a common structure.
They contain:

*  The game they are related to
*  The team they were send by
*  The action they describe (optional if the game has a single action)
*  The target of the action
*  A note - any extra text (optional)

Typically, all of the items are identified by a string id to fit into a text message and
roughly form a sentence (e.g. "mygame dreamteam found lastpuzzle wow, this game is awesome!").

### Architecture

In its core, Huntmaster is a message procesing system. Messages come in through a variety
of transports and need to be processed to events, logged and responded to.

#### Message broker

At its core is a [RabbitMQ](https://www.rabbitmq.com) message broker facilitating the flow of
messages through the system. It provides low latency, highly available and reliable message
collection and delivery throught the system decoupling its various parts. The other services
are pluggable and can be written in any language.

#### Event message collection

At the front of the pipeline are the message collection services. Their job is to receive the
messages from the various transports, translate them into a universal format described above,
pass them to incoming queue and acknowledge reception to the client.

#### Event message processing

Next step is the message processor(s). They take messages from the incoming queue validate them
and translate them to events which they write to the master event log and then confirm the message
to the mesage broker and notify listeners about new event. Finally, they create a response message
request pointing to the event, if applicable.

#### Statistics service

For the duration of the game a statistics service is running, which reads the master event log
and team list and can compute different statistics on them for a given timestamp - team distribution,
team positions, etc...

It's very simplest function is to give the event log to the Status display and push incoming events
to the connected clients.

#### Response message processing

The responder service looks up the correct response message, adds required statistics from the
stats service and passes the message into an outgoing queue of the right transport to use. The outgoing
queue is also the place where broadcast and other freeform messages go when created.

#### Message delivery

At the end of the pipeline are the message delivery services, one for each collection service. They
take the messages from the outgoing queue and deliver them.

#### Logging

At every stage of the pipeline the operations and errors are logged through the message broker into
different forms of log storage.

#### Operator console

The operator console has control over all of the services, gets information about messages in all queues
and the state of the system. It can change the game configuration and also manually insert messages into
any queue and replay logs of any stage of the pipeline to the following stage.

There is submit broadcast messages to a set of teams

#### Visualisation & Status display

Visualizations are mostly client-side in-browser applications. They read data from the stats service and
listen for updates. They are also the only part of the system that stays alive when the game is over.


