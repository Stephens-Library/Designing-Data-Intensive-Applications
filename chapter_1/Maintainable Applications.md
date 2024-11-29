# Maintainability
It is well known that the majority of the cost of software is not in its development but in its ongoing maintenance (fixing bugs, keeping its systems operational, investigating failures, adapting it to a new platform, modifying it for new use cases, repaying technical debt, and adding new features)

Many people working on software systems dislike maintenance of so-called *legacy* systems, perhaps it involves fixing other people's mistakes or working with platforms that are not outdated

Every legacy system is unpleasant in its way, so it is difficult to give general advice on dealing with them

However, we can design software in such a way that it will hopefully minimize pain during maintenance, and thus avoid creating legacy software ourselves, to this end we will pay particular attention to three design principles of software systems:
- *Operability*: Make it easy for operations teams to keep the system running smoothly
- *Simplicity*: Make it easy for new engineers to understand the system, by removing as much complexity as possible from the system
- *Evolvability*: Make it easy for engineers to make changes to the system in the future, adapting it for unanticipated use cases such as requirements change, also known as extensibility, modifiability, or plasticity

As previously with reliability and scalability, there are no easy solutions for achieving these goals, rather we will try to think about systems with operability, simplicity, and evolvability in mind

## Operability: Making Life Easy for Operations
It has been suggested that "good operations can often work around the limitations of bad software, but good software cannot run reliably with bad operations"

While some aspects of operations can and should be automated, it is still up to humans to set up that automation in the first place and to make sure it's working correctly

Operations teams are vital to keeping a software system running smoothly. A good operations team typically is reasonable for the following and more:
- Monitoring the health of the system and quickly restoring service if it goes into a bad state
- Tracking down the cause of problems such as system failures or degraded performance
- Keeping software and platforms up to date, including security patches
- Keeping tabs on how different systems affect each other, so that a problematic change can be avoided before it causes damage
- Anticipating future problems and solving them before they occur
- Establishing good practices and tools for deployment, configuration management and more
- Performing complex maintenance tasks, such as moving an application from one platform to another
- Maintaining the security of the system as configuration changes are made
- Defining processes that make operations predictable and help keep the production environment stable
- Preserving the organization's knowledge about the system, even as individual people come and go

Good operability means making routine tasks easy, allowing the operations team to focus their efforts on high-value activities. Data systems can do various things to make routine tasks easy, including:
- Providing visibility into the runtime behaviour and internals of the system with good monitoring
- Providing good support for automation and integration with standard tools
- Avoiding dependency on individual machines
- Providing good documentation and an easy-to-understand operational model
- Providing good default behaviour, but also giving the adminstrators the freedom to override defaults when needed
- Self-healing where appropriate, but also giving adminstrators manual control over the system state when needed
- Exhibiting predictable behaviour, minimizing surprises

## Simplicity: Managing Complexity
A software project mired in complexity is sometimes described as a *big ball of mud*

There are various possible symptoms of complexity: explosion of the state space, tight coupling of modules, tangled dependencies, inconsistent naming and terminology, hacks aimed as solving performance problems, special-casing to work around issues elsewhere, and many more

Making a system simpler does not necessarily mean reducing functionality, it can also mean removing *accidental* complexity

One of the best tools for removing accidental complexity is *abstraction*, a good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand facade

Not only is this reuse more efficient than reimplementing a similar thing multiple times, it also leads to higher-quality software, as quality improvements in the abstracted component benefit all applications that use it

For example, high-level programming languages are abstractions that hide machine code, CPU registers, and syscalls

SQL is an abstraction that hides complex on-disk and in-memory data structures, concurrent requests from other clients, and inconsistencies after crashes

Of course, when programming in a high-level language, we are still using machine code, we are just not using it directly, because the programming language abstraction saves us from having to think about it