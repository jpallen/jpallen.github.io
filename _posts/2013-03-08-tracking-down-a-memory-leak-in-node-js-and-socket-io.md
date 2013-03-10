---
title: Tracking down a memory leak in Node.js and Socket.IO
tagline: 
layout: post
tags : [nodejs, socketio, development]
reddit: http://www.reddit.com/r/node/comments/1a1l46/tracking_down_a_memory_leak_in_nodejs_08_and/
hn: http://news.ycombinator.com/item?id=5353773
---

*If you are running* **Node.js 0.8** *and* **Socket.IO** *over* **HTTPS** *then you will be
affected by this memory leak. See the bottom of the post for details of a fix.*

Node.js is a great bit of kit, but it's still a relatively young technology.
I was bitten by this recently when I had to investigate a large memory leak 
in one of the Node.js apps I maintain. There are tools that can help to
track down these sorts of problems, but it still took lots of trial and
error to understand, and I thought it would be worthwhile to share my
experience here.

Let me start by showing just how extreme the memory leak was. The graph
below shows two very normal things that can happen in a Node.js app: connections
coming in via WebSockets using the [Socket.IO](http://socket.io/) library, and uploading files. If we do either of
these individually then there is no problem and the memory flattens off nicely, but when
we send uploads and WebSocket connections together, **BOOM**, there is linear memory
growth:

![Memory leaking Node.js process](/assets/images/posts/memory-leak.png)

The Node.js version is 0.8.21 and the Socket.IO version is 0.9.13 here.

Getting a heapdump
------------------

Before doing any sort of debugging, I needed a snapshot of the state of the
memory of the Node.js app. Since the memory leak was so extreme, I only had to
wait a few minutes to be confident that a large proportion of the Node.js process's
memory was taken up by the leaking objects. To get a dump of the process's
memory, I used the [heapdump](https://github.com/bnoordhuis/node-heapdump)
module. To take a snapshot of the current state of the program's memory, it's
as simple as:

<pre class="prettyprint language-js">
heapdump = require("heapdump")
heapdump.writeSnapshot()
</pre>

Of course, there's no point doing this immediately when the application starts.
A clever trick that we use for things like this is to allow telnet access
directly to a REPL running in the context of the Node.js process (example in coffeescript):

<pre class="prettyprint language-js">
require("net")
  .createServer (socket) ->
    repl = require('repl')
    repl.start("my-node-process>", socket)
  .listen 5000, "localhost"
</pre>

I can then telnet in and take a snapshot after the process has grown it's memory
footprint large enough to be worth investigating:

	$ telnet localhost 5000
	my-node-process>require("heapdump").writeSnapshot()

This will write out a file called something like `heapdump.1234567890.heapsnapshot` 
to the same directory as the application's code.

Inspecting the heap snapshot
----------------------------

Node.js is based on the V8 javascript engine used by the Chrome browser. This is
very handy, because Chrome has a great debugging tool built in for inspecting
heap snapshots from V8, and it works well for inspecting the heap snapshot of a Node.js
process.

To load the heap snapshot, open up the Chrome developer tools, and go to the
'Profiles' tab. Right click in the left hand side bar, select 'Load Heap
Snapshot', and load the snapshot that was taken previously by `heapdump`.
The heap snapshot inspector lets one see all the objects in ones Node.js
app's memory at the time of the snapshot and shows
how many of each object there are and how much memory they are using. After sorting by
size, it was immediately clear that around 200Mb of my app's memory was taken up
by buffers. Expanding the Buffer list, I saw that there were many 10Mb Buffers
hanging around. Whatever is creating and holding onto those is undoubtably the
cause of the leak.

![Chrome Heap Snapshot Inspected](/assets/images/posts/chrome-heap-inspector.png)

Clicking on an object in the heap inspector shows one the object's retaining
tree. That is, the parent objects that hold a reference to that object, and the
objects that hold a reference to those, and so on right up to the top level of
your application. The way that Node.js's garbage collecting works means that an
object will not be removed from memory until there are no references to it from
elsewhere in the app. By examining a few of these buffers, it was clear that
they were being held onto by a `req.head` object which was referenced by a
WebSocket object. At this point, Socket.IO was starting to look suspicious.

The problem
-----------

A quick grep through the source code of Socket.IO quickly turned up
the following lines as the likely origin of the leaking `req.head` object:

<pre class="prettyprint language-js">
Manager.prototype.handleUpgrade = function (req, socket, head) {
  ...
  req.head = head
  ...
}
</pre>

This is called when a regular HTTP connection requests to be upgraded to a
WebSocket connection. From the Node.js docs for the `upgrade` event:

> head is an instance of Buffer, the first packet of the upgraded stream, this may be empty.

So it's a Buffer as expected. However, it's only the first packet, so
it certainly shouldn't be taking up 10Mb by itself, and from the heap snapshot
it looked like there were multiple `req.head` objects pointing to the same
Buffer. At this point I must admit to being a bit stuck and
flailing around on Google for a while. I did eventually find the answer though:

In Node.js version 0.8, the `tls` module, which is used by the `https` server in
Node.js, allocates 10Mb `SlabBuffer` objects up front. It does this because
allocating a buffer is relatively slow, and rather than allocating new buffers
for each incoming request it can instead quickly allocate portions
of this large buffer.
The `head` buffer, which is passed to Socket.IO, is allocated from this 10Mb `SlabBuffer`.
Crucially, the Node.js garbage collector will not remove the `SlabBuffer`
until all of its children buffers are removed, but
the `head` buffer is kept around for the whole time that the WebSocket
connection is open. This means that every WebSocket connection can potentially
be keeping 10Mb of space hanging around in memory.

This explains why our production process was seeing ever increasing memory
usage. We had lots of traffic over HTTPS so lots of these 10Mb `SlabBuffer`s
were being created, and we also had people constantly connecting via WebSockets and hanging around for
a while preventing the `SlabBuffer`s from ever being freed from memory.

The fix
-------

It's hard to fault the design of Socket.IO here. There is no indication that
keeping this small `head` buffer around should have such dire consequences.
It's also hard to fault the design in Node.js since in isolation the
`SlabBuffer` implementation is great idea for improving the speed of Node.js. I
think this is just one of those subtle bugs from an unexpected interaction
between two systems which both seem to be acting sensibly on the surface.

There are a few patches out there which address this issue for Socket.IO: 
[one here by jmatthewsr-ms](https://github.com/LearnBoost/Socket.IO/pull/1143) and
[one here by me](https://github.com/LearnBoost/Socket.IO/pull/1178). These
haven't been merged yet since there is some ongoing discussion about whose fault
this really is. There is [some suggestion](https://github.com/joyent/node/pull/4660)
that this should actually be fixed in
Node.js to avoid catching others out in the future. Either way, I think this is
a crucial problem that the community at large should be more aware of.

For an immediate fix, you can monkey patch Socket.IO to include an additional
last line in the `handleUpgrade` method in `lib/manager.js` which frees the
`req.head` buffer:

<pre class="prettyprint language-js">
Manager.prototype.handleUpgrade = function (req, socket, head) {
  var data = this.checkRequest(req)
    , self = this;

  if (!data) {
    if (this.enabled('destroy upgrade')) {
      socket.end();
      this.log.debug('destroying non-Socket.IO upgrade');
    }

    return;
  }

  req.head = head;
  this.handleClient(data, req);

  // Insert this line:
  delete req.head;
};
</pre>

A fix which I haven't tested but may help without solving the underlying cause
is to shrink the size of the SlabBuffers allocated by the `tls` module:

<pre class="prettyprint language-js">
require('tls').SLAB_BUFFER_SIZE = 100 * 1024 # 100Kb
</pre>

You can only do this in Node 0.8.20 and above though. I haven't tested this to
see if makes much difference, or what affect it has on speed. Use at your own
risk.

Conclusion
----------

This is a subtle bug, yet I suspect it is responsible for many of the complaints
about Socket.IO being a memory hog. The mix of circumstances are almost
certainly found in a large number of production services. WebSockets are a
common use case for Node.js and everyone should be using HTTPS wherever possible.

If this is something which has affected you and my post has helped out, please
let me know. Contact details are in the side bar.



