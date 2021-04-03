# The future of MMCore
Micro-Manager (https://micro-manager.org) is open-source software for operation of automated microscopes. It contains a GUI that communicates with microscope equipment through a device abstraction layer. The device abstraction layer was made so that anyone can create a "Device Adapter" to support any kind of microscopy related hardware device and make it available in the Micro-Manager application domain. Through the contributions of many people, in both academia and industry, hundreds of such Device Adapters now exist, operating a very large set of microscope hardware components. This rich resource can be used outside of the Micro-Manager GUI application and potentially facilitate operation of many custom-built microscopes, and the past 15 years has seen just that, as a wealth of plugins, scripts and other atuomation tools have been created to facilitate various types of microscopy.

However, as new types of microscopes increasingly use novel types of hardware, complex robotic automation, produce data at dramatically larger rates, the needs of modern microscopes have increasingly pushed up against the limits of what the current Micro-Manager architecture can enable. Furthermore, a decade and half of lessons have been learned about successes and failures of the current design.

The device layer, `MMCore`, is the most widely used of all the Micro-Manager components and is the source of many of the current limitations. The goal of this document is to provide a roadmap the development for successor to the Micro-Manager core. We hope for this to be a community driven effort, and feedback/discussions/contributions are welcome, from people of all backgrounds/experience levels.

In many  a familiarity with the current design of Micro-Manager is helpful (**TODO**: add link to first paper)

## How to navigate this repository

**TODO** instructions for how community contributes
- Different folders contain different subtopics
- Feel free to open issues to discuss certain topics
- Some issues we specifically want to get community contributed descriptions about current microscope types used etc


## Design principles
* **Performance**: Should be performant enough to take full advantage of hardware for acquiring/writing/reading data
* **Flexibility**: Should be able to control turnkey systems and strange custom-built systems alike
* **Modularity**: Low level API should sit on top of minimal, essential code for hardware control. Higher level code should be able to be mixed and matched as needed
* **Accessibility**: Easy to get started for the novice, easy to make powerful modifications for the expert; Not restricted to a single programming langauge
* **Backwards compatible (sometimes)**: Strive for backwards compatibily as much as possible, but not at the expense of a better future software.


## Overview 

Though there are many limitations described throughout this repository with the current `MMCore`, one major limitiation in particular underlies the main architectural changes in this reorganization: The foundations of the Micro-Manager device interface were developed in 2005 and 2006, and `MMCore` was developed implicitly around the concept of a microscope being a computer-controlled system consisting of a microscope stand (with built-in reflector changers, focus drive, etc..) equipped with cameras, stages, light sources, and various other peripherals attached. In the years since it has become apparent that this a quite limiting assumption, and a major goal of this project is to generalize many of the features that were specifically hard-coded to correspond to this 2005 Motorized microscope model. Doing so requires removing implicit assumptions corresponding to this model from `MMCore` and making a new, more general `MicroscopeModel` abstraction.

Thus, the three main components of this project are: 
1. A new and improved version of the `MMCore` called [`MMKernel`](mm_kernel)
2. A new [`MicroscopeModel`](microscope_model) module that generalizes micro-manager to many more microscope types 
3. A redesign of the [`MMDevice`](device_layer) API to better support exisitng device types, add new types, and improve performance.


<img src="overview.png" width="600">

**(TODO: insert brief description of current core architecture that goes along with this figure)**

