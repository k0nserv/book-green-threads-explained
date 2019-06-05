# Green Threads

Green threads solve a common problem in programming. You don’t want your code to block the CPU preventing the CPU from doing meaningful work. We solve this by using multitasking, which lets us suspend execution of one piece of code while we resume another and switch between “contexts”.

This is not to be confused with parallelism, while easy to confuse they are two different things. Think of it this way, green threads let’s us work smarter and more efficient and thereby using our resources more efficiently, and parallelism is like throwing more resources at the problem.

Commonly there are two ways to do this.

* Preemptive multitasking
* Non-preemptive multitasking \(or cooperative multitasking\)

#### Preemptive multitasking.

Some external scheduler stops a task and runs another before switching back. The task has nothing to say in this matter, the decision is made by "something" else \(often some sort of scheduler\). Kernels use this in operating systems, i.e. to allow you to use the UI while running the CPU to do calculations on single threaded systems. We’ll not talk about this kind of threading now, but my guess is that when you understand one paradigm you’ll have a good grasp on both.

#### Non-preemptive multitasking.

This is what we’ll talk about today. A task decides by itself when the CPU would be better off doing something else than waiting for something to happen in the current task. Commonly this is done by `yielding` control to the scheduler. A normal use case for this is to yield control when something that will block execution occurs. An example of this is IO operations. When the control is yielded a central scheduler direct the CPU to resume work on another task that is ready to actually do something else than just block.

