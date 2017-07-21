# Mirror Worlds Main Server


## Ports

Currently being developed on GNU/Linux systems: Debian 8.6, and Xubuntu
16.04.  It is expected to work on most GNU/Linux systems.


## Package Dependences


### nodeJS

### yui-compressor (optional, build-time only)


## Installing

It builds and installs using GNU make and bash.  In a bash shell, from
the directory with this README file is, run:


```
./configure --prefix=/INSTALLATION/PREFIX && make && make install
```
where ```/INSTALLATION/PREFIX``` the directory to install all the
installed files.  Running ```make``` the first time will download
files from the Internet via ```npm``` and ```wget```.

See:

```
./configure --help
```
for more configuration options.

## Running the server

The Mirror Worlds server is installed as mw_server in the directory
```PREFIX/bin/```.  Run
```mw_server --help```
for help.


#### Example:

From a bash shell:
```
mw_server --doc_root ${HOME}/public_html --http_port=3333 > access.log 2> error.log &
```



# Developer Notes


## File structure

Making nodeJS applications that use many files requires a planning the for
the file directory structure.  No matter how you lay it out there will
always be give and take.  ref: https://gist.github.com/branneman/8048520

1. We decided to keep the installed executables directory (bin/) free of
   any files that the user does not directly execute.  We use the fact
   that NodeJs resolves symbolic links to full file paths in __dirname to
   lib/ where we keep all the code.

2. We decided to keep the test running of the programs not require that
   the package be installed.  The structure of the source files is close
   to that of the installed files.

3. We decided to have the package installation be automated.


## Profiling

- https://nodejs.org/en/docs/guides/simple-profiling/

## Subscription Semantics

The subscription semantics that we define between client and server tends
parallel that of UNIX file semantics with open, write, read, close, and
unlink (remove or delete).  The server acts like the file system part of
the operating system, not knowing what the content (payload) of the file
is.  The client acts like the user of the file system knowing and using
what's in the files (subscriptions).  As in a file system we can open a
file (subscription) creating it or not, and for reading and/or writing.
Dependency structure in the subscriptions are analogous to directories in
a file system.  Circular dependencies can happen, which are analogous to
symlinks in a file system.  You can't create a file before you create the
directory that the file is in, in the same way you can't create a
subscription before you create another subscription that the other
subscription depends on.


Active server push:

  - adding or removing subscriptions are pushed to the client as
    advertisements or cancelations, and

  - subscription payload (file content) changes are pushed to the client;
    i.e. push read.

Analogs of server pushed properties of subscriptions do not exist in file
systems, and are due to the client's ability to act as a accepting
receiver where the client is taking the passive role.  Put another way,
file systems (servers) are usually just passive and do not push data to
the user (client).

Other subscription semantics (not like file system):

  - creating subscriptions without specifying name:  The id of the
    subscription is generated by the server.  The server does not care
    that the name is.

Semantics common to a file system:

  - creating subscriptions with a unique name.  The subscription can
    only be created once, so there is an inherit race condition that must
    be dealt with when all the clients run the same code.  Create is a
    race on file systems too when making a file with multiple users.

Questionable semantics:

  - are reads only available through server push

    - is there use for read pull (not likely)


So the subscription semantics is like a file system with the additions:
files that push reads to the user (client), and the ability to create
files without a file name.

It looks kind of like a networked file system with some additional
semantics, some which are more restrictive making it not like a networked
file system.


## Server Record

- **subscriptions** or *subs* the record for a networked events which are
  kept on the server.  Each subscription has the following properties:

  - *parents* and *children* may not be needed on the server side.

  - **parents** subscriptions that this subscription depends on.
    We can get siblings from the parents children.  TODO: Is this
    a general graph with directed links or just a tree?
  - **children** subscriptions that depend on this subscription

  - **initialized** if not set to true we are still waiting for
    the subcription client creator to *initialize* it.  It can be
    written to before it's *initialized* but no one can read it
    until it's *initialized*.
  - **creatorClient** the client webSocket that caused this
    subscription to be created.  Not necessarly the *owner*
    client, but likely is.  Note: many subscriptions do not
    have an *owner*, they keep state as all clients come and go.
  - **subscribers** a list of clients that read this record
  - **owner** this subscription will be removed if the client owner closes
    it's connection
  - **name** unique name relative to all siblings or null if not set.
    *name* or *className* will be set.
  - **className** if set this maybe one of many subscriptions of this
    class type.  *name* or *className* will be set.
  - **id** a unique per-subscription id gotten from a counter on the
    server.  It's an integer as a string.
  - **shortName** a short description that includes the id in it
  - **description** a longer description that may include HTML
  - **payload** the current state of the subscription that was sent from a
    client.  TODO: payloads may need to be able to be queued, timestamped,
    and/or have other properties.  TODO: queuing and timestamping.

## TODO

- Add the ability to push subscriptions to clients that have not and are
  not staged yet to load the subscription javaScript.

- Rooms and world segregation on a single server.


## Yup

 - Only clients may initiate the creation of a subscription.

 - any client may write to any subscription, i.e. the client-side
   javaScript controls what happens.

   - therefore, the writing client does not see the response
     until it gets the read from the server;

     - it has to be this way in order to keep subscription state
       on the server and all clients consistant

     - We do not open a subscription for writing.  We just write,
       depending on what the client javaScript tells us to do

 - All clients subscribe to all subscriptions by default.  Use
   *unsubscribe()* to opt out and *subscribe()* to opt back in.

   - therefore subscription policy is implemented via client javaScript



## Client Record

The client keeps a record of all advertised subscriptions, whether
subscribed or not.  Any client may write to any subscription.


## Client to Server Subscription Commands

All commands that are sent from the client to the server:

- **onopen** sets up websocket *onmessage*

- **initiate** server responds to client with *advertise* with an
  array of subscriptions that it may write to, or read via a subscribe

- **subscribe** request to read a subscription.  Server responds with
  *read* with the initial subscription state

- **write** writes payload/state to a subscription

- **initialize** initialize a subscription.  A subscription cannot
  be read from until it is initialized.  This is so a client can
  determine if subscription initialization need to be run or not.
  We only let one client initialize a subscription.  The server
  response from *get* determines if the subscription is new and
  needs to be initialized.  *initialize* is not used if this
  subscription is not new

- **get** request to create or connect to a subscription, server responds
  with *get*

- **onclose** websocket closes removes appropriate server resources

- **unsubscribe** unsubscribe from a particular subscription

- **destroy** removes the subscription from the server.  Server
  responds to all clients with *destroy*


## Server to Client Subscription Commands

All commands that are sent from server to clients:

- **initiate** server sends after a client connects, via WebSocket server
  event *connection*.  The client receives a client id, from a server
  counter.  The client responds by requesting files via http GET and also
  sends a *initiate* command to the server.

- **advertise** send new subscriptions, may be array of them.
  TODO: currently all subscriptions are known from the javaScript files
  that are staged to be loaded and the only subscriptions that need to be
  advertised are subscriptions with a className that are created by other
  clients, since they will not be known without advertisement.  We need to
  *advertise* subscriptions to the clients that are not yet staged to load
  the relevant javaScript.

- **get** response to *get* (create or connect request).  Client gets
  subscription id.

- **destroy** send to client to notify them that a subscription no
  longer exists

- **read** subscription read pushed to client


## Startup

The order of event going down the table:

| To Server     | To Client                                              |
| ------------- | ------------------------------------------------------ |
| ws connect    |                                                        |
|               | initiate - gets client id, adds models                 |
| initiate      |                                                        |
|               | advertise - array of subscriptions, save local list    |
|               | makes and subscribes
| get           |





## Client API

There is not server API.  The idea of the "Subscription Model" is that
extending the projects functionality does not require writing server
code.

- **mw** A singleton object

- **mw.createSubscriptionClass()**

- **mw.getSubscription()**

- **mw_addActor()**

