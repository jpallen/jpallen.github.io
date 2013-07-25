---
title: Mongoose vs mongojs benchmarks
layout: post
tags : [nodejs, mongodb, development, mongojs, mongoose]
published: false
---

*Quick conlusion: mongojs can handle approximately 1.4-1.5 times the requests in the
same time period as mongoose. However, this is only the result of a naive
benchmark that should be considered 'back of the envelope' at best.*

We've recently been hitting a bottle neck with talking to a MongoDB database
in one of the Node.js application that I maintain. To improve this I've had to
get a better understanding of the possible throughput that we can get when
calling the database. Our application is currently using mongoose as a
wrapper to talk to MongoDB since it provides some nice ORM features on top of
just talking to the database. However, these features will no doubt cause a
performance hit which might be too large to be worthwhile in a high throughput
application. Mongojs is a much thinner wrapper around MongoDB that really just
provides a clean interface to talk to the database with, and doesn't do any
'magic'. Let's find out how they compare.

Testing conditions
------------------

Any benchmark is necessarily a little artifical, so please bear this in mind
while reading this post. At the moment I'm only interested in `find` operations
since that's where our bottleneck was occuring. In my benchmark I used what I
would consider a small to medium sized document, with a single attribute with
1000 words of Lorem Ipsum text. The tests were run on the same machine as the
database so latency was negligable. I ran a set number of find queries for the
test document over 10 seconds and recorded the time taken from issuing the query
to getting a response through the driver's interface for each request.
The version of Node.js used was 0.6.19, mongojs was 0.7.2 and
mongoose was 3.5.8. The graphs below show a histogram of the distribution of
response times for a given number of requests per second with both mongojs and
mongoose.

Results
-------

### mongojs

The following graphs show the response times for mongojs when finding a document
with 1000 words of text. The graphs show the result of running between 3000 and
3500 queries per second. There is a very noticable change in the distribution of
the response times and a significant number of queries begin to take longer than
100ms. I think it is fair to say that this is range within which mongojs would
fail to meet our needs in production.

<div class="figure" markdown="1">
	<div class="subfigure" markdown="1">
		![Mongojs at 3000 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3000-per-second.png)
		<div class="caption">
			3000 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 3100 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3100-per-second.png)
		<div class="caption">
			3100 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 3200 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3200-per-second.png)
		<div class="caption">
			3200 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 3300 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3300-per-second.png)
		<div class="caption">
			3300 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 3400 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3400-per-second.png)
		<div class="caption">
			3400 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 3500 req per
		second](/assets/images/posts/mongo-benchmarks/mongojs-3500-per-second.png)
		<div class="caption">
			3500 req/s
		</div>
	</div>
</div>

The similar change for the mongoose driver takes place within the range between
2000 and 2500 requests per second, as shown in the graphs below.

<div class="figure" markdown="1">
	<div class="subfigure" markdown="1">
		![Mongojs at 2000 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2000-per-second.png)
		<div class="caption">
			2000 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 2100 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2100-per-second.png)
		<div class="caption">
			2100 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 2200 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2200-per-second.png)
		<div class="caption">
			2200 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 2300 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2300-per-second.png)
		<div class="caption">
			2300 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 2400 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2400-per-second.png)
		<div class="caption">
			2400 req/s
		</div>
	</div>
	<div class="subfigure" markdown="1">
		![Mongojs at 2500 req per
		second](/assets/images/posts/mongo-benchmarks/mongoose-2500-per-second.png)
		<div class="caption">
			2500 req/s
		</div>
	</div>
</div>

So both mongojs and mongoose fail at a similar order of magnitude of requests,
with mongojs unsurprisingly taking the lead by a factor of around 1.4 to 1.5
times the possible throughput.

Interestingly, there is little to differentiate the two a lower rates of
requests. The following graphs show the results of both mongojs and mongoose at
a rate of 1500 requests per second where they both perform admirably.

Discussion
----------

Above the thresholds shown here for mongojs and mongoose, the request times
becomes distributed quite evenly but get longer and longer. This indicates to me
that we are hitting some threshold of maximum throughput somewhere, above which
the requests to MongoDB are sitting in a queue waiting to be process. From these
results it's not clear where this bottleneck is, but I suspect it could be a 
a CPU bound operation to do with deserializing the results (where mongoose would
be expected to have a higher overhead).

Conclusion
----------

I don't think that the difference between mongojs and mongoose is sufficient
enough to strongly influence our choice of driver when starting out. By the time
you are pushing a thousand requests per second you will likely have bigger
scaling problems to worry about and should be doing your own benchmarks anyway!

Appendix
--------

If you're interested, here is the result of using a document which is 10 times
larger than the test case above. That is, 10000 words per document.

### mongojs

### mongoose

