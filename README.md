# About dpipes
Distributed Pipes (or *dpipes*) is a framework for writing applications that
communicate through something resembling a Unix pipe, but is actually a
distributed communication network. Piping applications together results in a
network of applications communicating together using a composeable interface.

You can think of DPipes as a simplified service mesh for Unix-like applications.

By default, any application that can be forked and use stdin and stdout file
handles can be used in a dpipe. All that is necessary is a host which runs a
**dpipesd** daemon.

In addition, the communication method can include both raw byte streams, JSON
object streams, and YAML object streams. These formats allow applications to
share more complex, structured data than a typical raw byte stream provides.

## What is a Distributed Pipe?
A Distributed Pipe, or DPipe, is a structured set of applications to link
together using a communication protocol. Metadata is passed to the applications,
which includes the same sort of data you would expect from a Unix process,
such as:
 - Environment variables
 - Optional command arguments
 - User account information (user, groups)
 - Local/Remote host information

## Communication Protocol
DPipes communicates between its nodes using a multiplexed transport protocol.
A simple routing protocol identifies how to connect applications in a DPipe.

Optionally, DPipes may provide a REST API to control and query the state of
DPipes and its pipes between applications. This API may pass through any
information regarding its state, including the last result of queries to
a service discovery agent, authentication+authorization, security, etc.

## Data Formatting
By default, applications will simply pass byte streams. Arguments to **dpipes**
(or an optional configuration file) will enable JSON or YAML data exchange.

If JSON or YAML data exchange is enabled, metadata will be inserted into the
data format. The following keys will be passed at the root object level:
 - **__dp.addr**
 - **__dp.port**
 - **__dp.service**

The following keys *may* be passed at the root object level:
 - **__dp.credential**
 - **__dp.environ**
 - **__dp.argv**

## Service Discovery
By default, DPipes operates by connecting to a static list of hosts and ports.
This can be quite limiting on most modern networks, so DPipes also supports
service discovery agents.

DPipes will use any service discovery agent that can be queried for a service
called *dpipesd* and return a list of hosts and ports. Dpipes will act as a 
service that provides applications to route connections to.

However, an alternate model would include routing applications to specific
services discovered with service discovery. The service names should include
a *dp.* prefix, and anything following that prefix would be the application
name. In this way, service discovery features to monitor service health and
advertise particular applications can be used to further improve interaction
between DPipes applications.

## Access
Due to the non-local nature of these pipes, network access to **dpipesd** should
be restricted to trusted hosts only.

Future releases of **dpipesd** will include authentication and authorization
schemes (probably using OAuth). When a user has been successfully authenticated,
this should be passed through metadata as the *__dp.credential* key.

## Security
By default, DPipes does not use a secured transport protocol. The transport
protocol can be replaced with a secure one - however, metadata about the
state of the connection will need to be passed along through the transport.

## Glossary
 - **dpipe**: A distributed pipe.
 - **dpipes**: A shorthand for 'Distributed Pipes'. A command-line tool to interact with **dpipesd**.
 - **dpipesd**: The Distributed Pipes daemon.

