# Reliability
Everybody has an intuitive idea of what it means for something to be reliable or unreliable. For software, typical expectations include:
- The application performs the function that the user expected
- It can tolerate the user making mistakes or using the software in unexpected ways
- Its performance is good enough for the required use case, under the expected load and data volume
- The system prevents any unauthorized access and abuse

Things that can go wrong are *faults*, and systems that anticipate faults can cope with them, are called *fault-tolerant* or *resilient*

Note that a fault is different from a failure. A fault is usually defined as one component of the system deviating from its spec, whereas a *failure* is when the system as a whole stops providing the required service to the user

It is impossible to reduce the probability of a fault to zero, therefore it is usually best to design fault-tolerance mechanisms that prevent faults from causing failures

Counterintuitively, in such fault-tolerance systems, it can make sense to increase the rate of faults by triggering them deliberately, like killing random processes without warning, this can ensure that the fault-tolerance machinery is continually exercised and tested

Though we generally prefer fault tolerance over fault prevention, there are cases where prevention is better than the cure, because no cure exists, such as if an attacker has compromised the system, however, we will mostly talk about faults that can be cured.

## Hardware Faults
In a large data center, hardware faults happen all the time
- Hard disks crash
- RAM becomes faulty
The power grid has a blackout
- Someone unplugs the wrong network cable

Hard disks are reported as having a mean time to failure (MTTF) of about 10 to 50 years, thus on a storage cluster with 10,000 disks we should expect on average one disk to die per day

### Responses to Hardware Faults
Our first response is usually to add redundancy to the individual hardware components to reduce failure rate, disks may be set up in a RAID configuration, servers may have dual power supplies and hot-swappable CPUs, and data centers may have batteries and diesel generators for backup powers

Until recently, redundancy of hardware components was sufficient, as long as a backup could be restored onto a new machine fairly quickly, the downtime in case of failure was fine in most applications

However, as data volumes and applications computer demands have increased, more applications have begun using a larger number of machines, which proportionally increases the rate of hardware faults

Moreover, in some cloud platforms such as AWS, it is fairly common for virtual machine instances to become unavailable without warning, as the platforms are designed to prioritize flexibility and elasticity over single-machine reliability

Hence there is a move towards a system that can tolerate the loss of entire machines by using software fault-tolerance techniques in addition to hardware redundancy
- A single-server system requires planned downtime if you need to reboot the machine
- A system that can tolerate machine failure can be patched one node at a time, without downtime of the entire system

## Software Errors
We usually think of hardware faults as being random and independent from each other: one machine's disk failing does not imply that another machine's disk is going to fail, there may be weeak correlations (common causes such as temperature), but otherwise it's very unlikely that a large number of hardware components fail at the same time

Another class of fault is a systematic error within the system, such faults are harder to anticipate because they are correlated across nodes, they tend to cause many more system failures, this inclues:
- Software bugs that cause every instance of an application server to crash when given a particularly bad input
- A runaway process that uses up some shared resource (CPU time, memory, disk space, network bandwidth)
- Cascading failures, where a small fault in one component triggers a fault in another component, which trigger further faults

The bugs that cause these software faults lie dormant for a long time until they are triggered by an unusual set of circumstances

There is no quick solution to the problem of systematic faults in software, thought lots of small things can help such as carefully thinking about assumptions and interactions in the system, thorough testing, process isolation, and allowing process to crash and restart

## Human Errors
Human design and build software systems, and the operators who keep the systems running are also human, how do we ake our systems reliable, in spite of numerable unreliable humans? The best system combine several approaches
- Design systems in a way that minimizes opportunities for error, for example well-designed abstractions, APIs, and admin interfaces make it easy to do "the right thing" and discourage "the wrong thing"
- Decoupel the places where people make the most mistakes from the places where they can cause failures
- Test thoroughly at all levels, from unit tests to whole-system integration tests and manual tests
- Allow quick and easy recovery from human errors, to minimize the impact in the case of a failure
- Set up detailed and clear monitoring such as performance metrics and error rates, in other engineering disciplines this is referred to as "telemetry"
- Implement good management practices and training

## How Important is Reliability?
Reliability is not just for nuclear power stations and air traffic control software

More mundane applications are also expected to work reliably, bugs in business applications cause lost productivity and outages of ecommerce sites can have huge costs in terms of lost revenue and reputation damage

There are situations in which we may choose to sacrifice reliability in order to reduce development costs (e.g when developing a prototype product for an unprovement market) or operational costs, but we should be very conscious when cutting corners
