Aaron's Pithy Thoughts on Architecture
----
Partial mind-dump, partial evolving document, the goal of which is to filter the noise and focus on what actually matters when it comes to architecture.

> **Caveat:** You are responsible for your own education.  No single guide will make you good.  This guide exists to consolidate and convey the concepts that I believe are most imporant to setting you on a solid foundation.

## The Big Picture

### Modularization, Encapsulation
You must think of your system as a set of interacting components.  For each component, you must decide:
- Roles
- Responsibility
- Interface

This requires a lot of up front thought.  The idea is to break the system down into its main responsibilities and encapsulate those responsibilities.  For each module, you need to be asking:
- What is this module's purpose?
- What problem does it solve?
- What is the API that properly abstracts and encapsulates this behavior?

There is no hard and fast rule on how to break down responsibilities or what level of abstraction to apply.  It is to some degree at the developers discretion and may be driven by domain- or device-specific considerations.

The main things to watch out for, both in design and implementation, are violations of encapsulation or bleeding of responsibilities.  A violation of encapsulation occurs when you don't strictly stick to your module's API.  If you find yourself going around the pure, high-level abstraction, you need to ask yourself why.  Perhaps the abstraction is incorrect.  Or perhaps there's some compromise in terms of efficiency or power consumption that you need to think carefully about.

Bleeidng of responsibilities occurs when a module is doing extra work that is, strictly speaking, outside of its core responsibiltiy.  Again, this may be a sign that you need to reevaluate your work breakdown or your interfaces between modules.

> In the early stages of a project, always be willing to reassess your architecture and modify as necessary.  The further along you get, the more locked in your architecture becomes.

### Directed Acyclicity
I always imagine the components of a system as being part of a hierarchy.  Examples of modules that exist on the bottom of this hierarchy would be a circular buffer or a SLIP encoder/decoder for identifying packet boundaries across a stream interface.  Such modules should be aware of no other components and expose interfaces that allow only for the modification, storage, or retrieval of data.

**What would a cyclical structure look like?**

One example would be a BLE manager and a command interpreter having mutual awareness of each other.  Let's say the BLE manager passes all packets coming in on a certain characteristic to the command interpreter.  Let's say also that the command interpreter passes all BLE related commands to the BLE manager.  Both modules have direct awareness of each other.  This is cyclical and a violation of the principle of directed acyclicity.

**What do?**  
What follows here are my thoughts on resolving this particular situation.  Others may disagree.

A command arriving in the system is an event.  The command intepreter should be registered to handle command events.  The interface receiving command events does not need to know anything about how they're handled, or even who's handling them.  It just needs to issue the event to the system.

The command interpreter, on the other hand, is potentially touching a lot of areas of the system.  This implies it is high up in the directed acyclic graph.  Maybe the command interpreter calls other modules directly.  Or maybe these are also events.  For example, maybe the BLE manager is registered to handle BLE command events.  These are the kind of things you need to think about.

#### Use Callbacks (or Generate Events)
Another area where it is common to violate acyclicity is where the lower-level module needs to provide some signal back to the higher-level module.  If there is only one higher-level dependent, it may be tempting to have the lower-level module call the higher-level one directly.  This creates cyclical awareness.  Don't do this.

Design the lower-level modules such that callbacks can be registered for events.  Or, alternatively, design them to generate events to which higher-level modules can subscribe.

Consider the `nrfx_uart` driver in the Nordic SDK.  A high level module can initiate an RX or TX operation using functions like `nrfx_uart_tx()`.  But ... it gets notified of completion via RX_DONE and TX_DONE events.  The nrf driver knows nothing about the clients using it.

> One good question to ask yourself when designing the lower-level layers is, "How easily could I pull this out and use it in another project?"  If the answer is "not easily," that is a sign of tight coupling and poor design.

## Module Design
It is important to think non-linearly.  I tend to think of the modules comprising a system as a set of independent agents acting in concert, all of their own volition.  They each take care of their own aspect of the system and they all do so simultaneously.  But ... how do you accomplish this.  Fundamentally, you need a concurrency model.

### Concurrency Model
You might be thinking, "Ok, this is either RTOS or big-loop," but you need to think deeper than this.  The way I think about system modules is very akin to "Active Objects", although I am looser in my thinking around it.  The [formal definition](https://en.wikipedia.org/wiki/Active_object) calls for each object to have its own thread and thread stack.  However, this is not strictly necessary.  With a good understanding of concurrency design patterns and the appropriate library components (such as a message queue or Nordic's `app_scheduler`), you can implement an RTOS-less version of active objects.

I find that I tend to think in terms of the Concurrency Design Patterns discussed in Douglass' [_Real-Time Design Patterns_](https://www.goodreads.com/book/show/179103.Real_Time_Design_Patterns).  I would recommend becoming familiar with:
- Message Queueing Pattern
- Interrupt Pattern
- Cyclic Executive Pattern

And looking into:
- Guarded Call Pattern
- Rendezvoud Pattern

This really comes down to how you think about systems and the interaction between modules and being able to make judgment calls as you feel appropriate.  Simply thinking RTOS vs non-RTOS is not going to get you to good design.  You can have terrible RTOS-based designs and highly asynchronous RTOS-less designs capable of meeting hard real-time requirements.

Also, I should point out that design patterns are only half the battle in embedded design.  You need to be leveraging timers, DMAs, and inter-peripheral signaling.  Your goal is to offload the CPU as much as possible and to never tie it up.  If you do need to perform long-running computation (GPS calculations, FFTs, etc.), it should be part of an interruptable, low-priority background thread that can provide a signal when results are ready.

### Active Objects
My thought process and conception around active objects closely mirrors Miro Samek's.  For a concise overview of Samek's principles, please listen to [embedded.fm episode 401](https://embedded.fm/episodes/401) between the 23:00 and 26:00 mark.

Note that I disagree with Samek on the necessity of using an RTOS.  Whereas Samek describes active objects as "a way to use RTOSes safely", I see active objects as a broader concept that is also compatible with the Cyclic Executive Pattern.

The fundamentals I follow are:
1. Each module is designed around an event-driven state machine.
2. Handling of events do not block waiting for completion.  If an operation is started (such as BLE connection) where you need to act on the result of that operation, that action should be triggered by a seprate event (e.g. `CONN_SUCCESS` or `SCAN_TIMEOUT`).  In the meantime, the module is in a connecting state waiting for the next message.
3. Event handling is serialized, either on its own thread or on the main loop for RTOS-less systems

Point number 3 is where I think the idea comes in that each object _must_ have its own thread.  As Samek describes, each thread has its own `while(1)` with its only point of blocking being that thread's message queue.  You can, however, achieve something that is functionally equivalent using the Cyclic Executive Pattern and a scheduler module (e.g. the Nordic SDK's `app_scheduler`).

### Flexibility
The overall theme here is a solid understanding of core principles combined with flexibility in approach.  I try not to be too dogmatic or unilateral in my designs.

## Visual Modeling
Visual modeling is _key_.  I always create diagrmas to do the heavy lifting of architectural documentation.  One or two diagrams and a couple well-written paragraphs should be enough to gain a high-level understanding of any module.  If not, the module is too complex.  Think of it like a resumé.  One page is best.  Two is acceptable.

Do not describe design details or API in your architecture doc.  API and usage deatils come from the doxygen.  Design details are in the code comments.

The documentation should be agile, adaptive, and catered to each individual module.  I eschew formal documentation as it tends to impose top-heavy processes while simultaneously inhibiting natural, efficient, and minimal description of each module.  I believe in the principle of keeping documentation close to the source and try to follow [Koopman's guidelines](http://www.koopman.us/) on "just the right amound of documentation."

The diagrams I find most useful are:
- **State diagrams:** A state diagram should be created for all modules that are active objects.  The diagram should exactly match the code.  I use these to drive TDD.  I also translate the diagram into statecharts (e.g. a matrix of states vs events) so that all sneak paths are apparent.  The state chart should indicate what happens for all possible sneak paths and unit tests should be developed to cover those cases.
- **Sequence diagrams:** These show the control flow of an application.  They visualize how signals propogate through a system.  Sequence diagrams may exist as explanatory aids to invididual modules or for higher-level user scenarios.
- **Data flow diagrams:** It's important to know how the data flows through your program: where it's being transformed and where it's being handed off.  This is different than the sequence diagram, which shows control flow.  Both the control and the data flow must be understood simultaneously.

> A _sneak path_ is a state transition that is not formally covered by the state diagram.  It happens when an event occurs in a state for which it is not expected.  If not handled explicitly, this may result in undefined behavior, swallowed events or unexpected state transitions.

One of the worst "hangs" an embedded system may experience is a "stuck" state machine.  E.g. a "connecting" state for which the "failed" event has already passed, but for which there is no timeout.  The "failed" event may have been swallowed by a previous state due to an incomplete understanding of the operation of the BLE stack.  A design flaw has now become an intermittent "bricking" issue for end users.  The most straightforward thing to do with sneak paths, at least during development, is to ASSERT loudly.  As the product evolves towards production release, the focus should move towards logging and/or graceful recovery.

> From what I can tell, data flow diagrams do not appear to be part of official UML, which is sad because, at the end of the day, moving and processing data is the fundamental paradigm that describes all software.  For this reason, I would steer clear of any modeling software that is UML only, such as PlantUML.

I will occasionally also use Activity Diagrams (i.e. flowcharts) and Class Diagrams, but in my view they are not as useful and I tend to use them sparingly.

## Data Managment
If you have one take-away from this section, it should be the concept of _ownership_.  All data _must_ have an owner.  Call it the "single ownership principle".

Now, Samek says you need to have separate data structures for message pssing between modules.  I somewhat disagree.  The primary point is that no data structure should ever have simultaneous ownership.  Let's take an example.

Let's go back to the command interperter described earlier.  Let's say the interpreter receives packets that have come in on some kind of interface (we don't care what) and have already been decoded (we don't care how).  Whatever did the decoding (`nanopb`?, `MsgPack`?) has placed the decoded packet in some kind of allocation unit from some kind of pool (heap?, block pool?, byte pool?).  This is the object the interpreter receives.

Now, all the interperter cares about is being able to see the data and being able to free the memory resource once its done.  This implies that once the data object is handed off, the interpreter becomes the owner.  But, prior to that, the decoder may be the owner.  The decoder may be processing a data stream and requesting allocation from the pool whenever it has a fully decoded packet ready.  It can then attach the allocated packet to a data ready event.  If we're really following OO, then the data object itself would have `GetData()` and `Free()` methods that the interpeter (or any other consumer of the packet) could use.

When Samek speaks of "special" data structures for the inter-module asynchronous communication, he means things like a thread's message queue, or the event structures of a [pub-sub event system](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/libraries/others/event_manager.html).  That is to say the _signaling_ between threads (or objects) must be of special design, but I don't see why you can't hand off ownership of normal data objects as part of an event.

## S.O.L.I.D.
One final thing to touch on when it comes to architecture are the [principles of SOLID](https://devopedia.org/solid-design-principles).  Study them.  Learn them.  Review them regularly.  There are many resources out there to help you understand and incorporate them into your thinking.

Let's go over each of the five principles continuing the same example.  We have a stream interface from which data is being packetized and then those packets are being decoded.  Decoded packets are being sent to the command interperter.

- **Single Responsibility Principle:** The stream interface is not the same thing that separates the stream into packets.  The packetizer is not the same thing as the decoder.  The decoder is not the same thing as the interperter.  In our case, the packetizer might be `slip.c` from the Nordic SDK and the decoder might be `nanopb`.  Also, the stream transport might be TCP, BLE, UART, or USB.  That aspect should be abstracted away.
- **Open/Closed Principle:** Open for extension means you can add functionality.  E.g. I could add new commands to the command interpreter.  Closed for modification means no breaking changes.  I cannot alter the stream protocol (or the interpreter) in any way that breaks existing functionality.
- **Liskov Substitution Principle:** Going back to the transport, it may be TCP/IP, but I should be able to easily swap that out.  Maybe we want the same packets to come directly from the cloud (TCP/IP) or from the app (BLE).  Luckily, we followed Liskov's example and created an abstract interface for the transport managers to implement.
- **Interface Segregation Principle:** This is about not muddying interfaces with more than a client needs to know.  For example, the data object passed off to the command interpreter has a means to read the data and a means to free the block.  That's it.
- **Dependency Inversion Principle:** In other words, code to the interface.  The interface _is_ your contract.  In the directed acyclic graph of modules I discussed above, this means that both a module and its client(s) code to the interface.  Objects are interchangeable both above and below that interface.  All modules must abide by thier contracts.  This is the foundation that makes TDD possible.  Without this, you will have a very hard time with TDD.

Bear in mind that S.O.L.I.D. is a set of guidelines.  It's more important to understand their rationale than to be dogmatic about them.  You don't need to declare an abstract interfact for every module.  On the other hand, some modules may need to implement multiple interfaces.  Use discretion and work more towards the spirit of the law rather than the letter.  You don't need C++ to implement OO principles, though well-designed C++ can certainly help with code clarity.  Again, it's about flexibility.

## A Note About TDD
With your module breakdown and their interfaces all figured out, it should be easy to apply test-driven development.  The test framework interacts with the module under development at its public API and the test harness mocks the APIs of the module's dependencies.  This, to me, is the proper conception of TDD and unit-testing (i.e. it's an architecture first approach).  Each test executable tests one "unit" where the unit is the module under test.  The test functions exercise the public API with the goal of achieving code, branch, and state-machine coverage.

Note that if the module under test relies on interfaces or abstraction layers for its dependencies, it is easier to apply a mocking framework.  Again, though, I do not recommend dogmatically following such a practice.  It doesn't make sense to try abstracting everything, and there are other methods of getting a mocking framework to work well.  Flexibility and discretion in all things.

-----
Copyright &copy; 2022 Aaron Fontaine
