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