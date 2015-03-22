---
layout: post
title:  "A Gentle Introduction to Riak Core"
date:   2014-12-18 23:56:45
categories:
- blog
permalink: riak-core_intro
---

## What is Riak Core?

People tend to refer to Riak Core as a framework or
toolkit for developing scalable, fault-tolerant,
distributed applications. However, I find this
description somewhat misleading. After reading this description,
people automatically assume that Riak Core ships with
off-the-shelf replication and synchronisation algorithms.
People familiar with Riak are tempted to (mistakenly) think that
replication and synchronisation in Riak Core boils down to
setting the right `N`, `R` and `W` values (in the context of
a [quorum-like algorithm](http://en.wikipedia.org/wiki/Quorum_%28distributed_computing%29)).
Nothing is further
from the truth. I personally prefer to think of Riak Core as a
scalable, fault-tolerant distributed look-up service similar to
the one described in [Stoica et al., 2001](http://people.csail.mit.edu/imcgraw/links/research/pubs/networks/Chord.pdf)
(I want to emphasise the "look-up service" bit).

If you are skeptical about Riak Core's scalability, you may
like to know that Riak Core is developed by [Basho, Inc.](http://basho.com/)
and is the cornerstone of [Riak](http://basho.com/riak/)
(Basho's flagship product and one of the most popular NOSQL
databases in the market).


## How can I get Riak Core?

Riak Core is an open-source project and can be downloaded from
its [GitHub repository](https://github.com/basho/riak_core).


## Concepts

Riak Core is heavily influenced by [Amazon's Dynamo paper](http://www.read.seas.harvard.edu/~kohler/class/cs239-w08/decandia07dynamo.pdf). Everything in Riak Core revolves around a circular
key-space (a.k.a. the ring) and the concept of virtual
nodes (hereinafter referred to as vnodes).

![Riak Core key-space]({{ site.url }}/assets/images/ring.png)

Riak Core has, at its core, a 160-bit circular key-space like
the one in the image above (which by the way I've got from
Basho's website). Note that the ring is divided into
a fixed number of partitions, which determine the number of
vnodes in the system. Riak Core uses consistent hashing[^1] to
map keys to specific positions in the key-space. This mapping
determines what partition is responsible for this key. Hence,
it also determines which vnode is responsible for that key.

So far, I have used the word vnode a couple of times, but what a
vnode really is is yet to be explained. A vnode (remember, a
virtual node) is nothing but an Erlang process responsible for
one (and only one) partition of the ring. A physical node
(represented as different colours in the image above) can (and most
likely will) host more than one vnode at a time. It is
important to bear in mind that vnodes are not bound to a
particular (physical) node. They can be relocated to other
nodes as new nodes are added to (or existing nodes are removed
from) the system.

Earlier in this write-up I said that Riak Core is a distributed
look-up a service, but all the concepts I have covered up until
now could very well be used to describe how a distributed key-value
database. Note, however, that Riak Core, by itself, has nothing
(or very little) to do with storage, data replication, nor many
other fundamental aspects of database design. Instead, Riak Core
is all about routing requests to their corresponding vnodes
based on a (routing) key.

[^1]: Is a special kind of hashing that minimises the number of keys that must be remapped when the hash-map is resized.


## Using Riak Core

Using Riak Core is simpler than what it might seem at first. It
boils down to making all pertinent initialisations, developing a
process implementing the vnode behaviour, and operating a
Riak Core cluster.

### Initialisation

If one wants to use Riak Core, the first thing he/she must do
is to initialise the Riak Core ring. This is done by means of
the `riak_core:register/2` function. This function takes a
unique name (an Erlang atom) and a list of properties
as input. At the very least, the user
has to specify a unique name for the ring and a module implementing
the vnode behaviour (e.g.,
`riak_core:register(myring, [{vnode_module, myapp_vnode}])`). Note
that we can only specify one vnode behaviour module. That is, all
vnodes in the ring are equal (except for the partition of the ring
they are responsible for, and the state they may hold).

Once the ring has been initialised, one must also start the
`riak_core_vnode_master` process (this is usually done by adding it
to the supervision tree). This process is responsible
for handling all requests and forwarding them the their
corresponding vnodes.

![Supervision Tree]({{ site.url }}/assets/images/suptree.png)

The image above illuestrates the supervision tree of a Riak Core
application (called memy), whose ring has 8 partitions.


### Riak Core "API"

As soon as one is done with the initial setup, one can exploit
Riak Core features by means of the `riak_core_util:chash_key/1`,
`riak_core_apl:get_apl/3`, `riak_core_vnode_master:command/3`,
`riak_core_vnode_master:sync_command/3`, and
`riak_core_vnode_master:sync_spawn_command/3` functions.

#### riak_core_util:chash_key/1
{% highlight erlang %}
-spec chash_key(Key :: {any(), any()}) -> binary().
{% endhighlight %}

This function is used to hash a (routing) key. It returns an
integer between 0 and 2^160 - 1. The obtained integer refers to
a particular position in Riak Core's 160-bit circular key-space.

> Note that this function takes a 2-element tuple as input (an
> obvious artifact from the times when Riak Core was hardcoded
> in Riak).

#### riak_core_apl:get_apl/3

{% highlight erlang %}
-spec get_apl(HashedKey :: binary(), N :: non_neg_integer(), Ring :: atom()) -> [{integer(), node()}].
{% endhighlight %}

This function is used to get the Active Preference List (APL)
for a given (hashed) key. One can specify how many vnodes will
be returned. An exception is thrown if the requested number of
vnodes is greater than the number of partitions. The last argument
must match the unique name given to the ring in
`riak_core:register/2`.

Each vnode is represented as a 2-element tuple where the first
element denotes the first key in the partition the vnode is
responsible for (as an integer), and the second element refers
to the (physical) node the vnode is running on.

#### riak_core_vnode_master:command/3

{% highlight erlang %}
-spec command(To :: {integer(), node()}, Request :: any(), Master :: atom()) -> 'ok'.
{% endhighlight %}

This function is similar to gen_server's cast/2 function. It
is used to send a request to a particular vnode process. Note,
however, that, unlike gen_server's cast/2, this function
requires a third argument, which is nothing but a reference
to the `riak_core_vnode_master` one must start when
initialising Riak Core's ring.

Like gen_server's cast/2, this function does not generate a
response from the vnode process handling the request.

#### riak_core_vnode_master:sync_command/3

{% highlight erlang %}
-spec sync_command(To :: {integer(), node()}, Request :: any(), Master :: atom()) -> any().
{% endhighlight %}

This function is similar to gen_server's call/2 function. It
is used to send a request to a particular vnode process. Note,
however, that, unlike gen_server's call/2, this function
requires a third argument, which is nothing but a reference
to the `riak_core_vnode_master` one must start when
initialising Riak Core's ring.

This function blocks the `riak_core_vnode_master` process. That is,
it prevents the `riak_core_vnode_master` process to handle multiple
requests concurrently.

#### riak_core_vnode_master:sync_spawn_command/3

{% highlight erlang %}
-spec sync_spawn_command(To :: {integer(), node()}, Request :: any(), Master :: atom()) -> any().
{% endhighlight %}

This function is similar to gen_server's call/2 function. It
is used to send a request to a particular vnode process. Note,
however, that, unlike gen_server's call/2, this function
requires a third argument, which is nothing but a reference
to the `riak_core_vnode_master` one must start when
initialising Riak Core's ring.

Unlike `sync_command/3`, this function does not block the
`riak_core_vnode_master` process. That is, it lets the
`riak_core_vnode_master` process handle multiple requests
concurrently.

#### riak_core_vnode_master:coverage/5

TBD


### The vnode behaviour

The vnode behaviour is, in many things, similar to Erlang/OTP's
gen_server behaviour. This behaviour lets you model the behaviour
of the vnode processes used by your Riak Core application (e.g.
what requests are supported, what to do when a node is added to
the system, ...). You will find its implementation details in the
`riak_core_vnode` module.

#### init/1
{% highlight erlang %}
-spec init([integer()]) -> {'ok', State :: any()} | {'stop', Reason :: any()}.
{% endhighlight %}

This callback function is similar to gen_server's init/1 callback.
It is used to initialise a vnode process.

#### terminate/2
{% highlight erlang %}
-spec terminate(Reason :: any, State :: any()) -> 'ok'.
{% endhighlight %}

This callback is similar to gen_server's terminate/2 callback.
It is called when the vnode process is about to terminate. It
can be thought as the opposite of `init/1`. It can be used to
do any necessary cleaning up.

#### handle_exit/3
{% highlight erlang %}
-spec handle_exit(ExitingPid :: pid(), Reason :: any(), State :: any()) -> {'stop', Reason :: any(), State :: any()} | {'noreply', State :: any()}.
{% endhighlight %}

This function is called when a process linked to the vnode
process dies. Two actions can be taken:

1. Terminate the vnode process (`{'stop', Reason :: any(), State :: any()}` must be returned).
2. Ignore the dying process (`{'noreply', State :: any()}` must be returned).


#### handle_command/3

{% highlight erlang %}
-spec handle_command(From :: 'ignore' | sender(), Request :: any(), State :: any()) -> {'reply', Reply :: any(), State :: any()} | {'noreply', State :: any()}.
{% endhighlight %}

This function is similar to to gen_server's handle_cast, handle_call
and handle_info. It is used to handle incoming requests that has been
sent by a caller process using functions such as
`riak_core_vnode_master:command/3`, `riak_core_vnode_master:sync_command/3`
and `riak_core_vnode_master:sync_spawn_command/3`.

Unlike gen_server processes, vnode processes do not have different
callback functions for different types of requests.

> Note that `From` is set to `ignore` when the request is sent using
> `riak_core_vnode_master:command/3`.

Like the gen_server module, the riak_core_vnode module also implements
a `riak_core_vnode:reply/2` function, which can be used to reply back
to the caller process while handling requests concurrently.

#### handle_coverage/4
TBD

#### is_empty/1

{% highlight erlang %}
-spec is_empty(State :: any()) -> {boolean(), State :: any()}.
{% endhighlight %}

This function is called when a node is added to (or removed from)
the system, which will trigger a handoff process. This function
is not called on all vnode process, but only on those affected
by the handoff.

This function returns a boolean indicating if the vnode holds
any data that needs to be transfer to the new vnode process.

#### delete/1

{% highlight erlang %}
-spec delete(State :: any()) -> {'ok', State :: any()}.
{% endhighlight %}

This function is called when a vnode process is deemed empty.
It can be used to perform a preemptive cleanup of the vnode's
resources.

#### handoff_starting/2

{% highlight erlang %}
-spec handoff_starting(TargetVnode :: {integer(), node()}, State :: any()) -> {boolean(), State :: any()}.
{% endhighlight %}

This callback function is called when Riak Core determines
that a handoff process affecting this vnode process must
be started.

It returns a boolean indicating whether the handoff process
must be interrupted.

#### handoff_cancelled/1

{% highlight erlang %}
-spec handoff_cancelled(State :: any()) -> {'ok', State :: any()}.
{% endhighlight %}

This function is called when a handoff process affecting
this vnode process gets cancelled. It can be used to undo
changes made in `handoff_starting/2`.

#### handoff_finished/2

{% highlight erlang %}
-spec handoff_finished(TargetVnode :: {integer(), node()}, State :: any()) -> {'ok', State :: any()}.
{% endhighlight %}

This function is called when Riak Core is done with
a handoff process affecting this vnode process.
That is, when all the data that had to be
transferred to the new vnode process has been
successfully migrated.

#### encode_handoff_item/2

{% highlight erlang %}
-spec encode_handoff_item(Key :: any(), Value :: any()) -> binary().
{% endhighlight %}

This callback function is used to encode the
data that needs to be transferred to a new
vnode process. It is called on the vnode
process that is giving up the ownership
of the partition.

#### handle_handoff_data/2

{% highlight erlang %}
-spec handle_handoff_data(Data :: binary(), State :: any()) -> {'reply', State :: any()}.
{% endhighlight %}

This callback function is used to decode
data received due to a handoff process. It
is called on the vnode process that is claiming
the ownership of the partition.

#### handle_handoff_command/3

{% highlight erlang %}
-spec handle_handoff_command(From :: 'ignore' | sender(), Request :: any(), State :: any()) -> {'drop', State :: any())} | {'forward', State :: any()} | {'reply', Reply :: any(), State :: any()} | {'noreply', State :: any()}.
{% endhighlight %}

This callback is similar to `handle_command/3`.
The difference is that this function is called
when the vnode process receives a request during
a handoff.

In addition to the typical `handle_command/3`
return types, this function can perform two
other tasks:

1. Forward the request so that it get handled
by the target vnode process.
2. Drop the request.

You need to implement this callback if you want
to be able to transfer data between vnode processes
due to the scheduling of a handoff. More concretely,
you will have to implement a clause for this
function that handles the `?FOLD_REQ` requests.
The `?FOLD_REQ` macro is provided by
Riak Core and is defined in the `riak_core_vnode`
header file. This request is used to convert a
vnode's state into a list of key-value pairs
that will later be used by the `encode_handoff_item/2`
callback function. Note that the `?FOLD_REQ` expands
to a record that contains both a folding function
(i.e. foldfun) and an initial accumulator (i.e. acc0).
The provided folding function and initial accumulator
work just fine on functions like dict:fold/3, map:fold/3,
... so, most of the times, implementing this function
clause will be a no brainer.


### Operating a Riak Core cluster

Once we are done with all the nifty implementation details
of our Riak Core application, it is just about time that
our resource-hungry application will start asking for another
(Erlang) node to run on. If we got our implementation right,
there is nothing to fear. You just have to deploy a second
Erlang node (running your application) and let Riak Core
take care of the rest. Riak Core ships with the
`riak_core:join/1` function, which can be used (on a new node)
to join an existing Riak Core cluster. This function takes
a (remote) Erlang node as input (e.g. myapp@192.168.1.10).
Once called, Riak Core will start its internal machinery
to ensure that all vnode processes are properly balanced
across all nodes in the cluster (remember that Riak Core
uses a consistent hashing algorithm that minimises the
amount of vnode processes that must be migrated when the
cluster is resized).

Similarly, Riak Core ships also with the `riak_core:leave/0`
function, which can be used on a node in order to make it
leave the Riak Cluster the node is currently participating
in.

Last but not least, Riak Core provides some functions such
as `riak_core_status:ringready/0` and
`riak_core_ring_manager:get_my_ring/0` that may come in
handy when it comes to inspect the state of Riak Core's
ring.