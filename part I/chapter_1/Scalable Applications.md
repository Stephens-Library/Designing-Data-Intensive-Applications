# Scability
Even if a system is working reliably today, that doesn't mean it will work reliably in the future

One common reason for degradation is increased load: perhaps the system has grown from 10,000 concurrent users to 100,000 concurrent users, or from 1 million to 10 million

Perhaps it is processing much larger volumes of data than it did before

*Scalability* is the term we use to describe a system's ability to cope with increased load

Note, however, that it is not a one-dimensional label that we can attach to a system

It is meaningless to say "X is scalable" or "Y doesn't scale", rather discussing scalability means considering questions like "If the system grows in a particular way, what are options for coping with growth?" and "How can we add computing resources to handle the load"

## Describing Load
First, we need to succinctly describe the current load on the system

Load can be described with a few numbers which we call *load parameters*

The best choice of parameters depends on the architecture of your system, it may be:
- Requests per second to a web server
- The ratio of reads to writes in a database
- The number of simultaneously active users in the chat room
- The hit rate on a cache

Perhaps the average case is what matters, or the bottleneck is dominated by a small number of extreme cases

## Twitter Case Study
To make this idea more concrete, let's use Twitter as an example, using data published in November 2012, two of Twitter's main operations are:
**Post Tweet**
A user can publish a new message to their followers (4.6k requests/sec on average, over 12k requests/sec at peak)

**Homeline Timeline**
A user can view tweets posted by the people they follow (300k requests/sec)

Simplify handling 12,000 writes per second would be fairly easy, however Twitter's scaling challenge is not primarily due to tweet volume, but due to fan-out, each user follows many people, and each user is followed by many people, there are broadly two ways of implementing these two operations:

1. Posting a tweet simply inserts the new tweet into a global collection of tweets. When a user requests their home line timeline, looks up all the people they follow, finds all the tweets for each of those users, and merges them (sorted by time), in a relational database, it could be queried as such:

```sql
SELECT tweets.*, users.* FROM tweets
    JOIN users ON tweets.sender_d = users.id
    JOIN follows ON follows.followee_id  = users.id
    WHERE follows.follwer_id = current_user
```

This method is known as **fan-out read**!

![image](photos/twitter_schema.png)
*This figure depicts a simple relational schema for implementing a Twitter home timeline*

2. Maintain a cache for each user's home timeline, like a mailbox of tweets for each recipient user

When a user posts a tweet, look up all the people who follow that user, and insert the new tweet into each of their home timeline caches

The request to read the home timeline is then cheap because its result has been computed ahead of time

![image](photos/twitter_tweets_datapipelin.png)
*This figure, is Twitter's data pipeline for delivering tweets to followers, with load parameters as of November 2012*

It represents **method 2, fan-out write**, let's delve a bit deeper into it!

1. User Posts Tweet
    - A user generates a tweet that gets stored in the "All Tweets" database
    - This write rate is shown as 4.6k writes/sec which means 4,600 tweets are created every second globally
2. Fan-out process
    - The tweet is distributed to the timelines of the user's followers, this is called a **fan-out**
    - If a user has many followers the system must create multiple copies of the tweet to be added to the timelines of these followers
    - This results in a high volume of write operations (to their home timeline caches)
3. Home Timeline Retrieval
    - When a user logs in or accesses their home timeline, the system retrieves tweets from the timeline storage specific to the user

### Analyzing the Two Methods
The first version of Twitter used approach 1, but the systems struggled to keep up with the load of home timeline queries, so the company switched to approach 2

This works better as the average rate of published tweets is almost two orders of magnitude lower than the rate of home timeline reads, and so in this case it's preferable to do more work at write time and less at read time

However, the downside of approach 2 is that posting a new tweet now requires a lot of extra work. On average, a tweet is delivered to about 75 followers, so 4.6k tweets per second becomes 345k writes per second to the home timeline caches, but this average hides the fact that the number of followers per user varies widely, some users have over 30 million followers, this means that a single tweet may result in over 30 million writes to home timelines!

Twitter tries to deliver tweets to followers within 5 seconds, which is a significant challenge

In the example of Twitter, the distribution of followers per user is a key load parameter for discussing scalability, since it determines the fan-out load

Finally, Twitter is moving to a hybrid of these two approaches where a small number of users with a large number of followers are exempt from this fan-out, these tweets are fetched separately and merged with that user's home timeline when it is read

This means that for celebrities their tweets are stored separately in a central repository, when one of their followers reads their home timelines, the system dynamically fetches the celebrity's tweet from the "All Tweets" Database, which is merged with pre-distributed tweets from other sources

## Describing Performance
Once the load on your system has been described, you can investigate what happens when the load increases, there are two ways to look at it:

1. When the load parameter increases and the system resources are unchanged (CPU, memory, bandwidth), how is the performance affected?
2. When a load parameter is increased, how much must the resources increase to keep the performances unchanged

*For clarification purposes, a load parameter is a factor that impacts the load of a system, not the actual load itself*

Both questions require performance numbers, so let's look briefly at describing the performance of the system

In a batch processing system such as Hadoop, we care about *throughput*, the number of records we can process per second, or the total time it takes to run a job on a dataset of a certain size

In the online system, we care about *response times*, that is the time between a client sending a request and receiving a response

## Latency and Response Times
*Latency* and *response time* are often used synchronously, but they are not the same

- The response time is what the client sees, aside from the actual time to process the request it also includes network delays and queuing delays
- Latency is the duration that a request is waiting to be handled, during which it is *latent* awaiting service

In practice, in a system handling a variety of requests, the response time can vary a lot, we therefore need to think of response time not as a single number, but as a *distribution* of values

Most requests are reasonably fast, but there are always outliers, perhaps random additional latency can be introduced by:
- A context switch to a background process
- The loss of a network packet
- TCP retransmission
- A garbage collection pause
- A page fault forcing a read from disk
- Mechanical vibrations in the server rack, or any other causes

It's common to see the *average* (arithmetic mean) response time of a service reported, though this is not a good metric as it doesn't tell you how many users experience it

A better way is to use *percentiles*, this is a good metric if you want to know how long users typically have to wait

In order to figure out how bad outliers are, higher percentiles should be looked at, the *95th*, *99th* and *99.9* percentiles are common (abbreviated to *p95*, *p99*, and *p999*)

High percentile responses also known as *tail latencies* are important as they directly affect users' experience

For example, Amazon describes response time requirements for internal services in terms of 99.9th percentile even though it only affects 1 in 1,000 requests

This is because customers with the slowest requests are often those who have the most data on their accounts because they have made many purchases

On the other hand, optimizing the 99.99th percentile (the slowest 1 in 10,000) was deemed too expensive and to not yield enough benefit for Amazon's purposes

Percentiles are often used in *service level objectives* (SLOs) and *service level agreements* (SLAs), contracts that define the expected performance and availability of a service

These metrics set expectations for clients of the service and allow customers to demand a refund if the SLA is not met

Queuing delays often account for a large part of the response time at high percentiles, as a server can only process a small number of things in parallel

It only takes a small number of slow requests to hold up the processing of subsequent requests, an effect sometimes known as *head-of-line blocking*

Even if those subsequent requests are fast to process, the client will see a slow overall response time due to the time waiting for the prior request to complete, due to this it is important to measure response times on the client side

When generating a load artificially to test the scalability of a system, the load-generating client needs to keep sending requests independently of the response time

If the client waits for the previous request to complete before sending the next one, that behaviour has the effect of artificially keeping the queues shorter in test than reality, which skews the measurement

## Percentiles in Practice
High percentiles are especially important in backend services that are called multiple times as part of serving a single end-user request

Even if you make the calls in parallel, the end-user request still needs to wait for the slowest of the parallel calls to complete

The end-user request still needs to wait for the slowest of the parallel calls to complete, this effect is known as *tail latency amplification*

If you want to add response time percentiles to the monitoring dashboards you need to efficiently calculate them on an ongoing basis, for example you may want to keep a rolling window of response time of requests in the last 10 minutes, every minute you calculate the median and various percentiles over the values in the window and plot those metrics on a graph

The naive implementation is to keep a list of response times for all requests within the time window and to sort that list every minute

If that is too inefficient, some algorithms can can calculate a good approximation of percentiles at minimal CPU and memory cost, such as forward decay, t-digest, or HdrHistogram

Beware that averaging percentiles to reduce the time resolution or to combine data from several machines is mathematically meaningless, the right way of aggregating response time is to add the histograms
- Percentiles are not averages, they are specific points in the data distribution, if you average the 95th percentiles of two datasets, the result is not necessarily the actual 95th percentile of the combined data as this depends on the shape and size of the entire distribution

## Approaches for Coping with Load
How do we maintain good performance even when our load parameters increase by some amount?

### Vertical vs Horizontal Scaling
People often talk of a dichotomy between *scaling up* (*vertical scaling*, moving to a more powerful machine) and *scaling out* (*horizontal scaling*, distributing the load across multiple smaller machines)

Distributing a load across multiple machines is also known as a *shared-nothing* architecture

A system that can run on a single machine is often simpler, but high-end machines can become very expensive, so very intensive workloads often can't avoid scaling out

In reality, good architectures usually involve a pragmatic mixture of approaches: for example, using several fairly powerful machines can still be simpler and cheaper than a large number of small virtual machines

### Elastic Systems
Some systems are elastic, meaning that they can automatically add computing resources when they detect a load increase, whereas other systems are scaled manually

An elastic system can be useful if the load is highly unpredictable, but manually scaled systems are simpler and have fewer operational expenses

While distributing stateless services across multiple machines is a fairly straightforward task, taking a stateful data system from a single node to a distributed setup can introduce a lot of additional complexity

For this reason, common wisdom until recently was to keep your database on a single node until scaling cost or high-availability requirements force you to make it distributed

*A **node** in this context is a physical or virtual machine, in a distributed system nodes work together to perform tasks like storing data, handling requests, or running computations

As the tools and abstractions for distributed systems get better, this common wisdom may change, at least from some kinds of applications

The architecture of systems that operate at large scale is usually highly specific to the application, there is no such thing as a generic, one-size-fits-all scalable architecture (aka *magic scaling sauce*)

The problem may be the volume of reads, the volume of writes, the volume of data to store, the complexity of the data, the response time requirements, the access patterns, or some mixture of all of these plus many more issues