[//]: # ( dpipes_version: 1.0.0 )

---

# About *dpipes*
Distributed Pipes (or *dpipes*) is a framework for writing applications that
communicate through something resembling a Unix pipe, but is actually a
distributed communication network for applications. Piping applications
together results in a network of applications communicating together using
a composeable interface.

By default, any application that can be forked and use stdin and stdout file
handles can be used in a *dpipe*. All that is necessary is a host which runs a
**dpipesd** daemon (and a network to connect distributed nodes).

In addition, the communication method can include both raw byte streams, JSON
object streams, and YAML object streams. These formats allow applications to
share more complex, structured data than a typical raw byte stream provides.

# What is a Distributed Pipe?
A Distributed Pipe, or *dpipe*, is a structured set of applications linked
together using a communication protocol. Metadata is passed to the applications,
which includes the same sort of data you would expect from a Unix process,
such as:
 - Environment variables
 - Optional command arguments
 - User account information (user, groups)
 - Local/Remote host information

## Comparison to other technology:

### - Service Mesh provider
*dpipes* is similar to a service mesh in that it provides a *sidecar* for each
application, and that *sidecar* communicates with other *sidecars* that are
connected to other applications.

The **dpipesd** process contains both the control plane and data plane. The
purpose of *dpipes* are to be self-contained distributed applications which can
interact with any other application on a system, so they are largely
self-contained.

*dpipes* do not attempt to provide enterprise-grade features to distributed
applications, such as traffic management, security policy enforcement, load
balancing, rate limits, circuit breakers, and so on. Such features are
admirable, but if you need all those features, you should get a real service
mesh.

### - API Gateway
*dpipes* is not an API gateway. *dpipes* doesn't even expose any API to
applications. APIs are great if you need a standardized interface for
applications to work together, but this is the opposite of what pipes are for.

Pipes (as in Unix pipes) are for simply passing data between *arbitrary*
applications without worrying about how to integrate them. *dpipes* does the same
thing, but in a distributed way, and adds one or two features that modern
applications may want.

# How *dpipes* Works

## Overview
To create a *dpipe*, you should ideally have more than one node (though
there's no reason you can't use just one node). Each node should have a
**dpipesd** application installed. This process can be run as a daemon or as
a foreground process. The process receives network connections and executes
applications as requested, passing data into the program's *stdin* and receiving
data from the program's *stdout*.

Data is passed between applications on their respective nodes. The client does
not receive anything but the output of the last application in the pipe.

*dpipes* is mainly only useful for parallel processing of data. Take the following example:

    $ dpipes 'cat /nfs/mydata/input.txt | sort | xargs ls -la'

There is basically no benefit to running this command with *dpipes*. The output
of each command is completed and then passed on to the next command on another
node. Performing this command on a single node without *dpipes* would probably
be much faster.

Instead, let's try some simple parallelization of the same task:

    $ for i in `seq 1 5` ; do dpipes "cat /nfs/mydata/input.$i.txt | sort | xargs ls -la" & ; done

Here we background multiple *dpipes* applications that process multiple data
files. Normally one could use a program like **dsh** to run commands in parallel
across a range of hosts. But in this case, each individual step is run on
a different node at the same time and processes data as it comes in.


## Configuration
All configuration to *dpipesd* can be specified in the config files *.dpipesdrc*
and */etc/dpipesd.conf*.

By default, *dpipes* has no idea where other *dpipes* nodes are, so you must
provide a configuration file with a list of nodes. Alternately you can
configure a service discovery agent to query for nodes.

Configuration can also be passed to remote *dpipes* nodes at execution time.
However, those nodes may elect to ignore any configuration options passed.

## Connection Negotiation
In order to support reliable execution of applications, each *dpipes* connection
begins with a *negotiation* phase. This phase validates metadata
passed between both nodes before any applications are opened.

This enables both nodes to agree on arbitrary information, like what version
of an application must be installed, *dpipesd* configuration options,
authorization requirements, what applications can be run, what data can be
passed between applications, etc.

This two-way validation of the connection enables applications to operate in
a more idempotent way than traditional Unix applications provide. If for some
reason a *dpipes* environment is not what was expected, the connection can
simply die, rather than run an application on a non-idempotent environment.

## Executing applications
By default, *dpipesd* will run as a daemon on a specific port and accept any
connection from another *dpipesd* agent. It will then run any application
requested, in whatever configuration is specified.

Options to *dpipesd*:
 - **--foreground** : *dpipesd* will not background itself.
 - **--exit** : *dpipesd* will exit immediately after running an application once.

If the argument '--' is given when running *dpipesd*, anything following it will be
taken as the only application available to run.

## Communication Protocol
*dpipes* communicates between its nodes using a multiplexed transport protocol.
A simple routing protocol identifies how to connect applications in a *dpipe*.

Optionally, *dpipes* may provide a REST API to control and query the state of
*dpipes* and its pipes between applications. This API may pass through any
information regarding its state, including the last result of queries to
a service discovery agent, authentication+authorization, security, etc.

## Application Data input/output
Applications will pass data to and from **dpipesd** using one of several
standard methods:

 - **byte streams**

   Default mode. Passes input and output as byte streams, the same way as normal Unix pipes.
   Metadata will be passed to applications using environment variables.

 - **data objects**

   Arguments or configuration to **dpipesd** enable this mode. Instead of byte
   streams, data is sent and received as serialized JSON or YAML objects.
   Metadata will be embedded in the data objects at the root level in a
   dictionary element named "**__dpipes**" (this can be overridden in
   configuration).

## Metadata

If JSON or YAML data exchange is enabled, metadata will be inserted into the
data format. The following keys will be passed at the root object level:
 - **dp.addr**
 - **dp.port**
 - **dp.service**

The following keys *may* be passed at the root object level:
 - **dp.credential**
 - **dp.environ**
 - **dp.argv**

Note: the prefix **__dp.** may change according to the **dpipesd** configuration.

## Service Discovery
By default, *dpipes* operates by connecting to a static list of hosts and ports.
This can be quite limiting on most modern networks, so *dpipes* also supports
service discovery.

### Advertising *dpipesd*
If **dpipesd** is configured with a service discovery agent, it will register
itself when it starts as a service called "*dpipesd*". **dpipesd** will also
query the service discovery agent or service registry for other nodes to connect
to for running applications.

### Advertising *dpipesd applications*
In addition to advertising itself, **dpipesd** can advertise specific
applications that it can run. In this way you can use the advanced features of
a service discovery framework (such as healthchecks and alerts) to keep track
of which nodes provide which applications.

The application-specific service name will be the prefix "*dp.*" plus the
application name, like "*dp.sort*".

### Using discovered services
If application-specific services are discovered, those services will be used
to run particular applications. For example, if service discovery shows that
there is a service called "*dp.sort*" running on 2 nodes, then every time
**dpipesd** tries to run the application "*sort*", it will use the "*dp.sort*"
service nodes.

## Access
No access control is included by default. Network access to **dpipesd** should
be restricted to trusted hosts only.

Future releases of **dpipesd** will include authentication and authorization
schemes (possibly using OAuth). When a user has been successfully authenticated,
this should be passed through metadata as the *__dp.credential* key. **dpipesd**
will pass authentication on to a separate agent to validate a credential or
request an authorization.

## Security
By default, *dpipes* does not use a secured transport protocol. The transport
protocol can be replaced with a secure one - however, metadata about the
state of the connection will need to be passed along from the transport layer
to the application.

## Glossary
 - **dpipe**: A distributed pipe.
 - **dpipes**: A shorthand for 'Distributed Pipes'. A command-line tool to interact with **dpipesd**.
 - **dpipesd**: The Distributed Pipes daemon.

