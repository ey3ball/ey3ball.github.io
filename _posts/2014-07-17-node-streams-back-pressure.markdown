---
layout: post
title:  "Understanding node streams, back-pressure ... the hard way"
date:   2014-07-17 01:00:00 +0200
categories: nodejs
---

# Context

Recently, I began using **NodeJS** for a small project. (see
[here](http://github.com/ey3ball/node-iptv-proxy))

The idea was to build an http proxy for live iptv streams (in a follow up post,
I'll explain its purpose in more details). Node seemed particularly well suited
for this :

  * It natively speaks http
  * Its data handling APIs are focused on a concept of stream, making it easy
    to "connect" a data source (the iptv stream) and a sink (the client
    application, your video player).

I quickly found out however, that my requirements were a bit exotic : given
that I'm proxying **live** streams, I'm not interested in *reliability*.
Actually it's the exact opposite : I needed my proxy to send data out as fast
as possible, potentially to multiple clients. If some of these clients are
requesting the same stream and one of them is too slow to keep up with the data
rate, it should experience packet loss. This way fast clients always read a
correct stream, while slow clients will see choppy playback and / or dropped
audio/video frames (but this will enable them to keep to stay "in sync" with
the source).

The issue is, the philosophy around NodeJS streams is the exact opposite : they
are **pull** based. They apply
[back-pressure](http://howtonode.org/streams-explained) on the source. This
basically means that if you attempt to
[pipe](http://nodejs.org/api/stream.html#stream_readable_pipe_destination_options)
some source data into multiple destinations, it will only flow as fast as the
slowest client can handle. Worse even, since NodeJS streams implement an
internal back-pressure algorithm, this will go up the chain and the slowest
client will impose its read rate on the source, which means you don't even get
a chance to drop packets by implementing a lossy "pipe" in-between (spoiler:
that's unless you understand node's internals correctly).

It took me a little while to wrap my head around how these mechanisms are
managed by NodeJS (and how to bypass them for my particular use case). This
article attempts to document various findings I've made along the way.

# Streams 101

NodeJS basically builds upon two types of stream :

  * Writable streams (streams you can send data to)
  * Readable streams (streams you can read data from)

A stream that's writable only is basically a sink, while a stream that's
readable only is a source.

Anything in-between must be a Duplex stream (both Readable and Writable). A
Transform stream is a special kind of Duplex stream, where the output is
computed directly from the source. A PassThrough stream is a Duplex/Transform
stream that does nothing but let data flow, untouched.

NodeJS provides a native way to build chains of streams linking a source to
sink thanks to the `pipe` method. This makes building data processing pipeline
easy.

Here is a basic example usage of using and connecting streams :

```javascript
/* process.stdin is a Readable stream
 * process.stdout is a Writable stream
 * 
 * We pipe stdin to stdout, this is similar
 * to the cat unix command
 */
process.stdin.pipe(process.stdout);

/* We could insert a duplex stream in-between,
 * for instance to implement "grep"
 */
process.stdin.pipe(new FilterStream_Grep("hello")).pipe(process.stdout)

```

Back to our use case, how do you proxy one source http stream to several
clients in NodeJS that's as simple as :

```javascript
source = http.request("http://....", function(http_source_stream) {
        http_source_stream.pipe(http_client1_output);
        http_source_stream.pipe(http_client2_output);
});
```

# A simple, leaky stream attempt

Back to our lossy pipe problem.

What was so hard ? When I ran into the issue my first though was : Ok, let's
write a dummy "leaky" stream. It will let data flow, like a PassThrough stream,
unless the destination's internal buffer becomes full. Should that happen our
stream will simply drop chunks until the destination becomes available again.

Simple enough right ?

Here is what it looks like after a dozen of minutes hacking around (in terrible
inline creation style as I was just trying to figure out this stuff as I wrote
it) :

```javascript
/* out leaky stream is a transform stream, output is exactly the input, unless
 * we need to drop some chunks */
var leaky = new require('stream').Transform();

/* We're using the ringbuffer npm package:
 * http://www.npmjs.org/package/ringbufferjs
 * basically:
 *  * enq: push data in buffer
 *  * dec: pull data from buffer
 *
 * if the buffer is full enq will overwrite the oldest
 * element in the ring
 */
leaky.rb = new ringbuffer(20);

leaky.target_ready = true;

/* implement a leaky passthrough using a transform stream that lossily
 * stores chunks in a ringbuffer
 *
 * see: http://nodejs.org/api/stream.html#stream_class_stream_transform 
 */
leaky._transform = function(chunk, encoding, done) {
        /* Push data to ringbuffer, eventually overwritten some other sample */
        this.rb.enq(chunk);

        if (this.target_ready) {
                /* if the destination is ready to receive, grab chunk from buffer */
                var send = this.rb.deq();

                /* push data out
                 * note: this.pipes contains an handle to a piped (.pipe())
                 * destination stream */
                if (!this.push(send.data) && this.pipes) {
                        /* push returns false: we're getting back-pressured */

                        console.log('full');
                        this.target_ready = false;

                        /* catch the drain event which tells us the destination is
                         * ready to receive again */
                        this.pipes.once('drain', this.got_drain.bind(this));
                }
        }

        done();
}

/* resume sending data to the destination */
leaky.got_drain = function() {
        var go = true;
        console.log('drain');

        while (!this.rb.isEmpty() && go) {
                go = this.push(this.rb.deq());
        }

        if (!go) {
                /* we're stuck again ! */
                this.pipes.once('drain', this.got_drain.bind(this));
        } else {
                this.target_ready = true;
        }
}

/* insert our leaky passthrough attempt in-between the source and
 * our destination */
source.pipe(leaky).pipe(destination);
```

So basically we have created a transform stream. It pushes data chunks in a
circular buffer. As long as the piped destination is not overwelmed, it also
immediately pulls a data chunk out of that buffer and pushes it downstream.
When the destination signals it can't handle more data, we let the circular
buffer get filled and potentially overflowed. We only resume sending new chunks
once the destination stream sends us a 'drain' (= ready to go) event.

Notice the two logs I have inserted here. One would print 'full' as soon as the
destination signals a buffer overrun and 'drain' as soon as the destination is
back to a nicer buffering level.

To my surprise, when running the code above with a slow client, these two logs
are never displayed.

# Show me that back-pressure

Initially I though that push() returning false was node's way of propagating
back-pressure. After all, if you look at the (evasive) documentation on the
[subject](http://nodejs.org/api/stream.html#stream_event_drain) back-pressure
is mentionned once in an example that shows how to interact with Writable
streams correctly. It therefore semmed only natural that using the internal
push() call when connecting to streams at a lower level would have the same
semantics, especially since according to the
[doc](http://nodejs.org/api/stream.html#stream_readable_read_size_1) :

```javascript
        // if push() returns false, then we need to stop reading from source
```

Which sounds a lot like back-pressure handling to me !

Therefore my initial conclusion was that back-pressure is actually handled at a
lower level, and that I can't bypass it in a custom stream unless I start
messing with its deep internals.

# Working around back-pressure : fake fast drain

Since the dummy transform stream approach failed, I turned to a second
solution. This time I'd write a fake destination stream, that can receive data
at full speed by listening to data events. Then I'd manually handle the
transport to the real destination by calling the higher level "write" method,
which is used by non-streamy clients in order to to push data to anything
streamy.

The advantage of this second method is that since the destination and the
source are not linked through a sequence of pipes, nodejs can't back-pressure
my source stream against my will.

What does it look like ? Here is my leaky stream proxy. It does essentially the
same thing that my leaky transform stream (hence I did not heavily comment it),
only one API level above. 

```javascript
Ringbuffer = new require('ringbufferjs');
Writable = new require('stream').Writable;

util = require('util');

util.inherits(Leaky, Writable);

/* The destination stream is given as the first argument when
 * instanciating this "leaky" stream proxy
 *
 * instead of using 
 *   $ source.pipe(leaky).pipe(destination);
 * which creates a global chain "source => leaky => destination"
 * we'd use here :
 *  $ source.pipe(new Leaky(destination));
 * which creates two chains :
 *   + source => leaky
 *   + leaky => destination
 * ie: destination is "hidden" behind leaky
 */
function Leaky(writable, opt) {
        Writable.call(this, opt);

        this._rb = new Ringbuffer(5);
        this._target_ready = true;
        this._target_stream = writable;
}

/* The streamy part, implement a sink (Writable-only stream) by
 * providing a _write method */
Leaky.prototype._write = function(chunk, encoding, done) {
        if (!this._target_stream)
                return done();

        this._rb.enq(chunk);

        if (this._target_ready) {
                /* Use the external API of Writable streams to push data :
                 * the effect is the same as a pipe (we move data around),
                 * except here the destination thinks we're not a "streamy"
                 * client since streams are not connected through this
                 * usual mecanism */
                if (!this._target_stream.write(chunk)) {
                        this._target_ready = false;
                        this._target_stream.once('drain', this._got_drain.bind(this));
                }
        }

        done();
}

Leaky.prototype._got_drain = function() {
        var go = true;

        while (!this._rb.isEmpty() && go) {
                go = this._target_stream.write(this._rb.deq());
        }

        if (!go) {
                this._target_stream.once('drain', this._got_drain.bind(this));
        } else {
                this._target_ready = true;
        }
}

module.exports = Leaky;
```

Does this work ?

Yes, finally ! As I suspected back-pressure doesn't hurt us anymore : the
source stream can be read at full speed and with this proxy inserted, a fast
client won't get stalled by a second slow destination. As expected these now
experience garbled video, lost frames, macroblocks ... you name it.

# Streams : deep dive

Now that we're starting to grasp what's happening here, time to get our hands dirty.

NodeJS streams are defined here :

  * <https://github.com/joyent/node/blob/master/lib/_stream_writable.js>
  * <https://github.com/joyent/node/blob/master/lib/_stream_readable.js>
  * <https://github.com/joyent/node/blob/master/lib/_stream_transform.js>
  * <https://github.com/joyent/node/blob/master/lib/_stream_duplex.js>

Looking at the `_stream_transform.js` file almost immediately confirms what we've
been able to infer : 

```javascript
// In a transform stream, the written data is placed in a buffer. When
// _read(n) is called, it transforms the queued up data, calling the
// buffered _write cb's as it consumes chunks. If consuming a single
// written chunk would result in multiple output chunks, then the first
// outputted bit calls the readcb, and subsequent chunks just go into
// the read buffer, and will cause it to emit 'readable' if necessary.
//
// This way, back-pressure is actually determined by the reading side,
// since _read has to be called to start processing a new chunk. However,
// a pathological inflate type of transform can cause excessive buffering
// here. For example, imagine a stream where every byte of input is
// interpreted as an integer from 0-255, and then results in that many
// bytes of output. Writing the 4 bytes {ff,ff,ff,ff} would result in
// 1kb of data being output. In this case, you could write a very small
// amount of input, and end up with a very large amount of output. In
// such a pathological inflating mechanism, there'd be no way to tell
// the system to stop doing the transform. A single 4MB write could
// cause the system to run out of memory.
```

Indeed going back to our initial observations regarding the semantics of
`push()`, it seems that it is definitely involved in handling back-pressure,
however, what we can understand here, is that push only ever returns false if a
transform stream writes **more data** that what it has *been fed with*.

In our leaky transform scenario, this mechanism never gets trigerred since we
write at most the same amount of data we're provided with. As a consequence
`push()` never attempts to back-pressure us. As suspected some other mecanisms
are in place, here it seems that we're getting throttled by someone else : the
`_read` call.

Let's dig a bit further and have a look at `_read` :

```javascript
// Doesn't matter what the args are here.
// _transform does all the work.
// That we got here means that the readable side wants more data.
Transform.prototype._read = function(n) {
  var ts = this._transformState;

  if (!util.isNull(ts.writechunk) && ts.writecb && !ts.transforming) {
    ts.transforming = true;
    this._transform(ts.writechunk, ts.writeencoding, ts.afterTransform);
  } else {
    // mark that we need a transform, so that any data that comes in
    // will get processed, now that we've asked for it.
    ts.needTransform = true;
  }
};
```

So `_read` calls `_transform`. More importantly looking at the rest of the
file, `_read` is the **only** method that ever calls `_transform`. Hence
`_transform` will only be triggered whenever the endpoints requests (pulls)
data.
As a result, in a transform stream, if you don't emit more data than what you
get, you can't possibly overflow the output with calls to `push`.  On the other
hand, since `_transform` is not called by `_write`, `_write` only accumulate
data in a queue and this, as long as the transform buffer has some free space
left. Once that buffer becomes full, back-pressure will be propagated upstream,
no `_transform` involved.

# Pimp my pipe

In the end, can we get an object that would get us the same behaviour than our
successful leaky attempt, but at the same time be a native stream ?

Definitely !

Having seen some stream internals we understand that the solution is to use a
Duplex stream in place of a Transform stream. Indeed a duplex stream lets us
write custom `_read` and `_write` routines. Following our latest observations
this gives us full control over how and when back-pressure is applied.

```javascript
Ringbuffer = new require('ringbufferjs');
Duplex = new require('stream').Duplex;

util = require('util');

util.inherits(Leaky, Duplex);

function Leaky(opt) {
        Duplex.call(this, opt);

        this._rb = new Ringbuffer(10);
        this._wants_data = false;
}

Leaky.prototype._write = function(chunk, encoding, done) {
        this._rb.enq(chunk);

        if (this._wants_data)
                this._wants_data = this.push(this._rb.deq());

        done();
}

Leaky.prototype._read = function (size) {
        var go = true;

        while (!this._rb.isEmpty() && go) {
                go = this.push(this._rb.deq());
        }

        this._wants_data = go;
}

module.exports = Leaky;
```

Using this Leaky stream is fairly easy :

```javascript
source.pipe(new Leaky()).pipe(destination);
```

And that's it : Leaky will act as a lossy buffer between a fast source, and a
slow destination.
