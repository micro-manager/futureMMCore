## The Current State of MMCore
The current version of MMCore leaves issues of efficient thread usage and thread safety entirely up to device adapter implementations and to the higher level Micro-Manager applications that
interface with MMCore. The core may be called upon from multiple threads simultaneously and any resulting calls to a device adapter will take place on the same thread as the
original caller. While this provides for a great deal of flexibility in how device adapters and Micro-Manager applications are written it places a burden on developers
to make sure that their software will not fail due to race conditions and other confusion situations that can arise in a multithreaded application. 

### Thread Safety
A device adapter that has functioned reliably for years may suddenly begin to exhibit confusing error conditions due to a minor change to the threading model used by the 
top-level application. In order to prevent simultaneous access to a sensitive section of code in a device adapter a [thread lock](https://en.wikipedia.org/wiki/Lock_(computer_science))
must be used. MMCore provides a simple way to use a thread lock in the form of `MMThreadGuard`, simply creating an instance of `MMThreadGuard` will block access to the
device adapter from other threads until the instance is deleted. Unfortunately since this feature is largely unknown many device adapters do not make use of it. 

### Efficiency
Since MMCore operates primarily on the same thread that as the higher-level application that calls it there are many cases where a function call in to MMCore resulting in device I/O
will block until the device I/O operation is completed. While this is not noticeable for many modern devices it can become an issue when there are a large number of devices being managed
or when older devices using a slow communication protocol are used. This is especially noticeable when the "device property cache" is refreshed, each device being managed by
MMCore is sequentially queried for all of its devices. Since each device generally doesn't share any hardware resources with other devices this sequential mode of operation is needlessly
slow.

## Simplifying Development While Increasing Efficiency
MMKernel should aim to lessen the burden of writing thread safe code while also improving device I/O performance. One solution to these issues could be to use an "Event Loop Model"
similar to that used by Node.js. Rather than directly calling device adapter methods from the main thread MMKernel should maintain a pool of worker threads with one thread per loaded
device adapter. Incoming requests from high-level applications will be processed by the event loop and tasks will be delegated to the appropriate worker thread. Since each
device adapter runs in only a single thread there would be no need for device adapters to take thread safety into consideration. Operations such as the refreshing of the "device
property cache" could be expected to run many times more quickly as all devices would be requesting property values from their respective hardware simultaneously.

### Greater Efficiency with an Asynchronous API
The addition of an asynchronous API for the getting/setting of device properties would allow even single-threaded high-level applications to make use of multi-threaded device I/O.
For example, a device property could be requested immediately without any device I/O delay. The actual retrieve the property value could be retrieved later once the device I/O has completed.
