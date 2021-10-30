# Routing Lab

The objective of this assignment is to give you experience with how routing
protocols work.  To accomplish this, you will implement the a basic distance
vector (DV) routing protocol and populate a router's forwarding tables using
the routes learned.

 - [Getting Started](#getting-started)
   - [Update Cougarnet](#update-cougarnet)
   - [Install Dependencies](#install-dependencies)
   - [Resources Provided](#resources-provided)
   - [Starter Commands](#starter-commands)
   - [Scenario Descriptions](#scenario-descriptions)
   - [Packets Issued](#packets-issued)
 - [Instructions](#instructions)
   - [Specification](#specification)
   - [Scaffold Code](#scaffold-code)
   - [Testing](#testing)
   - [Helps](#helps)
 - [Submission](#submission)

# Getting Started

## Update Cougarnet

Make sure you have the most up-to-date version of Cougarnet installed by
running the following in your `cougarnet` directory:

```
$ git pull
$ python3 setup.py build
$ sudo python3 setup.py install
```

Remember that you can always get the most up-to-date documentation for
Cougarnet [here](https://github.com/cdeccio/cougarnet/blob/main/README.md).


## Install Dependencies

Install [pyroute2](https://pyroute2.org/) by running the following:

```
$ sudo apt install python3-pyroute2
```


## Resources Provided

The files given to you for this lab are the following:
 - `dvrouter.py` - a file containing a stub implementation of a host (and
   router).  This is where you will do your work!

   Note that in this lab, your focus is not forwarding but rather routing.
   Thus, virtual hosts will not use the functionality in your `Host` class for
   forwarding but will instead use the native network stack of the virtual
   host.

   Nonetheless, with just a bit of effort, you _could_ drop in your `host.py`
   from the
   [Network-Layer Lab](https://github.com/cdeccio/byu-cs460-f2021/blob/master/lab-network-layer/),
   and have `DVRouter` inherit from that instead of from `BaseFrameHandler`.
   Then, if you
   [configure the host](https://github.com/cdeccio/cougarnet/blob/main/README.md#network-configuration-file)
   to use `native_apps=no`, you are using your own
   implementation for sending Ethernet frames, ARP, IP forwarding, and
   routing!
 - `scenario1.cfg`, `scenario2.cfg`, and `scenario3.cfg` -
   [network configuration files](https://github.com/cdeccio/cougarnet/blob/main/README.md#network-configuration-file)
   describing three topologies for testing your routing implementation.
 - `scenario1.py`, `scenario2.py`, and `scenario3.py` -
   scripts that run various tests in conjunction with the network configuration
   files.
 - `scenario1a.cfg` - a variant of `scenario1.cfg`, in which no script is
   called, but you can simply look at the toplogy and the forwarding tables in
   their initial state.


## Starter Commands

Take a look at the contents of `scenario1a.cfg`.  Then run the following to
start it up:

```
$ cougarnet --display --disable-ipv6 scenario1a.cfg
```

The `--disable-ipv6` option is used on the command line here and throughout the
remainder of the lab to prevent your forwarding tables from being populated
with IPv6 prefixes, so you can focus exlusively on routing with a single
protocol.

You will notice that, by default, terminal windows appear for every host.  You
can disable this with the `--terminal=none` option or by specifying
`terminal=false` in the configuration file.

Run the following commands on host `r2` to show its network interface
configuration and forwarding table:

```bash
r2$ ip addr 2> /dev/null
r2$ ip route
```

(The `2> /dev/null` simply redirects standard error, which is noisy due to
unknown causes related to working in a private network namespace :))

You should only see two entries in the forwarding table, each one corresponding
to an interface on the router.  These table entries are created automatically
by the system when the interfaces are configured with their respective IP
addresses and prefix lengths.  So basically at this point, `r2` knows that to
get to the subnet corresponding to its `r2-r1` interface, it sends a packet
out `r2-r1`, and to get to the subnet corresponding to its `r2-r3` interface,
it sends a packet out `r2-r3`.  The problem is that for anything else, it
doesn't know where to go!

Rather than manually creating static forwarding entries, like you did with the
previous
[homework](https://github.com/cdeccio/byu-cs460-f2021/blob/master/hw-network-layer/)
and
[lab](https://github.com/cdeccio/byu-cs460-f2021/blob/master/lab-network-layer/),
in this lab, you will update the forwarding tables dynamically using a distance
vector (DV) protocol.  Indeed, you will have a working router that will not
only be capable of _forwarding_ packets but also _routing_!

Note that the _forwarding_ function is handled using the native Linux network
stack; you do not have to implement its functionality!  Your focus is on
_routing_.  Thus, your DV protocol will be updating the actual forwarding table
on each router / virtual host.

Now look at the contents of `scenario1.cfg`, and run the following to start it
up:

```
$ cougarnet --disable-ipv6 scenario1.cfg
```

On the terminal from which you started Cougarnet, you will see log messages
indicating that ICMP packets are being sent and links being dropped as
[specified](#packets-issued).  On the terminal corresponding to each router,
you will simply see a notice every second that a DV table is being sent.  This
is simply a notice that the `send_dv()` method is being called. You will be
fleshing out that method, and others, such that the ICMP packets sent are seen
by multiple routers, including the destination.


## Scenario Descriptions

Your working router should work in the following three scenarios, described in
the files `scenario1.cfg`, `scenario2.cfg`, and `scenario3.cfg`, respectively.

### Scenario 1


In scenario, routers `r1` through `r5` are connected in a line.

```
  r1 --- r2 --- r3 --- r4 --- r5
```


### Scenario 2

In scenario, routers `r1` through `r5` are connected in a ring.

```
    r1 --- r2
    |        \
    |         \
    |          r3
    |         /
    |        /
    r5 --- r4
```

After some time, the link between `r1` and `r5` is dropped:

```
    r1 --- r2
             \
    |         \
    X          r3
    |         /
             /
    r5 --- r4
```

### Scenario 3

In scenario, routers `r1` through `r15` are connected in a more complex
topology.

```
       --- r7 ----r8
      /    |      |
     /     |      |
   r9      |      |
     \     |      |
      \    |      |
       --- r6     r2 --- r14 --- r15
            \    /  \    /
             \  /    \  /
      r11 --- r1      r3
              |       |
              |       |
              r4 ---- r5 --- r13
              |       |
              |       |
             r10     r12
```

After some time, the link between `r2` and `r8` is dropped:

```
       --- r7 ----r8
      /    |
     /     |      |
   r9      |      X
     \     |      |
      \    |
       --- r6     r2 --- r14 --- r15
            \    /  \    /
             \  /    \  /
      r11 --- r1      r3
              |       |
              |       |
              r4 ---- r5 --- r13
              |       |
              |       |
             r10     r12
```


## Packets Issued

ICMP packets are sent using `ping` in each scenario.  In each case, the point
of this call to `ping` is to check that:

 - there is a path (i.e., forwarding entries in routers along the way) from the
   source (i.e., the host on which `ping` is executed) and the destination
   (i.e., the destination argument specified on the command line);
 - there is a return path from the (original) destination to the (original)
   source;
 - the path from source to destination and back again is the shortest path.

In each scenario the first set of ICMP packets will be issued after routes will
have propagated, distance vectors converged, and forwarding tables have the
proper entries for shortest-path forwarding.  For scenarios 2 and 3 a second
set of ICMP packets will be issued after a given link has been dropped and the
distance vectors and forwarding table entries have been updated properly.


### Scenario 1

 - 4 seconds: ICMP packet sent from `r1` to `r5` and back again
 - 5 seconds: ICMP packet sent from `r2` to `r4` and back again


### Scenario 2

 - 4 seconds: ICMP packet sent from `r2` to `r5` and back again
 - 5 seconds: ICMP packet sent from `r2` to `r4` and back again
 - 6 seconds: Link dropped between `r1` and `r5`
 - 12 seconds: ICMP packet sent from `r2` to `r5` and back again
 - 13 seconds: ICMP packet sent from `r2` to `r4` and back again


### Scenario 3

 - 4 seconds: ICMP packet sent from `r9` to `r10` and back again
 - 5 seconds: ICMP packet sent from `r9` to `r11` and back again
 - 6 seconds: ICMP packet sent from `r9` to `r12` and back again
 - 7 seconds: ICMP packet sent from `r9` to `r13` and back again
 - 8 seconds: ICMP packet sent from `r9` to `r14` and back again
 - 9 seconds: ICMP packet sent from `r7` to `r15` and back again
 - 10 seconds: Link dropped between `r2` and `r8`
 - 18 seconds: ICMP packet sent from `r7` to `r15` and back again


# Instructions

Read Section 5.2.2 ("The Distance-Vector (DV) Routing Algorithm") in the book.
Then implement a DV router in `dvrouter.py` with the following functionality.


## Specification

 - A router starts out knowing only about the IP prefixes with which it is
   directly connected.  In a general sense, this includes the subnets to which
   it is directly connected.  However, for this lab, the prefixes that will be
   passed around will be /32's.  That is, we will treat IP _addresses_ as the
   IP _prefixes_.  This will simplify things, so you can focus on the routing.

   For example, in scenario 1, `r2`'s initial DV (i.e., before it receives any
   DVs from neighbors) from the [example given previously](#starter-commands)
   will look something like this:

   - Prefix: 10.0.0.2; Distance: 0
   - Prefix: 10.0.0.5; Distance: 0

   The IP address for each interface can be found with the `int_to_info`
   attribute.

 - A router sends its own DV to every one of its neighbors in a UDP datagram.
   You do not have to set up the socket for sending and receiving UDP datagrams
   containing DV messages.  This has been done for you.  When you have the payload
   ready, you can simply call `self._send_msg()`, which takes the following as
   arguments:

   - `msg` - a `bytes` instance containing the DV message.  This will be sent
     as a UDP payload.  You do not need to create any headers for this; they
     will be created for you as part of normal socket functionality.
   - `dst` - a `str` instance containing the IP address to which the message
     should be sent.

   Sending a DV message to every neighbor means sending a DV message out every
   interface.  Since this is a discovery process, you don't actually know the
   IP address of your neighbor, so you cannot use that for your destination
   address, `dst`.  Instead, for a given interface, you will send to the IP
   address that is the _broadcast_ address corresponding to the subnet on the
   interface.  The broadcast address for a given subnet is simply the subnet
   prefix with all of the host bits set--or, the very last address in the
   subnet.  For example, the broadcast address for 10.1.2.0/24 is 10.1.2.255.
   And the broadcast address for 10.1.2.20/30 is 10.1.2.23.
   
   However, in the lab, you won't have to calculate this yourself. The
   broadcast IP address for the subnet corresponding to given interface can be
   found with the `int_to_info` attribute, which is documented
   [here](https://github.com/cdeccio/cougarnet/blob/main/README.md#baseframehandler).
   Note that this subnet-specific broadcast address is used instead of a global
   broadcast (255.255.255.255) for (at least) two reasons:

   - Since our packet only needs to reach the other side of the link, to a
     neighbor that we know has an IP address on the same subnet, there is no
     reason to use a more general broadcast address with which the packet could
     potentially get forwarded beyond the subnet/link/local area network (LAN).
     Using this address will guarantee that.
   - Sending a packet with 255.255.255.255 requires root privileges.  We know
     how to do this, but the principle of least privilege indicates that we
     should only elevate when necessary.

 - Each DV message has the following properties:

   - the source IP address corresponding to the interface out from which the
     packet is being sent. This can be found with the `int_to_info` attribute,
     which is documented
     [here](https://github.com/cdeccio/cougarnet/blob/main/README.md#baseframehandler).
   - the name of the router sending the message.  This is can be found with the
     `hostname` attribute, which is initialized for you in `__init__()`.
   - the distance vector of the sending router.

   These properties can be put together however you want, but it is recommended
   that you create a `dict` object that you can convert to a `str` (and
   eventually to `bytes`) using JSON.  For example:

   ```python
   obj = { 'ip': '10.0.0.1', 'name': 'r1', 'dv': { ... } }
   obj_str = json.dumps(obj)
   obj_bytes = obj_str.encode('utf-8')
   ```

 - When a router receives a DV message from one of its neighbors, it does the following:

   - Converts the message to a `str` (from `bytes`) and decodes the JSON using
     something like the following:

     ```python
     obj_str = msg.decode('utf-8')
     obj = json.loads(obj_str)
     ```

   - Extracts the name, IP address, and DV of the neighboring node.
   - Discards the packet if it is one of its own packets (i.e., the
     `self.hostname` matches the name of the router in the DV message).
     Because the destination IP address is the (subnet) broadcast address, the
     sending router might actually receive its own message.
   - Maps the neighbor's name to its IP address.  This will make things a lot
     easier when creating forwarding table entries.
   - Saves the neighbor's DV, replacing any previous version, so it can be used
     later for running Bellman-Ford algorithm.
   - Uses its neighbors DVs to re-create its own DV and forwarding table.

 - A router creates its own DV using the Bellman-Ford algorithm.  By iterating
   through the DV of every neighbor, a router learns the shortest distance to
   every prefix known by its collective neighbors.  _Eventually_ (after
   several iterations), it will converge, such that its DV contains the
   shortest distances to each IP prefix.  Bellman-Ford requires comparing the
   the sum of 1) the cost of the link for a given neighbor and 2) that
   neighbor's distance (according to its DV), for all neighbors.  The distance
   from each neighbor to a destination is contained in the neighbors' DVs.  The
   cost to each neighbor, for the purposes of this lab, is simply 1.

   To a newly-created DV, a router adds entries for each local prefix (IP
   address), just as it did when the DV was initially created.  Just as before,
   those prefixes will always have a distance of of zero.

 - A router distributes its DV to its neighbors in two circumstances:

   - When a router's own newly-created DV (which creation is prompted by a DV
     message from a neighbor) is _different_ from its previous
     version, then a new DV message is distributed immediately.
   - A DV message is distributed to neighbors every one second, as a
     keep-alive, to let neighbors know that the link is still up.

 - A router updates its forwarding table whenever its own DV has changed after
   its re-creation.  For every prefix in its DV, the next hop is the IP address
   of the neighbor resulting from an incoming neighbor DV message).  A
   forwarding table entry for a prefix consists of the IP address of the
   neighbor having the lowest distance to that prefix as the next-hop IP
   address.  The `add_entry()` method of the forwarding table instance will
   allow you to pass `None` as the interface, and it can be inferred from the
   next hop (this is made possible because of the local forwarding table
   entries, which are created by default, as described
   [previously](#starter-commands)).

   There are two primary ways to update the forwarding table:

   - Call the `flush()` method on the forwarding table instance to clear out
     all existing entries, and then build the table from scratch; or
   - Call the `get_all_entries()` method on the forwarding table instance to
     get its current state, and then add/remove using the `add_entry()` and
     `remove_entry()` methods, respectively, to update the table.

 - A router keeps track of the last time that it received a DV message from
   every neighbor.  After three seconds have passed since receiving a DV
   message from a neighbor, the router discards that neighbor's DV, such that
   it and its prefixes are no longer considered in the computation of the
   router's own DV table (and forwarding table).


## Scaffold Code

In the file `dvrouter.py`, flesh out following the skeleton methods to help you
implement the above specification.

 - `handle_dv_message()`.  This method is called by `_handle_msg()`, which
   receives a UDP message from a UDP socket.  It takes the following as an
   argument:

   - `msg`: a DV message (`bytes`), consisting of JSON representing the DV of
     the neighbor that sent it.

   The method should do everything that is associated with receiving a DV
   message.

 - `update_dv()`.  This method takes no arguments.  The method should be called
   by `handle_dv_message()` and should implement the Bellman-Ford algorithm
   yielding a newly-created DV for the router.  In the case that the DV
   changes, the IP addresses of the neighbors with shortest distances to IP
   prefixes are used to create new forwarding table entries.

 - `send_dv()`.  This method takes no arguments.  The method should be called
   by `update_dv()` when a router's newly-created DV has changed.  It is also
   called by `send_dv_next()` (implemented for you) every 1 second, to
   implement a keep-alive.

 - `handle_down_link()`.  This method takes a single argument:

   - `neighbor`: the hostname (`str`) of the neighbor corresponding to the link
     that is no longer up.

   The method should be called whenever a router has detected a down link
   (through lack of keep-alive DV messages).  Basically this method should
   simply discard the DV of the neighbor whose link is down and then call
   `update_dv()`.


## Testing

Test your implementation against scenario 1:

```
$ cougarnet --disable-ipv6 scenario1.cfg
```

Determine the appropriate output--that is, which hosts should see the scheduled
ICMP packets on their way and back--and make sure that the cougarnet output
matches appropriately.

When it is working properly, test also with the `--terminal=none` option:

```
$ cougarnet --disable-ipv6 --terminal=none scenario1.cfg
```

Then proceed to test scenarios 2 and 3.

```
$ cougarnet --disable-ipv6 scenario2.cfg
$ cougarnet --disable-ipv6 scenario3.cfg
```

When all are working properly, test also with the `--terminal=none` option:

```
$ cougarnet --disable-ipv6 --terminal=none scenario1.cfg
$ cougarnet --disable-ipv6 --terminal=none scenario2.cfg
$ cougarnet --disable-ipv6 --terminal=none scenario3.cfg
```


## Helps

### Useful Methods

 - The `ForwardingTableNative.get_all_entries()` will return all entries
   currently in the forwarding table.  You can use this along the way to
   inspect the current state of your table.  By default the prefix and next
   hop are IP addresses.  However, if you pass `resolve=False` into
   `get_all_entries()`, it replaces those addresses with hostnames, which might
   help your troubleshooting.
 - Similarly, the `DVRouter.resolve_dv()` method takes as an argument a DV
   (`dict`) and replaces the IP address (key) with the corresponding hostname.

### Other Helps
 - Get the routing code working first, then focus on recovery after a dropped
   link is detected.
 - In `update_dv()`, do _not_ try to optimize by _updating_ your DV.  Re-create
   your DV every time, using the 1) initial (local) prefixes and 2) the
   prefixes learned from neighbors' DVs.
 - Your down link detection might be implemented in several ways.  One way is
   to use the scheduler documented
   [here](https://github.com/cdeccio/cougarnet/blob/main/README.md#networkeventloop),
   creating an event for a given neighbor that calls `handle_down_link()` after
   the specified time has passed and cancelling/re-creating the event every
   time a message was received from that neighbor.
 - Print to standard out for debugging purposes.  For a script running in a
   virtual host (i.e., with the `prog` option), all output will go to the
   terminal associated with that host, assuming `terminal=false` is not used in
   the configuration file and `--terminal=none` is not used on the command
   line.  See
   [the documentation](https://github.com/cdeccio/cougarnet/blob/main/README.md#additional-options).
   for more.
 - You can modify `scenario1.py`, `scenario2.py`, `scenario3.py`, and the
   corresponding configuration files all you want for testing and for
   experimentation.  If this helps you, please do it!  Just note that your
   submission will be graded using only your `dvrouter.py`. The other files
   used will be the stock files
   [you were provided](#resources-provided).
 - Save your work often.  You are welcome (and encouraged) to use a version
   control repository, such as GitHub.  However, please ensure that it is a
   private repository!


# Submission

Upload your functional `dvrouter.py` to the assignment page on LearningSuite.
