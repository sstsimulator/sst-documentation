# Level 3 Tutorial

## Table of Contents

1. [Developing extensions to SST](#DevelopingExtensions)
2. [Developing Custom SST components](#CustomSSTComponents)
3. [Developing SST Subcomponents](#DevelopingSubcomponents)
4. [Connecting Components with Ports](#ConnectingComponents)
5. [Integration Testing, Integration with PyBind](#IntegrationTesting)

## Developing Extensions to SST <a name="DevelopingExtensions"></a>

### Introduction and Environment

The SST infrastructure provides the ability to develop and utilize 
component models outside of those provided by the SST-Elements package.  
SST provides a set of classes for external models to inherit in order 
to ensure basic interoperability with the SST Core and adjacent models.  
Externally developed components may implement the core functionality 
required to model the desired behavior as long as the core SST 
interfaces are utilized to interact with the SST simulation environment.  

The process by which to develop external component models is as follows:
1. Outline the directory and file structure for the external component
2. Implement the simulation component using the interfaces provided by SST
3. Build the component
4. Register the component with the SST Core
5. Integrate the component into a simulation script
6. Execute the simulation

Using this development workflow, there are a number of key items to keep 
in mind when implementing external components.

* C++ Standards: The SST Core utilizes C++11 data structures and methods.  External 
elements should follow the C++11 standard as well.
* All external elements should implement and register a method that serves as a 
clock advancement method (clock tick).  This function will be registered and 
call with the SST Core in order to advance the internal clock
* All warning, debug and user messages should utilize the SST messaging 
infrastructure.  This ensures that messages are correctly delivered to the 
console when executing parallel simulations with threading and/or MPI.
* External components *should not* attempt to access MPI communicators directly.  
The SST Core will manage all the event communication and component to component 
communication.
* External components *should not* attempt to access C++11 threading primitives 
directly.  The SST Core will manage all the thread state, synchronization and 
atomicity of message state.
* The SST Core applications (`sst`, `sst-config`, `sst-register`, etc) must be 
in the user's current PATH environment variable in order to correctly build 
and install the external component.

### Directory Structure

The first step in developing an external SST component is to create
the directory tree structure for our sample component.  For the remainder 
of this tutorial, we will build an external component called 
`basicComponent`.  We will utilize this example to build increasingly 
complex external components.  For this component, we create a top-level 
directory called `basicComponent` that contains a top-level makefile 
and a text README file.  We also create a `src` directory that contains 
a makefile as well as the C++ implementation and header files for the 
component's implementation.

```
basicComponent
│   README.md
│   Makefile
│
└───src
    │   Makefile
    │   basicComponent.cc
    │   basicComponent.h
    │   ...
```

### Build Environment

Now that we have a basic directory infrastructure for our external 
component, we can start constructing the build environment.  As we saw 
above, we have two makefiles.  The top-level makefile drives one or more 
child makefiles that reside in the component subdirectories.  An example 
of the top level makefile (`basicComponent/Makefile`) is shown below.

```
#
# Makefile
#

.PHONY: src

all: src
debug:
        $(MAKE) debug -C src
src:
        $(MAKE) -C src
install:
        $(MAKE) -C src install
clean:
        $(MAKE) -C src clean

```

The child makefile residing in the `src` directory 
handles the majority of the build and installation logic.  Note that 
this makefile requires that the `sst-config` and `sst-register` executables 
are in the user's current PATH.  The `sst-config` application is utilized 
by the makefile to derive the required compiler and linker flags to build 
the external component against the respective SST Core installation.  
The `sst-register` command is utilized to "register" the new external 
component wit the SST Core installation.  This provides SST the essential 
details to load, query and execute the external component with the SST Core 
and within tools such as `sst-info`.

```
#
# src/Makefile
#


ifeq (, $(shell which sst-config))
 $(error "No sst-config in $(PATH), add `sst-config` to your PATH")
endif

CXX=$(shell sst-config --CXX)
CXXFLAGS=$(shell sst-config --ELEMENT_CXXFLAGS)
LDFLAGS=$(shell sst-config --ELEMENT_LDFLAGS)
CPPFLAGS=-I./
OPTIMIZE_FLAGS=-O2

COMPONENT_LIB=libbasicComponent.so

BASICCOMPONENT_SOURCES := $(wildcard *.cc)

BASICCOMPONENT_HEADERS := $(wildcard *.h)

BASICCOMPONENT_OBJS := $(patsubst %.cc,%.o,$(wildcard *.cc))

all: $(COMPONENT_LIB)

debug: CXXFLAGS += -DDEBUG -g
debug: $(COMPONENT_LIB)

$(COMPONENT_LIB): $(BASICCOMPONENT_OBJS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) $(LDFLAGS) -o $@ *.o
%.o:%.cc $(SIMPLEEXAMPLE_HEADERS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) -c $<
install: $(COMPONENT_LIB)
        sst-register basicComponent basicComponent_LIBDIR=$(CURDIR)
clean:
        rm -Rf *.o $(COMPONENT_LIB)
```

## Developing Custom SST Components <a name="CustomSSTComponents"></a>

### Component Class Definition

As a first example, we will create a simple component that has several key 
features:

* Inherits the base SST Core component infrastructure
* Reads parameters from an SST simulation configuration script
* Implements a simple `clock` handler that is called from the SST Core
* Prints console messages at a standard candence.

#### Class Definition

The first step in defining an external component is to create 
it's constituent header file.  There are a number of key items that 
must be defined in this header.  First, we must include the required 
SST Core header files.  These basic headers provide the base SST classes, 
template methods and ELI macros utilized to define an external component.  
This is the minimum set of headers required to construct an external component.

Next, we define the component's C++ namespaces and main class definition.
The outer namespace is the `SST` namespace to include all the required 
classes and methods provided by the SST Core infrastructure.  The inner 
namespace is our external component, `basicComponent`.  Within the 
`basicComponent` namespace we can define our initial component class.  
The `basicClock` class inherits the `SST::Component` base class.  This base 
class provides all the necessary functionality sufficient to intiaalize the 
component, read component parameters, register the clock handlers and 
interact with the core simulation methods.  The base class requires 
that we define a constructor method that receives the component ID as 
defined by the SST Core and an `SST::Params` object that contains 
all the parameters defined for the target component object in the 
SST simulation configuration.  We also define a basic destructor method.

For the private data, the implementation is only required to define 
a single private method as the clock handler.  This function, defined 
here as `clock`, requires a single argument that is the current cycle 
as defined by the SST Core simulation.  The `clock` method returns *true* 
if the component is ready to signal completion to the SST Core, *false* otherwise.  

In our implementation, we also define several optional parameters.  The first
parameter, `out`, is an `SST::Output` object utilized to coalesce messages 
through the SST Core to console.  This can include standard messages, warnings, 
errors and fatal messages the trigger the simulation to stop.  The second 
parameter, `clockFreq` is a string variable that will store the clock 
frequency of our component as configured by the SST simulation script.  Finally, 
we define two variables, `cycleCount` and `printInterval` as `Cycle_t` variables.  
Cycle_t is SST's data container to hold clock cycles.  The `cycleCount` 
variable is a counter that will be decremented every cycle until it reaches 
zero.  In this manner, the component will execute for `cycleCount` cycles.  The 
`printInterval` counter defines the interval at which we will print a message 
to the console during the component simulation.

```
//
// _basicComponent_h_
//

#ifndef _BASIC_COMPONENT_H_
#define _BASIC_COMPONENT_H_

#include <sst/core/sst_config.h>
#include <sst/core/component.h>

namespace SST {
namespace basicComponent {

class basicClock : public SST::Component
{
public:

  // Class members

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicClock(SST::ComponentId_t id, SST::Params& params);

  // Destructor
  ~basicClock();

private:

  // Clock handler
  bool clock(SST::Cycle_t cycle);

  // Params
  SST::Output* out;       // SST Output object for printing, messaging, etc
  std::string clockFreq;  // Clock frequency
  Cycle_t cycleCount;     // Cycle counter
  Cycle_t printInterval;  // Interval at which to print to the console

};  // class basicClocks
}   // namespace basicComponent
}   // namespace SST

#endif
```

#### SST ELI Definition

The next step in the development of our external component is to define the 
required set of SST `ELI` entries.  The ELI macro infrastructure embeds information 
in SST component libraries that can be queried and utilized by the SST Core.  
The ELI infrastructure provides what is effectively a "feature list" from within 
the respective component implementation.

Each new SST component is required to include at least one ELI macro definition.  This
macro, `SST_ELI_REGISTER_COMPONENT` contains the necessary information for 
the SST Core to load, register and execute the desired component infrastructure.  
The `SST_ELI_REGISTER_COMPONENT` macro requires six arguments as follows:

* *class* : The `class` parameter contains the name of the external components 
class.  In this case, our class name is `basicClock`.  This parameter is the 
actual object name, not a string containing the object name.
* *library* : The `library` parameter contains the name of the library that 
contains our component class.  You'll notice from our makefile above that we 
construct a shared library called `libbasicComponent.so`.  As a result, our 
libary parameter name becomes `basicComponent`.
* *name* : The `name` parameter contains the name of the component as a string.
This is generally the same as the class name, but quoted as a string.
* *version* : The `version` parameter contains the version information of the 
component, not the SST Core.  In this case, our version is "1.0.0", which is 
specified by `SST_ELI_ELEMENT_VERSION(1,0,0)`.
* *description* : The `description` parameter contains a basic textual 
description of the component's functionality in a quoted string.  This is utilized 
by the `sst-info` command line tool for printing component information.
* *category* : Finally, the `category` parameter contains a descriptor for the 
type of device that this component describes.  The permissible set of categories 
is as follows:
  * `COMPONENT_CATEGORY_UNCATEGORIZED` : Components that don't fit traditional hardware devices
  * `COMPONENT_CATEGORY_PROCESSOR` : Components that simulate processor devices
  * `COMPONENT_CATEGORY_MEMORY` : Components that simulate memory devices
  * `COMPONENT_CATEGORY_NETWORK` : Components that simulate network devices
  * `COMPONENT_CATEGORY_SYSTEM` : Components that simulate full systems

In addition to the required `SST_ELI_REGISTER_COMPONENT` ELI macro, we also 
utilize the `SST_ELI_DOCUMENT_PARAMS` macro to define the set of parameters 
defined within the respective component.  These parameters are read from the 
SST simulation configuration script and passed to the component in the 
`SST::Params` object.  The component constructor can then read the parameters 
from within the component constructor and utilize the values for the 
target simulation.  Each entry in the `SST_ELI_DOCUMENT_PARAMS` macro requires 
three arguments.  These arguments are described as follows:

* *name* : The `name` field denotes the name of the parameter as it will be 
utilized in SST simulation configuration scripts.  The name should be a quoted 
string.
* *description* : The `description` field contains a short description of the 
target parameter.  This information is utilzied by the `sst-info` command line 
tool to interpret the parameter lists from the external component and provide 
information to the user on what features are contained therein.
* *default value* : The `default value` contains the default value for the 
respective parameter if it is not specified in the simulation configuration.  
If this field is specified as `NULL`, then the SST Core will force the user 
to specify a value for the parameter prior to executing a simulation.

As you can see from our updated header file below, we define the two 
parameters, `clockFreq` and `clockTicks`, in the `SST_ELI_DOCUMENT_PARAMS` 
macro.  The default value for the clock frequency is *1Ghz* and the default 
value for the number of clock cycles to execute is *500*.

For the remaining ELI macros, `SST_ELI_DOCUMENT_PORTS`, `SST_ELI_DOCUMENT_STATISTICS`, and 
`SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS`, we do not specify any values as these 
macros are optional and not required for our basicComponent.  However, we will 
expand upon our initial example and add statistics below.

The full documentation for the ELI macro infrastructure 
can be found [here](http://sst-simulator.org/SSTPages/SSTDeveloperNewELIMigrationGuide/).

```
//
// _basicComponent_h_
//

#ifndef _BASIC_COMPONENT_H_
#define _BASIC_COMPONENT_H_

#include <sst/core/sst_config.h>
#include <sst/core/component.h>

namespace SST {
namespace basicComponent {

class basicClock : public SST::Component
{
public:

  // Register the component with the SST element library
  SST_ELI_REGISTER_COMPONENT(
    basicClock,                             // Component class
    "basicComponent",                       // Component library
    "basicClock",                           // Component name
    SST_ELI_ELEMENT_VERSION(1,0,0),         // Version of the component
    "basicClock: simple clocked component", // Description of the component
    COMPONENT_CATEGORY_UNCATEGORIZED        // Component category
  )

  // Document the parameters that this component accepts
  // { "parameter_name", "description", "default value or NULL if required" }
  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "clockTicks", "Number of clock ticks to execute",              "500"  }
  )

  // [Optional] Document the ports: we do not define any
  SST_ELI_DOCUMENT_PORTS()

  // [Optional] Document the statisitcs: we do not define any
  SST_ELI_DOCUMENT_STATISTICS()

  // [Optional] Document the subcomponents: we do not define any
  SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS()

  // Class members

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicClock(SST::ComponentId_t id, SST::Params& params);

  // Destructor
  ~basicClock();

private:

  // Clock handler
  bool clock(SST::Cycle_t cycle);

  // Params
  SST::Output* out;       // SST Output object for printing, messaging, etc
  std::string clockFreq;  // Clock frequency
  Cycle_t cycleCount;     // Cycle counter
  Cycle_t printInterval;  // Interval at which to print to the console

};  // class basicClocks
}   // namespace basicComponent
}   // namespace SST

#endif
```


### Component Implementation

Now that we have our basiComponent header defined, we can begin 
implementing the logic in the `basicComponent.cc` file.  Similar 
to other C++ class implementations, we first include the class's 
header file and namespace declarations.  Next, we define three 
functions that include the constructor, destructor and clock 
handler methods.

As mentioned above, the component's constructor requires the 
`ComponentId_t` and `Params` arguments passed from the SST Core.  
Further, we utilize the `id` value to initialize the base `Component` 
class.  Our constructor's implementation has three major functions.  
First, we create a simulation `Output` object that provides us the 
ability to print messages to the desired I/O output target using 
the SST infrastructure.  The constructor for the `Output` argument 
requires four arguments outlined as follows:

* *prefix* : This is a character string prefix that is prepended 
to all strings emitted by calls to `debug()`, `verbose()`, `fatal()` 
and `output()`.  This is useful for message coalescing of large, complex
simulations.  For our example, we do not specify a prefix value.
* *verbose_level* : `verbose_level` is a uint32_t value that defines 
the level of which calls to `debug()`, `verbose()` and `fatal()` are output 
if their `output_level` is less than or equal to the target value.  For our 
example, we specify a value of *1* such that all messages are printed if their 
`output_level` is *0* or *1*.
* `verbose_mask`: `verbose_mask` is a bitmask of allowed message types for
`debug()`, `verbose()` and `fatal()`.  The Output object will only 
output the message if the set bits of the output_bits parameter are set 
in the `verbose_mask` of the object.  In our example, we set this value to *0*.
* `location` : `location` sets the target output location, in this case 
`Output::STDOUT`.  Users may also specify: 
  * `Output::STDOUT` : print to STDOUT
  * `Output::STDERR` : print to STDERR
  * `Output::FILE` : prints to a file defined in a fifth argument to 
  the `Output` cosntructor or the file defined by the `--debug` runtime parameter.
  * `Output::NONE` : no output is generated

Once the `Output` object is created, components can send messages through to the 
SST Core using similar variadic syntax as is utilized by `printf` and the `output` method.  Character 
strings are pertibuted by special argument designates (%s, %d, %ld, etc) and the 
appropriate variables are passed to the function.

The next step in the constructor's implementation file is to read the 
parameters from the simulation input.  The `params` argument to the constructor 
holds all the values passed to the respective component.  Using this `params` 
object, we can use templated `find` methods to search the parameters 
list and derive their value.  In this way, we can utilize 
parameters find method in the form:

```
value = params.find<type>("param_name", "default_value");
```

The next step in the contrustor is to register the necessary functionality 
with the SST Core.  For our component, we seek to ensure that the 
our component is registered as a primary component and the simulation should 
not complete until our component signals the core that it is ready to complete.  
This requires us to use two functions from the SST Core's base component class, 
`registerAsPrimaryComponent()` and `primaryComponentDoNotEndSim()`, respectively.

Finally, we want to register our local clock handler, `clock` such that it is called 
by the SST core for clocked events.  For this, we utilize the `registerClock` method 
that requires us to pass the clock frequency (`clockFreq`) and the basicComponent's 
clock method as arguments.

In the final portion of our constructor, we initialize our `printInterval` 
internal variable.  Notice that we utilize the total number of clock cycles 
divided by *10*.  This implies that our component will print ten messages 
to the console from the `clock` method at an equivalent cadence.

Our destructor method is rather simple.  The only requirement for the destructor 
is to destroy the `Output` object created in the constructor.

The final method that we must implement is the `clock` method.  This method 
accepts a single argument from the SST Core that represents the simulated clock 
cycle for which the method was invoked.  This functions handles the main 
event loop for the simulated component.  This is where any component-specific 
logic is triggered for a respective clock cycle.

In our `basicClock` component, we only implement a small amount of logic.
First, if the clock cycle modulo the `printInterval` is equal to zero, 
we print a message to the output stream of the current simulation cycle 
and the current simulated time (in nanoseconds).  These functions are 
available from the base component class.  Next, we decrement 
our cycle count.  Once our cycle count has reached *0*, we signal the 
SST Core that our component has reached a completion point by calling 
`primaryComponentOKToEndSim()`.

```
//
// _basicComponent_cc_
//

#include "basicComponent.h"

using namespace SST;
using namespace SST::basicComponent;

// basicClock constructor
// - Read the parameters
// - Register the clock handler
basicClock::basicClock(ComponentId_t id, Params& params)
  : Component(id) {

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Retrieve the paramaters from the simulation script
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");
  cycleCount = params.find<Cycle_t>("clockTicks", "500");

  // Tell the simulation not to end until we signal completion
  registerAsPrimaryComponent();
  primaryComponentDoNotEndSim();

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicClock>(this, &basicClock::clock));

  out->output("Registering clock with frequency=%s\n", clockFreq.c_str());

  // This component prints the clock cycles & time every so often so calculate a print interval
  // based on simulation time
  printInterval = cycleCount / 10;
  if (printInterval < 1)
    printInterval = 1;
}

// basicClock destructor
basicClock::~basicClock(){
  delete out;
}

// main clock handler
bool basicClock::clock(Cycle_t cycles){

  // Print time in terms of clock cycles every `printInterval` clocks
  if( (cycles % printInterval) == 0 ){
    out->output("Clock cycles: %" PRIu64 ", SimulationCycles: %" PRIu64 ", Simulation ns: %" PRIu64 "\n",
                cycles, getCurrentSimCycle(), getCurrentSimTimeNano());
  }

  cycleCount--;

  // If the simulation exit condition has been met,
  // end the simulation
  if( cycleCount == 0 ){
    primaryComponentOKToEndSim();
    return true;
  }else{
    return false;
  }
}
```

### Building, Registering and Executing basicComponent

Now that we have the directory structure, makefiles, header and 
class implementation file complete, we can build our component 
from the top-level directory using the `make` command.  This 
will build the `src/libbasicComponent.so` library.

Installing and registering the external component with the SST 
Core can be done using the `make install` command.  Note that this 
requires the `sst-register` application to be in the PATH variable.
This ensures that the SST Core has the path to the shared library and 
can query the library's contents.

After registering the library with the SST Core, we can verify that the library 
is correctly configured by using the `sst-info` command.  In previous tutorial 
materials, we noted that the `sst-info` command can take an optional set of arguments 
that represent the component library name and/or component name in order to filter 
the results.  If we filter the results on `basicComponent`, `sst-info` will show 
us that our library has a single component, `basicClock` that has two parameters 
as follows.

```
$> sst-info basicComponent
Filtering output on Element = "basicComponent"
================================================================================
ELEMENT 0 = basicComponent ()
Num Components = 1
      Component 0: basicClock
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 0
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = clockTicks (Number of clock ticks to execute) [500]
    basicClock: basicClock: simple clocked component
    Using ELI version 0.9.0
    Compiled on: Mar 28 2022 17:56:04, using file: basicComponent.h
Num SubComponents = 0
Num Modules = 0
Num SSTPartitioners = 0
```

Once we have verified that our external component is correctly registered 
with the SST Core, we can begin testing.  For this, we can create a simple 
Python simulation script that utilizes the `basicClock` component.  
In this Python simulation script, we initialize our component and set the parameter 
arguments to values that are not the default defined in the implementation 
of the component.  Our simulation script will resemble the following.  
For more information on building and configuring Python and/or JSON 
configuration scripts, refere to the Level 1-2 tutorial material.

```
#
# basicClock.py
#

import os
import sst

clockComponent = sst.Component("ClockComponent", "basicComponent.basicClock")
clockComponent.addParams({
  "clockFreq"  : "1.5Ghz",
  "clockTicks" : 1000
})
```

Executing our test sample will induce output similar to the following:

```
$> sst basicClock.py
WARNING: Building component "ClockComponent" with no links assigned.
Registering clock with frequency=1.5Ghz
Clock cycles: 100, SimulationCycles: 66700, Simulation ns: 66
Clock cycles: 200, SimulationCycles: 133400, Simulation ns: 133
Clock cycles: 300, SimulationCycles: 200100, Simulation ns: 200
Clock cycles: 400, SimulationCycles: 266800, Simulation ns: 266
Clock cycles: 500, SimulationCycles: 333500, Simulation ns: 333
Clock cycles: 600, SimulationCycles: 400200, Simulation ns: 400
Clock cycles: 700, SimulationCycles: 466900, Simulation ns: 466
Clock cycles: 800, SimulationCycles: 533600, Simulation ns: 533
Clock cycles: 900, SimulationCycles: 600300, Simulation ns: 600
Clock cycles: 1000, SimulationCycles: 667000, Simulation ns: 667
Simulation is complete, simulated time: 667 ns
```

### Adding Statistics

Now that we have a basic, functional simulation component, we would 
like to begin collecting component-specific data during the simulation.  
This requires us to implement one or more SST `Statistic` objects where 
we can accumulate data.  We will utilize the same `basicClock` component 
created above and add three statistic metrics that represent:

* The number of clock cycles that are even numbers
* The number of clock cycles that are odd numbers
* The number of clock cycles evenly divisible by 100

The process of adding statistics will require us to edit the header 
and implementation file defined above.  Through this process we will 
be adding the following content: 

* ELI information defining the statistics
* Definitions for the statistics container variables
* Statistic variable initialization
* Statistic data accumulation

#### Defining Statistics

The first step in adding statistics to our `basicClock` component 
is to add the necessary `Statistic` variable containers to our header 
file.  As we see below, we define three templated `Statistic` containers 
as follows.  Notice that these variables are templated pointers.

* *EvenCycles* : Defined as an unsigned 64 bit value to track the number of clock 
cycles that are even numbers
* *OddCycles* : Defined as an unsigned 64 bit value to track the number of clock 
cycles that are odd numbers
* *HundredCycles* : Defined as an unsigned 32 bit value to track the number of clock 
cycles that are evenly divisble by 100

Now that we have our three `Statistic` containers defined, we need to add the 
necessary documentation for these values to the `SST_ELI_DOCUMENT_STATISTICS` 
ELI macro.  This process is similar to documenting our parameters 
from above.  For each statistic container, we need to add an entry to the ELI 
macro such that the SST Core may identify and correctly handle the statistic.  The 
arguments to the `SST_ELI_DOCUMENT_STATISTICS` macro are outlined as follows:

* *name* : The parameter is a string that identifies the name of the respective 
statistic.  This will be utilized in the implementation file when we 
initialize the statistic container.  It will also be utilized in the output 
generated by the SST Core as an identifier for the accumulated values.
* *description* : This parameter provides a simple description of the 
statistic.  This is utilized by the `sst-info` command line tool to 
provide information to the user regarding the component's statistics values.
* *unit* : This parameter is a string that identifies the units associated 
with the target statistics value.
* *enable level* : This parameter is a decimal value that defines the "level" 
for which the statistic is enabled.  For example, if a statistic's enable 
level is set to *3*, but the simulation configuration only enables statistics 
at level *2*, then this value will not be printed in the statistic output. In 
our example below, we enable the *EvenCycleCounter* at level 1, the *OddCycleCounter* 
at level 2 and the *HundredCycleCounter* at level 3.

```
//
// _basicComponent_h_
//

#ifndef _BASIC_COMPONENT_H_
#define _BASIC_COMPONENT_H_

#include <sst/core/sst_config.h>
#include <sst/core/component.h>

namespace SST {
namespace basicComponent {

class basicClock : public SST::Component
{
public:

  // Register the component with the SST element library
  SST_ELI_REGISTER_COMPONENT(
    basicClock,                             // Component class
    "basicComponent",                       // Component library
    "basicClock",                           // Component name
    SST_ELI_ELEMENT_VERSION(1,0,0),         // Version of the component
    "basicClock: simple clocked component", // Description of the component
    COMPONENT_CATEGORY_UNCATEGORIZED        // Component category
  )

  // Document the parameters that this component accepts
  // { "parameter_name", "description", "default value or NULL if required" }
  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "clockTicks", "Number of clock ticks to execute",              "500"  }
  )

  // [Optional] Document the ports: we do not define any
  SST_ELI_DOCUMENT_PORTS()

  // Document the statisitcs: we do not define any
  SST_ELI_DOCUMENT_STATISTICS(
    {"EvenCycleCounter", "Counts even numbered cycles", "unitless", 1},
    {"OddCycleCounter",  "Counts odd numbered cycles",  "unitless", 2},
    {"HundredCycleCounter", "Counts every 100th cycle", "unitless", 3}
  )

  // [Optional] Document the subcomponents: we do not define any
  SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS()

  // Class members

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicClock(SST::ComponentId_t id, SST::Params& params);

  // Destructor
  ~basicClock();

private:

  // Clock handler
  bool clock(SST::Cycle_t cycle);

  // Params
  SST::Output* out;       // SST Output object for printing, messaging, etc
  std::string clockFreq;  // Clock frequency
  Cycle_t cycleCount;     // Cycle counter
  Cycle_t printInterval;  // Interval at which to print to the console

  // Statistics
  Statistic<uint64_t>* EvenCycles;    // Even cycle statistics counter
  Statistic<uint64_t>* OddCycles;     // Odd cycle statistics counter
  Statistic<uint32_t>* HundredCycles; // Hundred cycle statistics counter

};  // class basicClocks
}   // namespace basicComponent
}   // namespace SST

#endif
```

#### Implementing Statistics

At this point, we can begin implementing our statistics in the 
C++ implementation for `basicClock`.  The first step in this 
process is to utilize the templated `registerStatistic` method 
to create the storage and initialize the `Statistic` containers in the 
`basicClock` constructor.  This 
requires that we utilize the statistic container names from the 
ELI information of our component's header.  Note how we initialize 
each of the target counters using their respective data types as template 
arguments.

Finally, we can begin utilizing our statistics counters in the 
`clock` function of our component.  For this, we build a series 
of conditional statements that, if true, add a single value to 
each respective counter using the `addData` method.  For these 
simple examples, each counter is incremented by *1*.  However, 
it is entirely possible to increment counters by other values 
depending upon what the target statistic is tracking.

```
//
// _basicComponent_cc_
//

#include "basicComponent.h"

using namespace SST;
using namespace SST::basicComponent;

// basicClock constructor
// - Read the parameters
// - Register the clock handler
basicClock::basicClock(ComponentId_t id, Params& params)
  : Component(id) {

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Retrieve the paramaters from the simulation script
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");
  cycleCount = params.find<Cycle_t>("clockTicks", "500");

  // Tell the simulation not to end until we signal completion
  registerAsPrimaryComponent();
  primaryComponentDoNotEndSim();

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicClock>(this, &basicClock::clock));

  out->output("Registering clock with frequency=%s\n", clockFreq.c_str());

  // Register statistics
  EvenCycles = registerStatistic<uint64_t>("EvenCycleCounter");
  OddCycles  = registerStatistic<uint64_t>("OddCycleCounter");
  HundredCycles = registerStatistic<uint32_t>("HundredCycleCounter");


  // This component prints the clock cycles & time every so often so calculate a print interval
  // based on simulation time
  printInterval = cycleCount / 10;
  if (printInterval < 1)
    printInterval = 1;
}

// basicClock destructor
basicClock::~basicClock(){
  delete out;
}

// main clock handler
bool basicClock::clock(Cycle_t cycles){

  // Print time in terms of clock cycles every `printInterval` clocks
  if( (cycles % printInterval) == 0 ){
    out->output("Clock cycles: %" PRIu64 ", SimulationCycles: %" PRIu64 ", Simulation ns: %" PRIu64 "\n",
                cycles, getCurrentSimCycle(), getCurrentSimTimeNano());
  }

  // Increment any potential statistics
  if( (cycles % 2) == 0 ){
    // even numbered cycle
    EvenCycles->addData(1);
  }else{
    // odd numbered cycle
    OddCycles->addData(1);
  }

  if( (cycles % 100) == 0 ){
    // every 100th cycle
    HundredCycles->addData(1);
  }

  cycleCount--;

  // If the simulation exit condition has been met,
  // end the simulation
  if( cycleCount == 0 ){
    primaryComponentOKToEndSim();
    return true;
  }else{
    return false;
  }
}
```

#### Testing the Statistics

Now that we have our statistics implemented, we can recompile 
and reinstall our external component using the `make` and 
`make install` commands.  The SST Core should now register 
our statistics in the `sst-info` output as follows:

```
$> sst-info basicComponent
ELEMENT 0 = basicComponent ()
Num Components = 1
      Component 0: basicClock
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 3
            STATISTIC 0 = EvenCycleCounter [Counts even numbered cycles] (unitless) Enable level = 1
            STATISTIC 1 = OddCycleCounter [Counts odd numbered cycles] (unitless) Enable level = 2
            STATISTIC 2 = HundredCycleCounter [Counts every 100th cycle] (unitless) Enable level = 3
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = clockTicks (Number of clock ticks to execute) [500]
    basicClock: basicClock: simple clocked component
    Using ELI version 0.9.0
    Compiled on: Mar 30 2022 20:06:41, using file: basicComponent.h
Num SubComponents = 0
Num Modules = 0
Num SSTPartitioners = 0
```

Before we execute our simulation test, we need to enable the statistics in 
our Python simulation script.  For this, we utilize the 
`setStatisticLoadLevel`, `setStatisticOutput` and `enableAllStatisticsForComponentType` 
methods.  For this example, we seek to output the data to a CSV 
file (StatisticOutput.csv) and enable all the counters.  The Python simulation 
script is otherwise unchanged

```
#
# basicClock.py
#

import os
import sst

clockComponent = sst.Component("ClockComponent", "basicComponent.basicClock")
clockComponent.addParams({
  "clockFreq"  : "1.5Ghz",
  "clockTicks" : 1000
})

# Setting this to '1' will enable the EvenCycleCounter
# Setting this to '2' will enable the Even and OddCycleCounter
# Setting this to '3' will enable all the counters
sst.setStatisticLoadLevel(3)

sst.setStatisticOutput("sst.statOutputCSV")
sst.enableAllStatisticsForComponentType("basicComponent.basicClock")
```

After we execute our simulation, SST will generate a statistics output 
file similar to the following that includes the accumulated values for 
each of the enabled statistics.  Note the comments in our simulation 
script above.  Setting the load level to values less than *3* will begin 
disabling certain counter output.  For more information on statistics 
configuration, see the Level 1 tutorial material.

```
$> cat StatisticOutput.csv
ComponentName, StatisticName, StatisticSubId, StatisticType, SimTime, Rank, Sum.u64, SumSQ.u64, Count.u64, Min.u64, Max.u64, Sum.u32, SumSQ.u32, Min.u32, Max.u32
ClockComponent, EvenCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
ClockComponent, OddCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
ClockComponent, HundredCycleCounter, , Accumulator, 667000, 0, 0, 0, 10, 0, 0, 10, 10, 1, 1
```

## Developing SST Subcomponents <a name="DevelopingSubcomponents"></a>

### Subcomponent Class Definition

Now that we have a functional example of a simple component definition, we 
will extend the `basicClock` component to include a subcomponent.  Subcomponents 
are useful when developing complex simulations that include hierarchies of 
simulated devices.  A common example of using components with subcomponents would 
be building a simulated CPU with a network interface controller (NIC).  The CPU 
would be implemented as a component and the NIC would be implemented as 
a subcomponent.  Subcomponents are implemented a similar manner as components 
using a base SST class implementation (`SST::SubComponent`).  Further, we can 
develop a base subcomponent class definition and utilize C++ inheritance to define 
multiple versions or implementations of the subcomponent that can be interchanged 
by the top-level component using the SST simulation input script.  For our example, 
we will implement a basic message subcomponent that receives randomly generated 
integers from the top-level component and buffers them to an internal vector.  The 
subcomponent will require that the user specify a parameter the defines when 
the buffer is "flushed" to the output console.  The process by which we develop 
our example subcomponent is as follows:

* Define the `basicSubcomponent` class infrastructure using `SST::SubComponent`
* Define the `basicMsg` inherited class infrastructure
* Define the subcomponent ELI information
* Implement the `basicMsg` class infrastructure
* Inject the subcomponent into the top-level component
* Update our simulation configuration and test the subcomponent

The first step in implementing our subcomponent is to modify the `basicComponent.h` 
header file.  We will define two new classes that represent our base 
subcomponent (`basicSubcomponent`) and the inherited subcomponent that contains 
our subcomponent's logic (`basicMsg`).  Notice that our subcomponent definitions 
exist in the same namespace as our component definition.  For this, we must include an additional 
header file, `<SST/core/subcomponent.h>`.  The `basicSubcomponent`
class definition can now inherit the `SST::SubComponent` class.  Just like our 
component definition, the subcomponent constructor requires a `ComponentId_t` 
and an `SST::Params` argument.  For this base class, we initialize the SST 
`SubComponent id` field with the id passed in from the simulation configuration.

The `basicSubcomponent` definition also contains two virtual methods that represnt 
the destructor and a `send` function.  The destructor is an optional virtual 
method that invokes a local destructor only if a child class as not defined 
a virtual destructor method.  The `send` function is a pure virtual method that 
is required to be defined by the child class.  We will see how this is used below.

Using the `basicSubcomponent` class definition, we can now define a child class 
that inherits the `basicSubcomponent` infrastructure.  As we see below, 
`basicMsg` inherits `basicSubcomponent` and defines an additional set of 
methods.  First, we define a local, virtual destructor and `send` method.  
We also define a local `clock` method that will be registered with the SST core.
Finally, we define a set of private variables similar to our top-level component 
definition.  The `out` variable connects to the `SST::Output` infrastructure, 
`clockFreq` holds the clock frequency read in from the parameter list, `count` 
is the number of integers to buffer before flushing a message (also read in from 
the parameter list) and `msgs` is a vector of integers.  The `msgs` variable serves 
as our internal message buffer.

Using our new subcomponent definition, we can now modify our existing 
`basicClock` class definition.  For this, we define two new private variables: 
`rng` and `msg`.  The `rng` variable is a random number generator object 
that is provided by the SST Core.  In this case, the `SST::RNG::MarsagliaRNG` 
defines a Marsaglia-style pseudo random number generator that requires 
us to also include the `sst/core/rng/marsaglia.h` header.  It is highly 
recommended to utilize the SST-provided random number generators.  This ensures 
that the random number generation is platform-agnostic.  Next, we define 
a `msg` object of type `basicSubcomponent`.  This will provide the top-level 
component access to the subcomponent features.

```
//
// _basicComponent_h_
//

#ifndef _BASIC_COMPONENT_H_
#define _BASIC_COMPONENT_H_

#include <sst/core/sst_config.h>
#include <sst/core/component.h>
#include <sst/core/subcomponent.h>
#include <sst/core/rng/marsaglia.h>
#include <vector>

namespace SST {
namespace basicComponent {

// --------------------------------------
// basicSubcomponent Subcomponent
// --------------------------------------
class basicSubcomponent : public SST::SubComponent {
public:
  // Register the subcomponent ELI information
  SST_ELI_REGISTER_SUBCOMPONENT_API(SST::basicComponent::basicSubcomponent)

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicSubcomponent(SST::ComponentId_t id, SST::Params& params)
    : SubComponent(id) { }

  // Destructor
  virtual ~basicSubcomponent() { }

  // Virtual method to inject send the subcomponent and integer
  virtual void send(int number) = 0;
};

// --------------------------------------
// basicMsg Inherited Subcomponent
// --------------------------------------
class basicMsg : public basicSubcomponent {
public:
  SST_ELI_REGISTER_SUBCOMPONENT_DERIVED(
    basicMsg,
    "basicComponent",
    "basicMsg",
    SST_ELI_ELEMENT_VERSION(1,0,0),
    "basicMsg : simple message handler subcomponent",
    SST::basicComponent::basicSubcomponent
  )

  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "count", "Wait for 'count' integers before flushing a message", "5"}
  )

  SST_ELI_DOCUMENT_PORTS(
  )

  // Constructor
  basicMsg(ComponentId_t id, Params& params);

  // Virtual Destructor
  virtual ~basicMsg();

  // Virtual send function
  virtual void send(int number);

  // Virtual clock handler
  virtual bool clock(Cycle_t cycle);

private:
  // Params
  SST::Output* out;           // SST Output object for printing, messaging, etc
  std::string clockFreq;      // Clock frequency
  unsigned count;             // Number of integers to store before flushing a message
  std::vector<int> msgs;      // Message storage
};

// --------------------------------------
// basicClock Component
// --------------------------------------
class basicClock : public SST::Component
{
public:

  // Register the component with the SST element library
  SST_ELI_REGISTER_COMPONENT(
    basicClock,                             // Component class
    "basicComponent",                       // Component library
    "basicClock",                           // Component name
    SST_ELI_ELEMENT_VERSION(1,0,0),         // Version of the component
    "basicClock: simple clocked component", // Description of the component
    COMPONENT_CATEGORY_UNCATEGORIZED        // Component category
  )

  // Document the parameters that this component accepts
  // { "parameter_name", "description", "default value or NULL if required" }
  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "clockTicks", "Number of clock ticks to execute",              "500"  }
  )

  // [Optional] Document the ports: we do not define any
  SST_ELI_DOCUMENT_PORTS()

  // Document the statisitcs: we do not define any
  SST_ELI_DOCUMENT_STATISTICS(
    {"EvenCycleCounter", "Counts even numbered cycles", "unitless", 1},
    {"OddCycleCounter",  "Counts odd numbered cycles",  "unitless", 2},
    {"HundredCycleCounter", "Counts every 100th cycle", "unitless", 3}
  )

  // Document the subcomponents
  SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS(
    {"msg", "Message Interface", "SST::basicComponent::basicMsg"}
  )

  // Class members

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicClock(SST::ComponentId_t id, SST::Params& params);

  // Destructor
  ~basicClock();

private:

  // Clock handler
  bool clock(SST::Cycle_t cycle);

  // Params
  SST::Output* out;       // SST Output object for printing, messaging, etc
  std::string clockFreq;  // Clock frequency
  Cycle_t cycleCount;     // Cycle counter
  Cycle_t printInterval;  // Interval at which to print to the console

  SST::RNG::MarsagliaRNG* rng;  // Random number generator

  // Subcomponents
  basicSubcomponent *msg; // basicSubcomponent

  // Statistics
  Statistic<uint64_t>* EvenCycles;    // Even cycle statistics counter
  Statistic<uint64_t>* OddCycles;     // Odd cycle statistics counter
  Statistic<uint32_t>* HundredCycles; // Hundred cycle statistics counter

};  // class basicClocks
}   // namespace basicComponent
}   // namespace SST

#endif

```

### Component ELI Information

Using the modifications defined above for our subcomponent definitions, we can 
now implement the necessary ELI macros that will define and register our 
subcomponents with the SST Core and the top-level component.  The first ELI 
macro is defined in the `basicSubcomponent` class.  This macro, 
`SST_ELI_REGISTER_SUBCOMPONENT_API`, requires a fully scoped class name and 
registers the `basicSubcomponent` as a valid subcomponent within the `basicComponent` 
class hierarchy.

Next, we add two required and one optional ELI macro to the `basicMsg` class 
defintion.  The first macro, `SST_ELI_REGISTER_SUBCOMPONENT_DERIVED` requires 
six arguments defined as follows:
* *class* : The `class` parameter contains the name of the external components 
class.  In this case, our class name is `basicMsg`.  This parameter is the 
actual object name, not a string containing the object name.
* *library* : The `library` parameter contains the name of the library that 
contains our component class.  You'll notice from our makefile above that we 
construct a shared library called `libbasicComponent.so`.  As a result, our 
libary parameter name becomes `basicComponent`.
* *name* : The `name` parameter contains the name of the component as a string.
This is generally the same as the class name, but quoted as a string.
* *version* : The `version` parameter contains the version information of the 
component, not the SST Core.  In this case, our version is "1.0.0", which is 
specified by `SST_ELI_ELEMENT_VERSION(1,0,0)`.
* *description* : The `description` parameter contains a basic textual 
description of the component's functionality in a quoted string.  This is utilized 
by the `sst-info` command line tool for printing component information.
* *fully qualified class*: The final parameter defines the fully qualified 
class structure that is utilized to correctly scope the included methods.

The second required ELI macro defines are parameter list for the subcomponent.  Thiss 
is the same parameter ELI as described above for our top-level component.  For 
our subcomponent, we define two parameters that include the clock frequency 
and the message flush counter.  The final ELI macro defined for our subcomponent 
is an optional macro to define the ports.  We will utilize this macro in future 
examples.

That last ELI macro that we need to implement is placed in the `basicClock` 
class definition.  This macro, `SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS` documents 
the subcomponent slot information.  The parameters for this macro are defined 
as follows:
* *slot name* : The name of the slot that will be utilized to load the 
subcomponent from the top-level component.
* *slot description* : Describes the subcomponent slot information.  This is 
utilized downstream by the `sst-info` tool.
* *subcomponent class*: Defines the fully qualified class type for the subcomponent.

### Subcomponent Class Implementation

With our class definitions complete, we can now implement the `basicMsg` methods 
and modify our existing `basicClock` implementation.  First, we need 
to define our `basicMsg` constructor.  Much in the same way as our top-level 
component, the `basicMsg` constructor initializes the SST output object and reads 
our required parameters.  It also registers the subcomponent clock method 
using the `registerClock` function.  Similarly, the destructor deletes the 
local `SST::Output` object.

Next, we define our virtual `send` function.  In this case, the send 
function is called from the top-level component and injects an integer into 
the `msgs` vector.  Finally, the `basicMsg` clock handler checks the size of the 
`msg` buffer against the `count` parameter.  If the message buffer is full, 
then the clock function prints the values and clears the buffer.

The next step in implementing our subcomponent is to tie the subcomponent object 
into the top-level component.  For this, we make two modifications to the existing 
`basicClock` constructor.  First, we need to load our subcomponent object 
using the `loadUserSubComponent` template method.  Notice that the template 
argument matches the base class type of `basicMsg` and the function argument 
is the slot name that we defined in our ELI macro.  Next, we need to create a 
new random number generator object using the `SST::RNG::MarsagliaRNG` class.  Make 
sure a delete the `rng` object in the `basicClock` destructor.

The final modification required for the `basicClock` function is to inject 
a randomly generated integer into the subcomponent every clock cycle.  This is 
as simple as calling the `send` method from our `basicMsg` *msg* object and 
passing in a pseudo random number from the Marsaglia random number generator.  This 
will be called for every clock cycle in the top-level component.

```
//
// _basicComponent_cc_
//

#include "basicComponent.h"

using namespace SST;
using namespace SST::basicComponent;

// basicMsg constructor
basicMsg::basicMsg(ComponentId_t id, Params& params)
  : basicSubcomponent(id,params){

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Read the parameters
  count = params.find<unsigned>("count",5);
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicMsg>(this, &basicMsg::clock));
  out->output("Registering subcomponent clock with frequency=%s\n", clockFreq.c_str());
}

// basicMsg destructor
basicMsg::~basicMsg(){
  delete out;
}

// basicMsg send function
void basicMsg::send(int number){
  msgs.push_back(number);
}

// basicMsg clock handler
bool basicMsg::clock(Cycle_t cycle){
  if( msgs.size() >= count ){
    out->output("Flushing message buffer at clock=%" PRIu64 "\n", cycle );
    for( unsigned i=0; i<count; i++ ){
      out->output("Buffer[%d]: %d\n", i, msgs[i]);
    }
    msgs.clear();
  }
  return false;
}

// basicClock constructor
// - Read the parameters
// - Register the clock handler
basicClock::basicClock(ComponentId_t id, Params& params)
  : Component(id) {

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Retrieve the paramaters from the simulation script
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");
  cycleCount = params.find<Cycle_t>("clockTicks", "500");

  // Tell the simulation not to end until we signal completion
  registerAsPrimaryComponent();
  primaryComponentDoNotEndSim();

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicClock>(this, &basicClock::clock));

  out->output("Registering clock with frequency=%s\n", clockFreq.c_str());

  // Register statistics
  EvenCycles = registerStatistic<uint64_t>("EvenCycleCounter");
  OddCycles  = registerStatistic<uint64_t>("OddCycleCounter");
  HundredCycles = registerStatistic<uint32_t>("HundredCycleCounter");

  // Load the subcomponent
  msg = loadUserSubComponent<basicSubcomponent>("msg");

  // Initialize the RNG
  rng = new SST::RNG::MarsagliaRNG(11, 272727);

  // This component prints the clock cycles & time every so often so calculate a print interval
  // based on simulation time
  printInterval = cycleCount / 10;
  if (printInterval < 1)
    printInterval = 1;
}

// basicClock destructor
basicClock::~basicClock(){
  delete out;
  delete rng;
}

// main clock handler
bool basicClock::clock(Cycle_t cycles){

  // Print time in terms of clock cycles every `printInterval` clocks
  if( (cycles % printInterval) == 0 ){
    out->output("Clock cycles: %" PRIu64 ", SimulationCycles: %" PRIu64 ", Simulation ns: %" PRIu64 "\n",
                cycles, getCurrentSimCycle(), getCurrentSimTimeNano());
  }

  // Increment any potential statistics
  if( (cycles % 2) == 0 ){
    // even numbered cycle
    EvenCycles->addData(1);
  }else{
    // odd numbered cycle
    OddCycles->addData(1);
  }

  if( (cycles % 100) == 0 ){
    // every 100th cycle
    HundredCycles->addData(1);
  }

  cycleCount--;

  // inject a message into the subcomponent
  msg->send( rng->generateNextInt32() );

  // If the simulation exit condition has been met,
  // end the simulation
  if( cycleCount == 0 ){
    primaryComponentOKToEndSim();
    return true;
  }else{
    return false;
  }
}

```

### Building and Testing the Subcomponent

Now that we have our header and implementation files complete, we can 
build and install our modified component library using the existing 
makefile:

```
$> make && make install
```

Once installed, we can utilize the `sst-info` command line tool to examine 
the new subcomponent features.  In the output below, we can see that our 
`basicMsg` subcomponent is registered as a subcomponent of `basicComponent` 
with the appropriate parameters.  We can also see that the `basicClock` component 
has a registered subcomponent slot for our `msg` object.

```
$> sst-info
Filtering output on Element = "basicComponent"
================================================================================
ELEMENT 0 = basicComponent ()
Num Components = 1
      Component 0: basicClock
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 3
            STATISTIC 0 = EvenCycleCounter [Counts even numbered cycles] (unitless) Enable level = 1
            STATISTIC 1 = OddCycleCounter [Counts odd numbered cycles] (unitless) Enable level = 2
            STATISTIC 2 = HundredCycleCounter [Counts every 100th cycle] (unitless) Enable level = 3
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 1
            SUB COMPONENT SLOT 0 = msg (Message Interface) [SST::basicComponent::basicMsg]
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = clockTicks (Number of clock ticks to execute) [500]
    basicClock: basicClock: simple clocked component
    Using ELI version 0.9.0
    Compiled on: Apr 11 2022 08:27:43, using file: basicComponent.h
Num SubComponents = 1
      SubComponent 0: basicMsg
      Interface: SST::basicComponent::basicSubcomponent
         NUM STATISTICS = 0
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = count (Wait for 'count' integers before flushing a message) [5]
    basicMsg: basicMsg : simple message handler subcomponent
    Using ELI version 0.9.0
    Compiled on: Apr 11 2022 08:27:43, using file: basicComponent.h
Num Modules = 0
Num SSTPartitioners = 0
```

In order to test our subcomponent, we need to modify the existing 
simulation configuration script.  We utilize the existing `clockComponent` 
definition and add a subcomponent using the `setSubComponent` method.  Notice 
that this method uses the `msg` naming convention to define the subcomponent slot 
and the `basicComponent.basicMsg` class name for the implementation.  Similarly, 
we define a set of required parameters for the subcomponent to represent the 
clock frequency (`clockFreq`) and the message buffer counter (`count`).

```
#
# basicClock.py
#

import os
import sst

clockComponent = sst.Component("ClockComponent", "basicComponent.basicClock")
clockComponent.addParams({
  "clockFreq"  : "1.5Ghz",
  "clockTicks" : 1000
})

msgComp = clockComponent.setSubComponent("msg", "basicComponent.basicMsg")
msgComp.addParams({
  "clockFreq"  : "1.5Ghz",
  "count" : 10
})

sst.setStatisticLoadLevel(3)
sst.setStatisticOutput("sst.statOutputCSV")
sst.enableAllStatisticsForComponentType("basicComponent.basicClock")
```

Finally, we can test our new subcomponent configuration by using the 
same SST command as outlined above.  Notice that there are messages 
printed to the console from the top-level component and the subcomponent.

```
$> sst basicClock.py
Registering clock with frequency=1.5Ghz
Registering subcomponent clock with frequency=1.5Ghz
Flushing message buffer at clock=10
Buffer[0]: 1071494452
Buffer[1]: 1803936666
Buffer[2]: -1672778268
Buffer[3]: 902491587
Buffer[4]: -449790690
Buffer[5]: 693997229
Buffer[6]: -319940607
Buffer[7]: -1946751616
Buffer[8]: -725729641
Buffer[9]: -1926751637
Flushing message buffer at clock=20
Buffer[0]: 583439521
Buffer[1]: -2008186991
Buffer[2]: -1945981918
Buffer[3]: -2092680791
Buffer[4]: 231893527
Buffer[5]: 954171654
Buffer[6]: -757984509
Buffer[7]: 605021792
Buffer[8]: 946094904
Buffer[9]: 824807622
Flushing message buffer at clock=30
Buffer[0]: -71994282
Buffer[1]: 1065813667
Buffer[2]: 481084644
Buffer[3]: -347217646
Buffer[4]: 1103132595
Buffer[5]: -1809672255
Buffer[6]: -1845996752
Buffer[7]: -1103673170
Buffer[8]: -732405447
Buffer[9]: -2125526074
.....
Clock cycles: 1000, SimulationCycles: 667000, Simulation ns: 667
Flushing message buffer at clock=1000
Buffer[0]: -1391397147
Buffer[1]: -1286613259
Buffer[2]: 1314549343
Buffer[3]: 1225175738
Buffer[4]: 547476592
Buffer[5]: -1032637418
Buffer[6]: 135824697
Buffer[7]: -2059975138
Buffer[8]: 1103774199
Buffer[9]: -563594638
Simulation is complete, simulated time: 667 ns
```

## Connecting Components with Links & Ports <a name="ConnectingComponents"></a>

### Introduction to Links and Events

The SST infrastructure provides a convenient infrastructure for simulating 
the exchange of arbitrary events between components or subcomponents.  The SST 
`Link` class provides the basic methods by which to setup links between 
components, initialize the links, assign latency to the links and exchange 
messages.  The SST `Event` class provides the basic methods by which to define 
data payloads as well as the serialization methods required to pack the payloads 
into discernible packets for sending/receiving across the links.

In our example, we will develop a basic link event class, add a link port 
definition to our subcomponent and implement a message handler to capture 
the received event packets.  For this example, we will utilize the 
component/subcomponent infrastructure developed above to flush the integer 
message buffer across a link to an adjacent component/subcomponent pair.

This process includes the following steps:
* Define a new `SST::Event` child class to handle the message payloads
* Define a new port within our subcomponent class for link connectivity
* Update the ELI information for our subcomponent
* Implement the necessary message send/receive functions within the subcomponent class
* Update our `basicClock` simulation script to define two unique components
connected via links

### Defining the Event Handler

First, we need to add two include statements that provide us access to 
the `sst/core/event.h` methods and the `sst/core/link.h` methods.
Now we may define a new class that implements the `SST::Event` 
methods.  Our class, `basicMsgEvent` inherits functionality from `SST::Event` 
and defines two constructors.  We define a basic constructor as well as 
an overloaded constructor that requires an argument that is our message buffer vector.  
The message buffer vector is assigned to the private variable `payload`.  We are also 
required to define and implement three methods as follows:

* *serialize_order* : This method overrides the base `SST::Event` serialization 
method.  The serializer ensures that the event payload is correctly organized 
as a serialized binary, regardless of type or size.  This is vital for correct 
event handling given that C++ objects may contain disparate data members.
* *ImplementSerializable* : This method is utilized to define the inherited child 
class as a serializable class infrastructure.  It requires a single 
argument that is the fully qualified class name of the inherited child.  This is 
required by all inherited children of `SST::Event` that seek to exchange payloads 
over links.
* *getPayload* : This method is a convenience method that allows the calling 
component/subcomponent to retrieve the private payload data from the event.

```
// --------------------------------------
// basicMsgEvent Event handler
// --------------------------------------
class basicMsgEvent : public SST::Event {
public:
  // Basic Constructor : inherits SST::Event()
  basicMsgEvent() : SST::Event() { }

  // Overloaded Constructor
  basicMsgEvent(std::vector<int> Msg) : SST::Event(), payload(Msg) { }

  // Events must provide a serialization function that serializes
  // all data members of the event
  void serialize_order(SST::Core::Serialization::serializer &ser)  override {
    Event::serialize_order(ser);
    ser & payload;
  }

  // Retrieve the event payload data
  std::vector<int> getPayload() { return payload; }

  // Register this event as serializable
  ImplementSerializable(SST::basicComponent::basicMsgEvent);

private:
  // Message payload
  std::vector<int> payload;
};
```

### Defining the Link Infrastructure

Now that we have our event payload class defined, we can begin 
modifying our existing subcomponent class in order to implement 
our SST links.  Note that we implement links within a subcomponent, 
but links may also be implemented in top-level components.  The first 
step in defining the link infrastructure is to add a private 
method to handle the incoming received event paylods, `handleEvent`.  
The `handleEvent` method requires a single argument in the form 
of an `SST::Event` that represents the received event payload.  Next, 
we need to define a private variable to hold our `SST::Link` link handler.  
This object will be utilized to send event payloads over the link to an 
adjacent component or subcomponent.

Note that since our `basicMsg` subcomponent handles all the link and port 
configuration, we are not required to modify any portions of the `basicClock` 
class infrastructure.

```
//
// _basicComponent_h_
//

#ifndef _BASIC_COMPONENT_H_
#define _BASIC_COMPONENT_H_

#include <sst/core/sst_config.h>
#include <sst/core/component.h>
#include <sst/core/subcomponent.h>
#include <sst/core/event.h>
#include <sst/core/link.h>
#include <sst/core/rng/marsaglia.h>
#include <vector>

namespace SST {
namespace basicComponent {

// --------------------------------------
// basicMsgEvent Event handler
// --------------------------------------
class basicMsgEvent : public SST::Event {
public:
  // Basic Constructor : inherits SST::Event()
  basicMsgEvent() : SST::Event() { }

  // Overloaded Constructor
  basicMsgEvent(std::vector<int> Msg) : SST::Event(), payload(Msg) { }

  // Events must provide a serialization function that serializes
  // all data members of the event
  void serialize_order(SST::Core::Serialization::serializer &ser)  override {
    Event::serialize_order(ser);
    ser & payload;
  }

  std::vector<int> getPayload() { return payload; }

  // Register this event as serializable
  ImplementSerializable(SST::basicComponent::basicMsgEvent);

private:
  // Message payload
  std::vector<int> payload;
};

// --------------------------------------
// basicSubcomponent Subcomponent
// --------------------------------------
class basicSubcomponent : public SST::SubComponent {
public:
  // Register the subcomponent ELI information
  SST_ELI_REGISTER_SUBCOMPONENT_API(SST::basicComponent::basicSubcomponent)

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicSubcomponent(SST::ComponentId_t id, SST::Params& params)
    : SubComponent(id) { }

  // Destructor
  virtual ~basicSubcomponent() { }

  // Virtual method to inject send the subcomponent and integer
  virtual void send(int number) = 0;
};

// --------------------------------------
// basicMsg Inherited Subcomponent
// --------------------------------------
class basicMsg : public basicSubcomponent {
public:
  SST_ELI_REGISTER_SUBCOMPONENT_DERIVED(
    basicMsg,
    "basicComponent",
    "basicMsg",
    SST_ELI_ELEMENT_VERSION(1,0,0),
    "basicMsg : simple message handler subcomponent",
    SST::basicComponent::basicSubcomponent
  )

  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "count", "Wait for 'count' integers before flushing a message", "5"}
  )

  // { “name”, “description”, vector of supported events }
  SST_ELI_DOCUMENT_PORTS(
    {"msgPort",
     "Link to another component. Uses an event handler to capture events.",
     { "basicComponent.basicMsgEvent", ""}}
  )

  // Constructor
  basicMsg(ComponentId_t id, Params& params);

  // Virtual Destructor
  virtual ~basicMsg();

  // Virtual send function
  virtual void send(int number);

  // Virtual clock handler
  virtual bool clock(Cycle_t cycle);

private:
  // Event handler
  void handleEvent(SST::Event *ev);

  // Params
  SST::Output* out;           // SST Output object for printing, messaging, etc
  std::string clockFreq;      // Clock frequency
  unsigned count;             // Number of integers to store before flushing a message
  std::vector<int> msgs;      // Message storage

  SST::Link *linkHandler;     // Link handler object
};

// --------------------------------------
// basicClock Component
// --------------------------------------
class basicClock : public SST::Component
{
public:

  // Register the component with the SST element library
  SST_ELI_REGISTER_COMPONENT(
    basicClock,                             // Component class
    "basicComponent",                       // Component library
    "basicClock",                           // Component name
    SST_ELI_ELEMENT_VERSION(1,0,0),         // Version of the component
    "basicClock: simple clocked component", // Description of the component
    COMPONENT_CATEGORY_UNCATEGORIZED        // Component category
  )

  // Document the parameters that this component accepts
  // { "parameter_name", "description", "default value or NULL if required" }
  SST_ELI_DOCUMENT_PARAMS(
    { "clockFreq",  "Frequency of period (with units) of the clock", "1GHz" },
    { "clockTicks", "Number of clock ticks to execute",              "500"  }
  )

  // [Optional] Document the ports: we do not define any
  SST_ELI_DOCUMENT_PORTS()

  // Document the statisitcs: we do not define any
  SST_ELI_DOCUMENT_STATISTICS(
    {"EvenCycleCounter", "Counts even numbered cycles", "unitless", 1},
    {"OddCycleCounter",  "Counts odd numbered cycles",  "unitless", 2},
    {"HundredCycleCounter", "Counts every 100th cycle", "unitless", 3}
  )

  // Document the subcomponents
  SST_ELI_DOCUMENT_SUBCOMPONENT_SLOTS(
    {"msg", "Message Interface", "SST::basicComponent::basicMsg"}
  )

  // Class members

  // Constructor: Components receive a unique ID and the set of parameters
  //              that were assigned in the simulation configuration script
  basicClock(SST::ComponentId_t id, SST::Params& params);

  // Destructor
  ~basicClock();

private:

  // Clock handler
  bool clock(SST::Cycle_t cycle);

  // Params
  SST::Output* out;       // SST Output object for printing, messaging, etc
  std::string clockFreq;  // Clock frequency
  Cycle_t cycleCount;     // Cycle counter
  Cycle_t printInterval;  // Interval at which to print to the console

  SST::RNG::MarsagliaRNG* rng;  // Random number generator

  // Subcomponents
  basicSubcomponent *msg; // basicSubcomponent

  // Statistics
  Statistic<uint64_t>* EvenCycles;    // Even cycle statistics counter
  Statistic<uint64_t>* OddCycles;     // Odd cycle statistics counter
  Statistic<uint32_t>* HundredCycles; // Hundred cycle statistics counter

};  // class basicClocks
}   // namespace basicComponent
}   // namespace SST

#endif

```

### Link ELI Information

Much in the same manner as defining new subcomponents, we also need to 
setup the necessary ELI macro information for the SST port configurations.  For
this, we will add an entry to the `basicMsg` subcomponent ELI macro for 
`SST_ELI_DOCUMENT_PORTS`.  This defines the available ports 
and the supported events for each port.  The required fields for this 
ELI macro as listed as follows:

* *name* : Defines the name of the respective port to be utilized when loading 
link configurations.  In this case, we use `msgPort`
* *description* : Describes the link functionality in order to provide additional 
information from the `sst-info` command line tool
* *vector of supported events* : This parameters takes a vector of strings that represents 
the list of supported event types.  The event types are defined by their 
component class hierarchy.  In this case, we have only a single event: `basicComponent.basicMsgEvent`

### Implementing the Link Infrastructure

Now that we have our class infrastructure defined in the `basicComponent.h` 
header file, we can begin implementing the new methods.  The first step in doing 
so requires us to modify the `basicMsg` constructor method and add the 
link configuration.  This can be done via a single call to `configureLink` 
that requires the port name and handler method as arguments.  In this case, 
we use `msgPort` as the port name and our `basicMsg::handleEvent` method 
as the message handler.  Note that the object created is stored as the 
`linkHandler`.

The next step is to define our `basicMsg::handlEvent` method.  Remember 
from our class definition that this method requires an `SST::Event` 
object as an argument.  This argument is dynamically cast to 
our `basicMsgEvent` class type such that we have access to our 
simulation-specific payload data.  If the cast is success (non null 
pointer), then we can begin processing our event.  If the cast 
is not successful, then an adjacent component has sent us an unknown 
event payload type.  Assuming that the event is valid, we can 
retrieve the payload contents using our `getPayload` method 
and print the contents to the SST output console.

Note that the `handleEvent` method is a callback method.  It is called 
when an event arrives on designated port specified by the link 
configuration.  It is the responsibility of the handler to delete the event object after event 
processing is complete.

The final modification required to successfully send/receive event payloads 
is to modify our `basicMsg::clock` method.  Remember from our previous example 
that we printed the message payload when the buffer size reached the `count` 
counter.  In this example, when the counter saturates, we create a new 
`basicMsgEvent` object using our overloaded `basicMsgEvent` constructor 
and inject the serialized payload into the link via the `linkHandler->send` 
method.

```
//
// _basicComponent_cc_
//

#include "basicComponent.h"

using namespace SST;
using namespace SST::basicComponent;

// basicMsg constructor
basicMsg::basicMsg(ComponentId_t id, Params& params)
  : basicSubcomponent(id,params){

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Read the parameters
  count = params.find<unsigned>("count",5);
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicMsg>(this, &basicMsg::clock));
  out->output("Registering subcomponent clock with frequency=%s\n", clockFreq.c_str());

  // Configure the links
  linkHandler = configureLink("msgPort", new Event::Handler<basicMsg>(this, &basicMsg::handleEvent));
}

// basicMsg destructor
basicMsg::~basicMsg(){
  delete out;
}

// basicMsg send function
void basicMsg::send(int number){
  msgs.push_back(number);
}

// basicMsg event handler
void basicMsg::handleEvent(SST::Event *ev){
  basicMsgEvent *event = dynamic_cast<basicMsgEvent*>(ev);

  if( event ){
    std::vector<int> msgs = event->getPayload();

    if( msgs.size() > 0 ){
      out->output("Received message buffer at clock%" PRIu64 "\n", getCurrentSimCycle());
      for( unsigned i=0; i<count; i++ ){
        out->output("Received buffer[%d]: %d\n", i, msgs[i]);
      }
    }
    delete event;
  }else{
    out->fatal(CALL_INFO, -1, "Error: Bad event type received!\n" );
  }
}

// basicMsg clock handler
bool basicMsg::clock(Cycle_t cycle){
  if( msgs.size() >= count ){
    out->output("Flushing message buffer at clock=%" PRIu64 "\n", cycle );
    for( unsigned i=0; i<count; i++ ){
      out->output("Sent Buffer[%d]: %d\n", i, msgs[i]);
    }

    basicMsgEvent *event = new basicMsgEvent(msgs);
    linkHandler->send(event);

    msgs.clear();
  }
  return false;
}

// basicClock constructor
// - Read the parameters
// - Register the clock handler
basicClock::basicClock(ComponentId_t id, Params& params)
  : Component(id) {

  // Create a new SST output object
  out = new Output("", 1, 0, Output::STDOUT);

  // Retrieve the paramaters from the simulation script
  clockFreq  = params.find<std::string>("clockFreq", "1GHz");
  cycleCount = params.find<Cycle_t>("clockTicks", "500");

  // Tell the simulation not to end until we signal completion
  registerAsPrimaryComponent();
  primaryComponentDoNotEndSim();

  // Register the clock handler
  registerClock(clockFreq, new Clock::Handler<basicClock>(this, &basicClock::clock));

  out->output("Registering clock with frequency=%s\n", clockFreq.c_str());

  // Register statistics
  EvenCycles = registerStatistic<uint64_t>("EvenCycleCounter");
  OddCycles  = registerStatistic<uint64_t>("OddCycleCounter");
  HundredCycles = registerStatistic<uint32_t>("HundredCycleCounter");

  // Load the subcomponent
  msg = loadUserSubComponent<basicSubcomponent>("msg");

  // Initialize the RNG
  rng = new SST::RNG::MarsagliaRNG(11, 272727);

  // This component prints the clock cycles & time every so often so calculate a print interval
  // based on simulation time
  printInterval = cycleCount / 10;
  if (printInterval < 1)
    printInterval = 1;
}

// basicClock destructor
basicClock::~basicClock(){
  delete out;
  delete rng;
}

// main clock handler
bool basicClock::clock(Cycle_t cycles){

  // Print time in terms of clock cycles every `printInterval` clocks
  if( (cycles % printInterval) == 0 ){
    out->output("Clock cycles: %" PRIu64 ", SimulationCycles: %" PRIu64 ", Simulation ns: %" PRIu64 "\n",
                cycles, getCurrentSimCycle(), getCurrentSimTimeNano());
  }

  // Increment any potential statistics
  if( (cycles % 2) == 0 ){
    // even numbered cycle
    EvenCycles->addData(1);
  }else{
    // odd numbered cycle
    OddCycles->addData(1);
  }

  if( (cycles % 100) == 0 ){
    // every 100th cycle
    HundredCycles->addData(1);
  }

  cycleCount--;

  // inject a message into the subcomponent
  msg->send( rng->generateNextInt32() );

  // If the simulation exit condition has been met,
  // end the simulation
  if( cycleCount == 0 ){
    primaryComponentOKToEndSim();
    return true;
  }else{
    return false;
  }
}
```

### Building and Testing Linked Components

Now that our implementation is complete, we can build, install and begin 
testing our external component/subcomponents.  After installing 
via `make && make install`, we can see our new port configurations 
in the `sst-info` output.  Notice that our `basicMsg` subcomponent 
now contains a single port definition to represent `msgPort`.

```
$> sst-info basicComponent
Filtering output on Element = "basicComponent"
================================================================================
ELEMENT 0 = basicComponent ()
Num Components = 1
      Component 0: basicClock
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 3
            STATISTIC 0 = EvenCycleCounter [Counts even numbered cycles] (unitless) Enable level = 1
            STATISTIC 1 = OddCycleCounter [Counts odd numbered cycles] (unitless) Enable level = 2
            STATISTIC 2 = HundredCycleCounter [Counts every 100th cycle] (unitless) Enable level = 3
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 1
            SUB COMPONENT SLOT 0 = msg (Message Interface) [SST::basicComponent::basicMsg]
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = clockTicks (Number of clock ticks to execute) [500]
    basicClock: basicClock: simple clocked component
    Using ELI version 0.9.0
    Compiled on: Apr 11 2022 12:48:13, using file: basicComponent.h
Num SubComponents = 1
      SubComponent 0: basicMsg
      Interface: SST::basicComponent::basicSubcomponent
         NUM STATISTICS = 0
         NUM PORTS = 1
            PORT 0 = msgPort (Link to another component. Uses an event handler to capture events.)
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 2
            PARAMETER 0 = clockFreq (Frequency of period (with units) of the clock) [1GHz]
            PARAMETER 1 = count (Wait for 'count' integers before flushing a message) [5]
    basicMsg: basicMsg : simple message handler subcomponent
    Using ELI version 0.9.0
    Compiled on: Apr 11 2022 12:48:13, using file: basicComponent.h
Num Modules = 0
Num SSTPartitioners = 0
```

Now that we're ready to begin simulation testing, we need to create a simulation 
configuration that contains two components, each with unique subcomponents connected 
via their respective message ports.  For this, we utilize the same basic simulation 
methods above, but create two unique component+subcomponent pairs.  We also 
create a single link, `link0` and connect the `msgPort` from the `msgComp0` 
and `msgComp1` subcomponents using a 1ns latency.

```
#
# basicClock.py
#

import os
import sst

clockComponent0 = sst.Component("ClockComponent0", "basicComponent.basicClock")
clockComponent0.addParams({
  "clockFreq"  : "1.5Ghz",
  "clockTicks" : 1000
})

msgComp0 = clockComponent0.setSubComponent("msg", "basicComponent.basicMsg")
msgComp0.addParams({
  "clockFreq"  : "1.5Ghz",
  "count" : 10
})

clockComponent1 = sst.Component("ClockComponent1", "basicComponent.basicClock")
clockComponent1.addParams({
  "clockFreq"  : "1.5Ghz",
  "clockTicks" : 1000
})

msgComp1 = clockComponent1.setSubComponent("msg", "basicComponent.basicMsg")
msgComp1.addParams({
  "clockFreq"  : "1.5Ghz",
  "count" : 10
})

compLink = sst.Link("link0")
compLink.connect( (msgComp0, "msgPort", "1ns"), (msgComp1, "msgPort", "1ns") )

sst.setStatisticLoadLevel(3)
sst.setStatisticOutput("sst.statOutputCSV")
sst.enableAllStatisticsForComponentType("basicComponent.basicClock")
```

We can now test our simulation configuration.  Using the simulation script 
above, we can see that our components are generated pseudorandom integers 
and exchanging them via the configured links.

```
$> sst basicClock.py
Registering clock with frequency=1.5Ghz
Registering subcomponent clock with frequency=1.5Ghz
Registering clock with frequency=1.5Ghz
Registering subcomponent clock with frequency=1.5Ghz
Flushing message buffer at clock=10
Sent Buffer[0]: 1071494452
Sent Buffer[1]: 1803936666
Sent Buffer[2]: -1672778268
Sent Buffer[3]: 902491587
Sent Buffer[4]: -449790690
Sent Buffer[5]: 693997229
Sent Buffer[6]: -319940607
Sent Buffer[7]: -1946751616
Sent Buffer[8]: -725729641
Sent Buffer[9]: -1926751637
Flushing message buffer at clock=10
Sent Buffer[0]: 1071494452
Sent Buffer[1]: 1803936666
Sent Buffer[2]: -1672778268
Sent Buffer[3]: 902491587
Sent Buffer[4]: -449790690
Sent Buffer[5]: 693997229
Sent Buffer[6]: -319940607
Sent Buffer[7]: -1946751616
Sent Buffer[8]: -725729641
Sent Buffer[9]: -1926751637
Received message buffer at clock7670
Received buffer[0]: 1071494452
Received buffer[1]: 1803936666
Received buffer[2]: -1672778268
Received buffer[3]: 902491587
Received buffer[4]: -449790690
Received buffer[5]: 693997229
Received buffer[6]: -319940607
Received buffer[7]: -1946751616
Received buffer[8]: -725729641
Received buffer[9]: -1926751637
Received message buffer at clock7670
...
Clock cycles: 1000, SimulationCycles: 667000, Simulation ns: 667
Flushing message buffer at clock=1000
Sent Buffer[0]: -1391397147
Sent Buffer[1]: -1286613259
Sent Buffer[2]: 1314549343
Sent Buffer[3]: 1225175738
Sent Buffer[4]: 547476592
Sent Buffer[5]: -1032637418
Sent Buffer[6]: 135824697
Sent Buffer[7]: -2059975138
Sent Buffer[8]: 1103774199
Sent Buffer[9]: -563594638
Clock cycles: 1000, SimulationCycles: 667000, Simulation ns: 667
Flushing message buffer at clock=1000
Sent Buffer[0]: -1391397147
Sent Buffer[1]: -1286613259
Sent Buffer[2]: 1314549343
Sent Buffer[3]: 1225175738
Sent Buffer[4]: 547476592
Sent Buffer[5]: -1032637418
Sent Buffer[6]: 135824697
Sent Buffer[7]: -2059975138
Sent Buffer[8]: 1103774199
Sent Buffer[9]: -563594638
Simulation is complete, simulated time: 667 ns
```

## Integration Testing & Integration with PyBind <a name="IntegrationTesting"></a>

### Introduction

As we've seen in our previous tutorial material, SST provides a rich 
set of Python interfaces to build and configuration simulations using 
in-situ and external SST components.  However, when constructing complex 
simulations, these interfaces may require a great deal of manual Python 
development in order to build components, subcomponents and links.  SST 
provides a convenient method by which to extend the base set of Python 
interfaces using PyBind with component-specific interfaces that can be imported and 
called from Python simulation scripts.  Current SST elements such as 
`merlin`, `firefly` and `ember` are great examples of how elements 
can provide Python-centric interfaces for configuring complex simulations.  

This portion of our tutorial will extend our `basicComponent::basicClock` 
component infrastructure with additional PyBind interfaces and demonstrate 
how these interfaces can be used to simplify simulation configurations.  The process 
of building such interfaces includes the following steps:

* *Component Class Definition*: This step requires that we define an additional 
class that registers our component with the SST PyBind interfaces
* *Python ELI Information*: Using the new component class, we inject ELI macros 
that initiate the registration of the external component's functionality our PyBind 
environment
* *Python Module Development*: Using the existing SST Python interfaces, we develop 
a component-specific Python module that is imported into our external element library.
* *Build Environment*: In order to correctly import the module into the library, 
we will need to edit our existing makefile such that our Python module is imported 
into the C++ class
* *Testing*: Finally, we can develop a new simulation input using our PyBind interfaces 
and test our external component library

### Component Class Definition

The first step in building PyBind integration for our external component 
is to create an inherited `SSTElementPythonModule` class.  This class 
performs two main functions.  First, it imports the Python module 
developed below into the main library and it registers the PyBind interfaces 
with the SST Core.  Once installed, this will provided the external component's 
Python interfaces to Python configuration scripts.

For this we will create a new file in the `src` directory called 
`libbasicComponent.cc`.  This will contain only the class definition and logic 
required for our PyBind implementation.  This file requires that we include 
three header files that represent the `sst_config` header, the `element_python` 
header (which provides all of the PyBind interface logic) and the main header 
for our `basicComponent`.  Next, we create the same namespace infrastructure 
as was defined in our `basicComponent` header.  This allows us to access 
all the functional logic from the `basicClock` class as well as all of the 
relevant ELI macros defined in our main header.

The next step is to define a character array that will contain the hexadecimal 
representation of our Python script developed below.  This variable (`pybasicclock`), 
defined outside of the `basicClockPyModule` is key to importing all the PyBind 
interfaces into the shared library object such that the SST Core can access 
the Python methods without installing the original `*.py` script.  Note that 
we will *not* explicitly develop the `pybasicclock.inc` file as this will 
be generated by our modified makefile below.

Now that we have a container for our Pythin infrastructure, we need to create 
an inherited class from `SSTElementPythonModule` that contains a single 
constructor method.  The constructor is required to define a single parameter 
that defines the name of the library.  Within the constructor, we are also required 
to utilize a single method.  The `createPrimaryModule` creates the top-level 
python module that links to the `sst.library` Python core.  This call requires 
two arguments.  The first is our character array containing the hex-based 
Python code.  The second contains a string that defines the name of the 
Python module script developed below.

```
//
// _libbasicComponent_cc_
//

#include <sst/core/sst_config.h>
#include "basicComponent.h"

// Install the python library
#include <sst/core/model/element_python.h>

namespace SST {
namespace basicComponent {

char pybasicclock[] = {
#include "pybasicclock.inc"
  0x00};

// --------------------------------------
// basicClockPyModule Class
// --------------------------------------
class basicClockPyModule : public SSTElementPythonModule {
public:
  // Constructor
  basicClockPyModule(std::string library) :
    SSTElementPythonModule(library) {
    auto primary_module = createPrimaryModule(pybasicclock,
                                              "pybasicclock.py");
  }

  // class, library, version
  SST_ELI_REGISTER_PYTHON_MODULE(
    SST::basicComponent::basicClockPyModule,
    "basicComponent",
    SST_ELI_ELEMENT_VERSION(1,0,0)
  )

  SST_ELI_EXPORT(SST::basicComponent::basicClockPyModule)
}; // basicClockPyModule
} // namespace basicComponent
} // namespace SST
```

### Python ELI Information

Now that we have the basic `libbasicComponent.cc` class infrastructure in place, 
we need to add two ELI macros in order to correctly register our class 
and PyBind infrastructure with the SST core.  The first, 
`SST_ELI_REGISTER_PYTHON_MODULE`, has three arguments that represent:

* *class* : Name of the class that is being registered. This should not be 
quoted, as it will be used in a template expansion
* *library* : A string containing the name of the library (`basicComponent`)
* *version* : Element version using the standard `SST_ELI_ELEMENT_VERSION` macro

The second ELI macro, `SST_ELI_EXPORT`, requires a single argument in the form 
of the fully qualified C++ class of our PyBind class infrastructure.  This registers 
the repsective class with the SST Core as an exported interface.

### Python Module Development

The next step in our PyBind development is to generate a Python module script 
that implements any component-specific logic.  This will be implemented using 
a Python class that contains some number of internal methods.  There are a 
few basic requirements in developing this module infrastructure, noted 
as follows:

* The Python script must import the `sst` Python interfaces.  It can also 
import other standard Python module interfaces, such as `sys`.
* The class must define an `__init__` method.  This method is not required 
to perform any critical logic, but must be present.
* All additional methods must include the `self` variable as an argument.  Optionally, 
methods may include other data items to define simulation-specific parameter 
inputs

For our example PyBind interface, we seek to generate a two-component simulation 
input that automatically links two subcomponents together.  The clock frequency 
for the components/subcomponents, the number of clock ticks, the buffer counter 
and the link latency will be provided as parameters.  Further, we would like an 
optional component that enables the SST statistics using a parameter that defines 
the desired statistic level.

For our PyBind environment, we create four methods as follows:
* `__init__` : prints a simple message to the console
* `getName`  : returns the name of the PyBind infrastructure
* `enableBasicClockLink` : Initializes two simulation components, each with 
subcomponents.  The subcomponents are connected via a link.  Notice in the 
example provided below that we utilize the basic SST Python interfaces utilized 
in previous examples.  We build components using `sst.Component` and insert 
subcomponents and links accordingly.  This method 
requires the following paramters:
  * `BCFreq` : The clock frequency string for the `basicClock` components
  * `Ticks` : The number of clock ticks for the `basicClock` components
  * `MCFreq` : The clock frequency string for the `basicMsg` subcomponents
  * `MCCount` : The buffer size used for flushing our integer buffer within `basicMsg`
  * `Lantency` : The latency of our link between the `basicMsg` subcomponents
* `enableStatistics` : Initializes the statistics gathering interfaces 
in the SST core for our external components at the desired level.  This method 
requires the following parameter:
  * `level` : defines the statistics level for our components

```
#!/usr/bin/env python
#
# pybasicclock.py
#

import sys
import sst

class basicClock():
    def __init__(self):
      print("Initializing basicClock")

    def getName(self):
      return "basiClockModuleExample"

    def enableBasicClockLink(self, BCFreq, Ticks, MCFreq, MCCount, Latency):
      bcComps = []
      mcComps = []
      for i in range(0, 2):
        print("Building basicClock Component+Subcomponent " + str(i))
        sst.pushNamePrefix("COMP"+str(i))
        bc = sst.Component("ClockComponent", "basicComponent.basicClock")
        bc.addParam("clockFreq", BCFreq)
        bc.addParam("clockTicks", Ticks)
        mc = bc.setSubComponent("msg", "basicComponent.basicMsg")
        mc.addParam("clockFreq", MCFreq)
        mc.addParam("count", MCCount)
        bcComps.append(bc)
        mcComps.append(mc)
        sst.popNamePrefix()
      cl = sst.Link("link0")
      cl.connect( (mcComps[0], "msgPort", Latency), (mcComps[1], "msgPort", Latency) )

    def enableStatistics(self, level):
      print("Enabling statistics")
      sst.setStatisticLoadLevel(level)
      sst.setStatisticOutput("sst.statOutputCSV")
      sst.enableAllStatisticsForComponentType("basicComponent.basicClock")
```

### Build and Installation

Prior to compiling our PyBind example, we need to edit the makefile in the 
`src` directory.  These edits are required to populate the `pybasicclock.inc` 
include file for our PyBind class definition.  The key addition to the makefile 
is the `BASICCOMPONENT_INCS` directive and the `inc` build target that utilizes 
the `od` tool to construct a hexadecimal representation of our Python module 
script.

Once the edits to the makefile are complete, we can build and install our 
example using `make` and `make install`.

```
ifeq (, $(shell which sst-config))
 $(error "No sst-config in $(PATH), add `sst-config` to your PATH")
endif

CXX=$(shell sst-config --CXX)
CXXFLAGS=$(shell sst-config --ELEMENT_CXXFLAGS)
LDFLAGS=$(shell sst-config --ELEMENT_LDFLAGS)
CPPFLAGS=-I./
OPTIMIZE_FLAGS=-O2

COMPONENT_LIB=libbasicComponent.so

BASICCOMPONENT_SOURCES := $(wildcard *.cc)

BASICCOMPONENT_HEADERS := $(wildcard *.h)

BASICCOMPONENT_OBJS := $(patsubst %.cc,%.o,$(wildcard *.cc))

BASICCOMPONENT_INCS := $(patsubst %.py,%.inc,$(wildcard *.py))

all: $(COMPONENT_LIB)

debug: CXXFLAGS += -DDEBUG -g
debug: $(COMPONENT_LIB)

$(COMPONENT_LIB): $(BASICCOMPONENT_INCS) $(BASICCOMPONENT_OBJS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) $(LDFLAGS) -o $@ *.o
%.o:%.cc $(SIMPLEEXAMPLE_HEADERS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) -c $<
%.inc:%.py
        od -v -t x1 < $< | sed -e 's/^[^ ]*[ ]*//g' -e '/^\s*$$/d' -e 's/\([0-9a-f]*\)[ $$]*/0x\1,/g' > $@
install: $(COMPONENT_LIB)
        sst-register basicComponent basicComponent_LIBDIR=$(CURDIR)
clean:
        rm -Rf *.o *.inc $(COMPONENT_LIB)
```

### Module Testing

With our new PyBind enabled library installed, we can create new 
simulation scripts that utilize our PyBind interfaces.  The first 
step in doing is to import the `sst` interfaces and import 
all the interfaces from our external PyBind-enabled library.  The latter 
requires us to insert `from sst.basicComponent import *`.  Note that 
the library name matches the name we set for the library in the 
`SST_ELI_REGISTER_PYTHON_MODULE` ELI macro.

The next step includes creating a `main` function that intializes 
our simulation configuration.  We utilize the `basicClock` method 
as defined by our Python script to create a new object.  Once 
created, we can utilize the `enableBasicClockLink` method with the 
appropriate parameters to create the remainder of our components, 
subcomponents and links.  The PyBind interfaces provided by our 
library handle all the necessary component creation and linkage 
for our simulation model.

```
#
# basicClock_PyModule.py
#

import os
import sst
from sst.basicComponent import *

if __name__ == "__main__":
  myBasicClock = basicClock()
  myBasicClock.enableBasicClockLink("1.5Ghz", 1000, "1.5Ghz", 10, "1ns" )
```

Once complete, we can execute our new simulation script as follows.  Notice 
that we see messages propagating and model creation time from our PyBind 
methods.  The resulting simulation is then executed by the SST Core.

```
$> sst basicClock_PyModule.py
Initializing basicClock
Building basicClock Component+Subcomponent 0
Building basicClock Component+Subcomponent 1
Registering clock with frequency=1.5Ghz
Registering subcomponent clock with frequency=1.5Ghz
Registering clock with frequency=1.5Ghz
Registering subcomponent clock with frequency=1.5Ghz
Flushing message buffer at clock=10
Sent Buffer[0]: 1071494452
Sent Buffer[1]: 1803936666
Sent Buffer[2]: -1672778268
Sent Buffer[3]: 902491587
Sent Buffer[4]: -449790690
Sent Buffer[5]: 693997229
Sent Buffer[6]: -319940607
Sent Buffer[7]: -1946751616
Sent Buffer[8]: -725729641
Sent Buffer[9]: -1926751637
Flushing message buffer at clock=10
Sent Buffer[0]: 1071494452
Sent Buffer[1]: 1803936666
Sent Buffer[2]: -1672778268
Sent Buffer[3]: 902491587
Sent Buffer[4]: -449790690
Sent Buffer[5]: 693997229
Sent Buffer[6]: -319940607
Sent Buffer[7]: -1946751616
Sent Buffer[8]: -725729641
Sent Buffer[9]: -1926751637
...
Clock cycles: 1000, SimulationCycles: 667000, Simulation ns: 667
Flushing message buffer at clock=1000
Sent Buffer[0]: -1391397147
Sent Buffer[1]: -1286613259
Sent Buffer[2]: 1314549343
Sent Buffer[3]: 1225175738
Sent Buffer[4]: 547476592
Sent Buffer[5]: -1032637418
Sent Buffer[6]: 135824697
Sent Buffer[7]: -2059975138
Sent Buffer[8]: 1103774199
Sent Buffer[9]: -563594638
Simulation is complete, simulated time: 667 ns
```

Additionally, if we edit our script and add the `enableStatistics` 
method to our `myBasicClock` object with a level set to *3*, the 
simulation will generate the required statistics output as well.  Note 
that the naming conventions found in the statistics output follow that 
of the PyBind interface methods.

```
#
# basicClock_PyModule.py
#

import os
import sst
from sst.basicComponent import *

if __name__ == "__main__":
  myBasicClock = basicClock()
  myBasicClock.enableBasicClockLink("1.5Ghz", 1000, "1.5Ghz", 10, "1ns" )
  myBasicClock.enableStatistics(3)
```

```
ComponentName, StatisticName, StatisticSubId, StatisticType, SimTime, Rank, Sum.u64, SumSQ.u64, Count.u64, Min.u64, Max.u64, Sum.u32, SumSQ.u32, Min.u32, Max.u32
COMP0.ClockComponent, EvenCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
COMP0.ClockComponent, OddCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
COMP0.ClockComponent, HundredCycleCounter, , Accumulator, 667000, 0, 0, 0, 10, 0, 0, 10, 10, 1, 1
COMP1.ClockComponent, EvenCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
COMP1.ClockComponent, OddCycleCounter, , Accumulator, 667000, 0, 500, 500, 500, 1, 1, 0, 0, 0, 0
COMP1.ClockComponent, HundredCycleCounter, , Accumulator, 667000, 0, 0, 0, 10, 0, 0, 10, 10, 1, 1
```

## References
* [SST ELI Infrastructure](http://sst-simulator.org/SSTPages/SSTDeveloperNewELIMigrationGuide/)
