This was written by tcatm in 2016. Since then it was taken offline. The plan 
still has relevance so I am re-hosting it here. The next commits will provide 
an update to the current state.

# Layer 3 Master Plan

Transitioning towards a Layer 3 mesh will alleviate growing pains of current 
mesh networks. By creating a virtual next-hop router run on every mesh node we 
will gain detailed control over clients and their roaming events while greatly 
reducing management traffic and even removing the need for periodic global 
managemenet broadcasts. We’ll also get to tackle the major tasks,

1. Distribution of routes and 
1. Client roaming,

Directly as opposed to the interception and manipulation of live traffic.

For now, we’ll assume an IPv6 mesh. All methods described here could be easily 
adopted for IPv4 if needed.

# The Virtual Router
A basic building block of this master plan has been in Gluon since the very 
beginning: gluon-next-node. This rather simple package allows all nodes within 
a mesh to respond to a common MAC, IPv4 and IPv6 address from the perspective 
of connected clients. As of now, this feature is often used to access a node’s 
status page and that’s about the only use case in the wild. However, we can 
also use these common addresses as the default gateway for all clients; and 
this is the real game changer: If we do that, as far as the clients are 
concerned, there’s only one gateway responsible for routing their traffic. This 
gateway is a virtual one, present on all nodes.

Now this is pure awesome. Each node gets full routing control over the clients 
traffic. As of today, the nodes wouldn’t know what to do with that traffic. 
Here’s where babel (or any suitable Layer 3 mesh protocol, really) gets baked 
into the firmware. It’s main feature is distributing routes across nodes and 
ensuring reachability of these routes. We’ll use it to do just that. And the 
route will be really simple. In fact, we’ll start by distributing a default 
route towards the internet at one of our nodes (presumably that node has 
internet access) and babel will do its magic to tell all nodes about that 
route. So there we have it, all nodes now know the default route and any node 
can route traffic for any client. By virtue of the next-node feature any client 
can use any node to route its traffic towards the internet. It could even 
switch nodes between each packet and it would just work without any hiccup.

Thus, traffic originating from clients destined to reach the internet, or some 
other remote network, is a solved problem. We can do this today. Please make 
sure, you completely get this before continuing. It will become a fundamental 
building block of our future network architecture.

# Getting Packets to Clients
For networks to be somewhat useful, it is often desired for clients to receive 
responses. Let’s suppose a client tries to access a webserver. The client will 
send its request towards the server and the virtual router inside the mesh will 
work its magic so the request reaches that webserver. The webserver sends its 
response and, by summoning routing magic already deployed on the internet 
today, this response will make it back to the internet gateway node of our 
mesh. Now, we’ll run into a hard problem but at least we can take hold of that 
little packet our client somewhere within our mesh network is eagerly waiting 
to receive.

The problem is this: We know that the response packet is destined for a client 
on our network but we really have no idea which node this client is connected 
to nor do we have any means to establish even a vague direction in which to 
send this packet. We do know the clients IP, though. It’s conveniently written 
on the packets header.

Let’s introduce a new concept: A routing destination for “undiscovered 
clients”. This route will receive any packet destined for a client that we 
don’t yet know. How can we do this? It’s really simple: We, by convention, know 
that all clients will use an IP from a certain prefix (say 2001:db8::/64) and 
that for clients we already have discovered we will also know a more specific 
route (a host route actually). We are going to push packets for this catchall 
route towards a little daemon. Let’s call that daemon l3roamd, the Layer 3 
Roaming Daemon.

Any packet destined for a client that we have yet to locate will end up in 
l3roamd and this is where we’ll introduce yet more magic.

# Locating a Client

In l3roamd we will actually look at the destination IP address in detail. What 
happens next, is conceptually really simple as well. We’re going to flood a 
request through our mesh. Other l3roamd’s on other nodes will receive it and 
we’ll ask them to do a IPv6 Neighbor Discovery for the IP address we’re looking 
for in their local client networks. So whenever a packet for an unknown client 
arrives at any node, all nodes will look for a matching client. Oh, and we 
don’t even expect a response. Instead, we expect that l3roamd that actually 
found the client to insert a host route into the Layer 3 mesh that gets 
distributed to all nodes, including that tiny node where the packet eagerly 
waiting to be transmitted to the client. That node’s l3roamd will notice the 
new route to the client (by watching the kernel’s routing table) and as soon as 
it appears will resend the packet. This time, it will be routed to the client 
and not be caught by our catchall route as before. In fact, any further packet 
to that client can now be routed directly.

# Roaming Clients

Did you notice what just happened? We’ve actively discovered a client’s IP and 
in the process the node where this client is connected has become aware of its 
MAC address, too. It also has a vague idea of which nodes may be interested in 
sending packets to that client. That’s a lot of information we can use it. 
Let’s start a database and write down

* our node identifier,
* our IP address,
* the client’s MAC address,
* IP addresses used by that client and
* timestamps for when each IP was last discovered.

Oh, and for good measure, flood this database entry through the mesh. Any node 
will thus have the same knowledge and could, for any client identified by its 
MAC address, lookup all corresponding IP addresses used by that client.

So now our client decides to switch to another node. What happens? Well, the 
wireless layer will detect the client’s MAC address and we’re going to hook 
l3roamd into that as well. From the point of view of that other node, a new 
client has appeared. Just because l3roamd will be built to be really curious 
about clients, it will try a lookup of that clients MAC address in the 
distributed client database. It’s going to find the entry created by the node 
previously serving the client and it will do two things:

* Insert a host route for each IP address known to be used by that client into 
  the mesh and
* contact the previous node to let it know it no longer serves this specific 
  client.

If all goes well, the previous node is going to drop its routes and the new 
routes will take over. Updating the route will be cheap if both nodes are close 
to each other. Only a few neighbours will need to update their routes while 
nodes further away may not need to recalculate all metrics just yet. In case 
the previous node has disappeared from the mesh, all is well, too. The new 
node’s routes will take over.

# Getting l3roamd ready

So this is seamless client roaming. What are the problems we need to solve for 
this to work?

l3roamd needs to be written, of course. The biggest pain point being its 
distributed database. We’d like it to contain as little data and require as 
little management traffic as reasonably possible. It is probably fine to forget 
a client and its routes in case of disappearing nodes. We can rediscover these 
routes. It may suffice for only a single node to hold information about a 
specific client. Other nodes would then query the network for the node holding 
that information. This will cause hiccups when roaming from a disappearing node 
but that may be acceptable. Further ideas are welcome!


# Multiple Client Prefixes

We’d like to distribute multiple prefixes to clients and source routes on nodes. 
For this, we’ll need a small daemon managing these prefixes. The idea is this:

* Internet gateway nodes will distribute a prefix (say a /64).
* Nodes will pick up all prefixes.
* Nodes will choose which prefixes to announce to clients.

This will enable anyone to share their internet connection, even native 
prefixes, to anyone within the mesh. We’d like for nodes to only distribute a 
subset of the prefixes they deem suitable for a client to use. A prefix may be 
suitable if

* the corresponding route is short,
* there are a enough gateways for this prefix available,
* a yet to be defined metric is within a reasonably range.

This metric may cover properties like available bandwidth and even latency.

Please note that many gateway nodes can distribute the same prefix and nodes 
distributing prefixes are not necessarily the nodes providing routes for these 
prefixes.

# Source routing

You may know a route as just consisting of a destination prefix and a gateway. 
Routes can also have a source prefix. This acts like a filter. A route with a 
source prefix can only be used if the IP address of the sender is within the 
source prefix. For our Layer 3 Master Plan we’d like to employ source routing 
throughout all components of the mesh. This will allow coexistence of multiple 
prefixes within the same mesh. In the most extreme case, this will allow 
connecting different meshes (e.g. Freifunk Communities) without doing any harm.

# DNS

This is a aspect of the Layer 3 Master Plan we haven’t put any thought into 
yet. The general idea is probably, that nodes are going to provide a DNS 
resolver on the next node address. There’s no further concept yet and if you 
feel like digging into this, by all means go ahead. It’s certainly an 
interesting topic and unrelated to l3roamd or the client prefix daemon. It’s 
not even related to babel.

Could we do anycast DNS in a Layer 3 mesh?

# Multicast Routing

It may be desirable to provide multicast routing within a Layer 3 mesh, either 
for nodes or maybe also for clients. We could probably build parts like 
l3roamd, the client prefix daemon and telemetry daemons ontop a multicast 
network.

# Telemetry

There’s no concept for telemetry (think alfred, respondd) for a Layer 3 mesh 
yet. Collecting information locally is trivial. Getting it to some sinks not so 
much.

