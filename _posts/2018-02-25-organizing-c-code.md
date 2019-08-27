---
layout: post
title: Organizing C Code
subtitle: ''
date: 2018-02-25 06:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: I have found a set of guidelines and principles useful when deciding how
  best to organize C modules

---
Software modules within a large embedded system must manage many concerns such as logic, data storage, state, side effects, and concurrency to name a few. Many modules are essentially mini-applications within the larger system which often utilize other components to provide a system service or implement a particular piece of functionality. These mini-applications can have quite a bit of their own complexity and must be designed and organized with testability, understandability, and maintainability in mind. I have found a set of guidelines and principles useful when deciding how best to organize C modules. As a starting point, it is helpful to identify the set of concerns that nearly all modules must deal with:

* **Type Definitions** - Defines the types the comprise the domain in which the module operates. Includes both types internal to the module and types exported to other modules within the system.
* **Core Business Logic** - Comprises the core of the module and consists of _algorithmic_ components that simply compute results from data and _policy_ components that make decisions about what actions the module will perform.
* **State** - Consists of any data managed by the module that changes over time
* **Concurrency** - Often closely related to state, consists of managing multi-threaded access to the module's state and functions
* **Side Effects** - Consists of actions taken by the module such as writing to the network, rendering to the UI, controlling device settings, etc.

For instance, suppose we have a module that monitors the set of remote devices active on the local area network and controls the settings of those devices based on which other devices are also active on the network (e.g. ensuring only one motor controller is engaged even if there are multiple such motor controllers on the network). This module must also provide the information about the set of active devices to other modules, which may be executing on separate threads, within the same system. In this hypothetical device management module, the concerns are

* **Type Definitions** - Definitions of types to represent remote devices and their settings
* **Core Business Logic** - Logic for deciding which settings to apply to the remote devices based the set of devices active on the network
* **State** - The set of devices active on the network and their current settings
* **Concurrency** - Providing safe access to the device state information to other threads running within the same system
* **Side Effects** - Making network requests to configure the settings of remote devices, reading network messages to discover which devices are active

## The Conglomeration Approach: Mixing Concerns

To the outside world, we want our module to expose a high-level interface that encapsulates all of these concerns together. Often, it may seem that the most straightforward approach would be to jumble all of the concerns together within a single source file that implements the public API exported to other modules. In this conglomeration, business logic is contained within callback functions invoked from the device discovery library and is intermixed with network calls to remote devices that configure the settings of those devices. Our single source file may look something like this:

{% gist 6aaf5efa76bde8c74d10cc8913790cbf %}

This singe source file mixes all of the concerns it must manage together. From a high-level, this singe source file looks like a jumble of multiple concerns.

![](/images/organizing-c-code/Conglomeration.svg)

Although this approach may seem like the most straightforward initially, intermixing all the module's concerns together introduces many challenges.

**Testability is inhibited** when the core business logic is mixed in with code that performs side effects and manages state and concurrency. Since the core logic cannot be isolated and exercised in unit tests, the only way to test this module is to perform end-to-end systems test of the entire module. These types of tests tend to be much more complex, less reliable, less maintainable, and more difficult to configure and set up. In this case, to test our module, we would need to set up a real network with real devices that we must somehow be able to make come and go off the network. We must also be able to observe their settings changes based on what our module tells the remote devices to do. Such a test would be very difficult to automated reliably which means our module will most likely only be exercised through manual testing.

**Understandability and maintainability is reduced**. To make a change, add a feature, or fix a bug, the developer must fully understand the entirety of the module when all the concerns it manages are jumbled together. With the logic mixed in with the state and concurrency management, updating the core logic requires understanding all possible side effects that logic may have also been performing (mutating state, sending network messages, etc.) as well as ensuring all concurrent access is managed appropriately.

**The logic is less reusable** when it is intermixed with code that calls to send network requests and is invoked by the service discovery library we are using. If we wanted to use a different service discovery library or configure remote device settings using another mechanism (e.g. HTTP or our own custom protocol), then we would have to make significant changes to the module and risk introducing regressions to the business logic. We likely only have manual testing at our disposal for catching regressions and validating changes. Porting the business logic to run on a different device that may not support the same device discovery and network protocols used in the initial implementation also becomes very difficult. Doing so will often require the extensive use of compiler flags throughout the code which _greatly_ reduce readability, understandability, and maintainability of the module.

## Splitting Concerns

The key principle that I have found most useful to structuring the C modules I write is to recognize that these concerns the module must manage should be split apart and treated separately. Furthermore, there is a strict dependency hierarchy among the different concerns that should be respected to ensure that the core essence of the module does not depend on specific lower level implementation details that are somewhat incidental to the core responsibilities of the module. Ideally, I like to structure modules using the following general organization

![](/images/organizing-c-code/SplitConcerns.svg)

The most essential concern of any module, which has implications for the entire module, are the **type definitions**. Therefore, at the lowest-level core of the module, I usually place the type definitions, and, since this is C, this also includes all low-level functions necessary for working with types--initialization, destruction, equality comparisons, string representations, list manipulation, etc.--as needed by the module. I often use a separate header and source file for the type definitions and associated type functions.

The next most essential concern of a module is the **core business logic**. The core business logic should depend only on the type definitions and nothing else in the module. We want the core logic to be (1) easily testable, and (2) independent of peripheral details, such as network protocols used, about how the module may perform some of its actions. Therefore, the core business logic should consist entirely of _pure functions_ that operate solely on data using types defined in the module's type definitions or public types exported by other modules in the system. The core business logic should perform no side effects, should be completely stateless, and should not need to worry about concurrency at all. I often use a separate source and header file to contain all core business logic, and if the logic is large and complex enough I will even split it among additional smaller source and header files. There are several reasons for separating the core business logic into its own source and header file(s)

* The business logic is clearly separated from the other concerns
* Enforcing that functions in the core business logic are stateless becomes simpler, simply do not allow any static mutable variables that could contain state in those files
* Unit tests can easily target the core business logic in isolation by executing against the functions defined in the core logic's header file(s)
* The core business logic is more re-usable and portable because it is all isolated within a single compilation unit

Finally, at the top of the dependency tree are the components for managing **state**, **side-effects**, and **concurrency**. When these concerns become large and complex enough, I will split them into separate files to keep things understandable. At the top, I will place a "controller" component that manages the control flow and interaction between the components; the controller will delegate to the core logic to make the complex policy decisions and will get its own hands dirty performing side effects and mutating state. Finally, the public API exported to other parts of the system will access the module's functionality through the controller. If these components are small enough, then I will often roll them up together into a single file. The important point is that the concerns dealt with in this file are kept out of the business logic which is kept pure.

Because the top layer deals with concerns such as state and side effects, it is difficult to test without resorting to end-to-end systems tests. However, since most of the business logic has been moved out of the top layer and into its own component, the top layer is usually not very complex. That complexity has been pushed down into the core logic component which is easy to test. The end result of this structure is that the most complex code is also the easiest to test, and the code that is the most difficult to test is also the simplest.

This organization is essentially just applying the dependency inversion principle and the [clean architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html) described by Bob Martin to C module design. Indeed, if we draw the module diagram using circles instead of boxes and arrows, we can see that it really is just a manifestation of the clean architecture.

![](/images/organizing-c-code/CleanArchitecture.svg)

## Refactoring the Device Manager Module

Now we can apply these principles to refactor our hypothetical device management module. The first step is to move our type definitions and associated type functions into a separate header and source file, call it `device_config_types.h` and `device_config_types.c`

{% gist 59f78c1117841da3d1828a3cc3fce110 %}

Next, we need to define the most important component of the module, the core business logic. We will put this into its own source and header file, called `device_config_updater.h` and `device_config_updater.c` (I take this naming convention from [Elm](http://elm-lang.org/) because this component usually plays the same role as the `update` function in the [Elm Architecture](https://guide.elm-lang.org/architecture/)). Because we want only _pure_ functions that are stateless and perform no side-effects in this component, any state must be passed in as parameters and any side-effects to be performed must be returned as data structures which simply _describe_ the desired side effects to be performed (again, this is also taken from the Elm architecture with its concept of [commands](https://guide.elm-lang.org/architecture/effects/)). Because this is C, and we do not have support for persistent data structures like [other](https://clojure.org/reference/data_structures) [languages](https://msdn.microsoft.com/en-us/magazine/mt795189.aspx) [do](https://facebook.github.io/immutable-js/), we will cheat a bit and let our pure functions mutate the state parameter passed in to move the application to the next state. Regardless, all functions in our logic component are easily and fully unit-testable.

{% gist 939a8fe29607189a8083b3e75140931d %}

Finally, we will tie everything together in a single, top-level component that manages the state, concurrency, and side-effect concerns of the module, delegating to the logic component to perform most of the complex computations. We will call this top level component `device_config_controller.h` and `device_config_controller.c`.

{% gist 92751eabdf3d9e3c191b6afe8b5d6f3c %}

## Summary

The key guideline I have found most useful for organizing C modules is to keep concerns separate rather than jumbling them all together. In particular, it is extremely important to keep type definitions and core business logic pure and separate so that it is more testable, understandable, maintainable, and reusable. Keep the ugly stuff--state, side-effects, and concurrency--at the peripheral of the module and leave the core beautiful and pure.