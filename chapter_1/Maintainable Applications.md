# Maintainability
It is well known that the majority of the cost of software is not in its development but in its ongoing maintenance (fixing bugs, keeping its systems operational, investigating failures, adapting it to a new platform, modifying it for new use cases, repaying technical debt, and adding new features)

Many people working on software systems dislike maintenance of so-called *legacy* systems, perhaps it involves fixing other people's mistakes or working with platforms that are not outdated

Every legacy system is unpleasant in its way, and so it is difficult to give general advice on dealing with them

However, we can design software in such a way that it will hopefully minimize pain during maintenance, and thus avoid creating legacy software ourselves, to this end we will pay particular attention to three design principles of software systems:
- *Operability*: Make it easy for operations teams to keep the system running smoothly
- *Simplicity*: Make it easy for new engineers to understand the system, by removing as much complexity as possible from the system
- *Evolvability*: Make it easy for engineers to make changes to the system in the future, adapting it for unanticipated use cases such as requirements change, also known as extensibility, modifiability, or plasticity

As previously with reliability and scalability, there are no easy solutions for achieving these goals, rather we will try to think about systems with operability, simplicity, and evolvability in mind