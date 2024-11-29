# Scability
Even if a system is working reliable today, that doesn't mean it will work reliably in the future

One common reason for degradation is increased load: perhaps the system has grown from 10,000 concurrent users to 100,000 concurrent users, or from 1 million to 10 million

Perhaps it is processing much larger volumes of data then it did before

*Scalability* is the term we use to describe a system's ability to cope with increased load

Note, however that it is not a one-dimensional label that we can attach to a system

It is meaningless to say "X is scalable" or "Y doesn't scale", rather discussing scalability means consideration questions like "if the system grows in a particular way, what are options for coping with growth?" and "How can we add computing resources to handle the load"

## Describing Load
First, we need to succintly describe the current load on the system

Load can be described with a few numbers which we call *load parameters*

The best choice of parameters depends on the architecture of your system, it may be:
- Requests per second to a web server
- The ratio of reads to writes in a database
- The number of simulataneously active users in the chat room
- The hit rate on a cache

Perhaps the average case is what matters, or the bottleneck is dominated by a small number of extreme cases

