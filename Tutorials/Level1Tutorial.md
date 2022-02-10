# Level 1 Tutorial

## Table of Contents
1. [What is SST](#WhatIsSST)
2. [Components of SST](#ComponentsOfSST)
3. [Basic Installation](#BasicInstallation)
4. [Basic Commands](#BasicCommands)
5. [Basic Runtime Functions](#BasicRuntimeFunctions)
6. [Collecting Data from SST](#CollectingData)
7. [Links to Documentation](#LinksToDocumentation)

## What is SST: <a name="WhatIsSST"></a>
  To combat the ever-changing and multifaceted technologies of the HPC 
  community, an effort was made to create a unified tool box used to evaluate 
  these challenges. The Structural Simulation Toolkit (SST) an open, modular, 
  parallel, multi-criteria, multi-scale simulation framework  was created to 
  design and procure HPC systems. By providing a number of interfaces and 
  utilizes for simulation models, SST, will demolish the need for simulations 
  for individual system components. A now unified framework exists for parallel 
  simulation of large machines at multiple levels. The SST has been used in 
  variety of network, memory and applications.

## Components of SST: <a name="ComponentsOfSST"></a>
  To meet these requirements, the SST is comprised of a simple simulation core 
  that contains a parallel discrete event simulator (DES) and support services 
  for simulation. components, representing hardware systems such as processors, 
  network switches, or memory devices, interface with the simulation core to 
  communicate and operate with a common notion of timeframe. The simulation 
  core also provides support services such as power and area estimation, 
  checkpointing, configuration/initialization of the simulation, and statistics 
  gathering. The SST’s modular interface eases the integration of existing 
  simulators into a common framework and is licensed under a BSD-like license. 
  In addition, the components of the SST model are compatible with external 
  models such as Gem5 and DRAMSim2. Because of the open source core, the 
  modular framework is easily extensible with new models. 

  The SST uses a (DES) model layered on top of MPI.  The simulations are 
  comprised of components or building blocks of the simulation model are 
  connected by links. These components interact by sending events over 
  links—with each link having a minimum latency. In addition, components can 
  load subcomponents and modules for even more functionality and customization. 
  To achieve better performance, the SST uses a conservative (i.e. 
  no roll-back) distance-based optimization. At the start of the simulation, 
  the system topology is represented by a graph with components as nodes and 
  connections between them as edges, with each edge labeled with the minimum 
  latency between the connected components. The Zoltan library is then used to 
  partition components across the MPI ranks with the goal of balancing the load 
  and partitioning across the highest latency links. Tests indicate that the 
  algorithm is scalable and shows less than 25% overhead at 128 ranks (11,904 
  simulated components) compared to a single rank for detailed simulations. 

![image](https://user-images.githubusercontent.com/74792926/121704586-e5454680-caa1-11eb-8c1f-b33daad81498.png)

## Basic Installation <a name="BasicInstallation"></a>

### Software Prerequisites

SST can be built on a variety of Linux and OSX platforms.  Currently, Windows 
is not a supported platform.  For each of these systems, there are a number 
of prerequisite software packages that need to be installed prior to building 
SST from source.  This includes the following:

* C/C++ Compiler (GCC,Clang)
* GNU Autotools
* OpenMPI (4.0.5 is preferred)
* Python 3.X+
* Git (if cloning the source tree)

### SST-Core Installation

Once the required software packages have been installed, you can either download 
the SST-Core package from the SST [release page](http://sst-simulator.org/SSTPages/SSTMainDownloads/) 
or check out the source code from [Github](https://github.com/sstsimulator/sst-core).

If you download the release package, untar it as follows:
```
$> tar xzvf sstcore-X.Y.Z.tar.gz
$> cd sstcore-X.Y.Z
```

If you seek to clone the source repository directly, do the following.  Note 
the final autogen command to generate the required configuration scripts.  This 
is only required if you clone the source repository directly.
```
$> git clone https://github.com/sstsimulator/sst-core.git sst-core
$> cd sst-core
$> ./autogen.sh
```

Now that you have the SST-Core source, you should select an appropriate 
installation location.  For this example, we will utilize `/opt/sst` as our 
installation target.  At this point, we can configure, build and install 
the SST-Core infrastructure as follows:
```
$> mkdir /opt/sst
$> ./configure --prefix=/opt/sst
$> make -j
$> make install
$> export PATH=$PATH:/opt/sst/bin
$> sst --version
```

In addition to the aforementioned basic build instructions, there are a number 
of additional configuration options that can be specified.  A short summary 
of these additional options are listed as follows:

| **Option**  | **Description**  |
|:-|:-|
| `--with-hdf5=/path/to/HDF5` | Enables support for HDF5 statistics output |
| `--disable-mpi`             | Disables MPI (only support for single node |
| `--disable-mem-pools`       | Disables memory pools |
| `--enable-debug`            | Enables debug mode  |
| `--enable-event-tracking`   | Enables event tracking for debug mode  |
| `--enable-profile`          | Enables performance profiling of core features  |

For more information regarding these options and other options, do the following:
```
$> ./configure --help
```

### SST-Elements Installation

Now that the SST-Core has been installed, we can build the SST-Elements package.  
The SST-Elements package provides individual *components* that simulate specific 
pieces of hardware.  The SST-Elements package provides a number of pathological 
hardware elements such as memories, caches, processor cores and network interfaces.  
It also provides a number of examples on how to construct larger simulations of 
full systems.  The SST-Elements provides a number of components that are built 
by default as well as a number of components that require external libraries.  
For the purposes of this tutorial, we will build the SST-Elements package 
with the default options.

Much in the same manner as the SST-Core package, we can select the latest release 
package of SST-Elements or build directly from the git source tree.  
The standard SST-Elements release packages are available on the 
[release page](http://sst-simulator.org/SSTPages/SSTMainDownloads/)
or check out the source code from [Github](https://github.com/sstsimulator/sst-elements).

If you download the release package, untar it as follows:
```
$> tar xzvf sstelements-X.Y.Z.tar.gz
$> cd sstelements-X.Y.Z
```

If you seek to clone the source repository directly, do the following.  Note 
the final autogen command to generate the required configuration scripts.  This 
is only required if you clone the source repository directly.
```
$> git clone https://github.com/sstsimulator/sst-elements.git sst-elements
$> cd sst-elements
$> ./autogen.sh
```

Now that you have the SST-Elements source, you should select an appropriate 
installation location.  For this example, we will utilize `/opt/sst/elements` as our 
installation target.  At this point, we can configure, build and install 
the SST-Elements infrastructure as follows:
```
$> mkdir /opt/sst/elements
$> ./configure --prefix=/opt/sst/elements --with-sst-core=/opt/sst
$> make -j
$> make install
$> sst-info -q
```

At this point in the tutorial, we have a functional SST environment 
with both the base SST-Core and the SST-Elements packages installed.

## Basic Commands <a name="BasicCommands"></a>

There are a number of basic commands that are commonly utilized when 
running and/or debugging an SST simulation.  We summarize the most 
commonly used command sets here and the situations where they are 
appropriately utilized.  However, all of the major SST command line tools 
provide a `--help` option to print useful command line options 
and their associated parameters.  

### SST Execution and Utility Functions

* _Executing a Simulation_: The most common command line tool is the `sst` command.  
This is utilized to build and execute a simulation run using the SST Core and associated 
Elements libraries.  Using the `sst` command line tool and passing a loadable 
simulation graph using Python or JSON can be performed as follows:
```
$> sst /path/to/simulation.py
$> sst /path/to/simulation.json
```

* _Executing with Verbosity_: Similar to the command above, users may enable 
verbose output of the graph construction and simulation by passing the `-v`
or `--verbose=LEVEL` option.  The higher the `LEVEL` specified, the more verbose
the output will be.  This is often useful for debugging new simulation configurations.  
```
$> sst -v /path/to/simulation.py
$> sst --verbose=4 /path/to/simulation.py
```

* _Testing Simulation Graph Syntax_: One of the common issues that users encounter 
when constructing complex simulation graphs is basic syntactic issues in their 
respective input files that prevent SST from constructing a valid simulation.  It is 
often useful to utilize the SST graph loader(s) to validate a graph using the 
internal initialization functions without actually executing the simulation.  
This can be performed using the following command.  Note that SST will report 
that the simulation completed, but the reported simulation time will be 0s.
```
$> sst --run-mode=init /path/to/simulation.py
Simulation is complete, simulated time: 0 s
```

* _Outputting DOT Configuration Graphs_: It is also often useful for debugging 
or publication purposes to direct SST to output a GraphViz DOT file that contains 
the configuration graph of the candidate simulation.  This can be performed 
with or without actually executing the simulation.  We show examples for both 
as follows:
```
$> sst --output-dot=foo.dot /path/to/simulation.py
$> sst --run-mode=init --output-dot=foo.dot /path/to/simulation.dot
$> dot -Tpdf foo.dot > foo.pdf
```

* _Converting Between Simulation File Types_: One of the latest features 
provided by SST is the ability to develop simulation inputs using Python 
or JSON syntax.  Python is often useful when developing large simulations 
that require redundant components generated (multiple CPUs with 
memory hierarchys).  However, JSON inputs are often 
easier to discern the respective graph 
connectivity given the inherent syntax of JSON.  SST provides an internal 
feature that permits users to specify in a graph in one format and convert 
it to another format (with or without executing the simulation).  Further, 
users may also specify large, complex graph inputs and direct SST to 
output the same graph in a more expanded form.  This can also be useful 
for debugging complex simulations as the graph output will be the raw 
form of what SST utilized for the actual graph connectivity.  Various 
forms of inputting and outputting graphs are shown below (all without 
actually executing the simulation).
```
$> sst --run-mode=init --output-json=foo.json /path/to/simulation.py
$> sst --run-mode=init --output-config=foo.py /path/to/simulation.json
$> sst --run-mode=init --output-config=foo.py /path/to/complex/simulation.py
```

### SST Information
The SST infrastructure may potentially contain a large number of 
available components for a target installation.  Each of the available 
components may have a large number of available configuration options 
and/or contain nested sets of subcomponent configuration options.  In order 
to assist users to discern the required or optional configuration parameters 
for components or subcomponents, SST provides the `sst-info` command.  This 
command outputs the relevant component and subcomponent parameters, options 
and statistics.  There are a number of potential methods for querying 
the configured components.  We details these as follows:

* _Querying All Components_: The first method is to simply query all known 
configured components.  By simply executing the `sst-info` command, 
the SST-Core will output all the configuration parameters, statitics and nested 
subcomponents for all the known components.  This includes any component built 
within the installed SST package as well as any user-registered components.
```
$> sst-info
```

* _Querying Specific Components_: If you seek to query the information for 
specific components, `sst-info` provides a command line option to specify 
the name of the component or subcomponent to narrow the search results.  This 
can be done be specifying the component and/or the component/subcomponent 
combination.  Examples of doing this are as follows using the `miranda` 
component.
```
$> sst-info miranda
$> sst-info miranda.CopyGenerator
```

An example of the output from the `miranda.CopyGenerator` command from above 
is as follows:
```
PROCESSED 1 .so (SST ELEMENT) FILES FOUND IN DIRECTORY(s) /path/to/sst-elements-library
Filtering output on Element = "miranda.CopyGenerator"
================================================================================
ELEMENT 0 = miranda ()
Num Components = 1
Num SubComponents = 11
      SubComponent 0: CopyGenerator
      Interface: SST::Miranda::RequestGenerator
         NUM STATISTICS = 0
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 5
            PARAMETER 0 = read_start_address (Sets the start read address for this generator) [0]
            PARAMETER 1 = write_start_address (Sets the start target address for writes for the generator) [1024]
            PARAMETER 2 = request_size (Sets the size of each request in bytes) [8]
            PARAMETER 3 = request_count (Sets the number of items to be copied) [128]
            PARAMETER 4 = verbose (Sets the verbosity of the output) [0]
    CopyGenerator: Creates a single copy of stream of reads/writes replicating an array copy pattern
    Using ELI version 0.9.0
    Compiled on: Dec 16 2021 11:12:05, using file: generators/copygen.h
Num Modules = 0
Num SSTPartitioners = 0
```

### SST Configuration
Finally, the `sst-config` tool can be utilized by system administrators and 
component developers to resolve the respective SST configuration and compilation 
environment.  The `sst-config` tool has the ability to report information 
regarding which components were built with the target installation, which 
compilers and libraries were utilized as well as the respective compilation 
options required to successfully build and link out-of-tree components against 
the target installation.  We describe several of the most common options as follows:

* _Print the Entire Configuration_: Utilizing the raw `sst-config` command with 
no options will print all data stored with the respective installation.  This includes 
locations of libraries, compiler utilized to build SST and any user-installed 
(out-of-tree) components.  This can be done as follows:
```
$> sst-config
```

* _Print the C++ Compiler_:  For those seeking to develop external components, 
it is often useful to derive the target compiler and arguments utilized to build 
the SST-Core.  This ensures that the internal SST loading functions have the 
ability to load and execute external components packaged as shared libraries.  This 
can be done using the `sst-config` tool as follows:
```
$> sst-config --CXX
$> sst-config --ELEMENT_CXXFLAGS
$> sst-config --ELEMENT_LDFLAGS
```

A useful example of integrating these commands into an external Makefile resembles 
the following:
```
ifeq (, $(shell which sst-config))
 $(error "No sst-config in $(PATH), add `sst-config` to your PATH")
endif

CXX=$(shell sst-config --CXX)
CXXFLAGS=$(shell sst-config --ELEMENT_CXXFLAGS)
LDFLAGS=$(shell sst-config --ELEMENT_LDFLAGS)
CPPFLAGS=-I./
OPTIMIZE_FLAGS=-O3

FOO_SOURCES := $(wildcard *.cc)

FOO_HEADERS := $(wildcard *.h)

FOO_OBJS := $(patsubst %.cc,%.o,$(wildcard *.cc))

all: libfoo.so

debug: CXXFLAGS += -DDEBUG -g
debug: libfoo.so

libfoo.so: $(FOO_OBJS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) $(LDFLAGS) -o $@ *.o
%.o:%.cc $(FOO_HEADERS)
        $(CXX) $(OPTIMIZE_FLAGS) $(CXXFLAGS) $(CPPFLAGS) -c $<
install: libfoo.so
        sst-register foo foo_LIBDIR=$(CURDIR)
clean:
        rm -Rf *.o libfoo.so
```

## Basic Runtime Functions <a name="BasicRuntimeFunctions"></a>

As we see from the commands above, executing a simulation with SST 
is as simple as passing a simulation script or JSON file to the `sst` 
command.  However, in order to do so, we must first develop an SST 
simulation in Python or JSON format.  In this section, we will provide 
a basic overview of utilizing Python to create a sample simulation 
input file.  Additional, advanced Python scripting topics will be discussed 
in the Level 2 Tutorial.

Each Python-based simulation input script has requires at least three 
key sections as outlined below:
* **Python imports**: Each simulation input file must import the `sst` Python
library.  This provides the fundamental component configuration options 
and functions that link directly to the SST-Core and any configured elements.
* **SST Core Options**: The core options section describes the top-level configuration 
options for the SST-Core.  This includes items such as the SST program options 
that set the baseline clock candence and simulation runtime.
* **SST Component Configuration**: The SST component configuration section contains 
the bulk of the information in a Python simulation script.  This includes the 
definition, parameters and any required subcomponents for each target component 
in the simulation.

The following example is a simple SST configuration script that imports 
the required SST functions, sets two SST core program options and initializes 
a single SST component.  The first step in creating a new simulation input 
is to utilize the `import sst` statement.  This ensures that all the required 
Python functions for SST are enabled.  Users may also utilize additional Python 
import statements in order to enable additional Python-derived functionality 
within the simulation script.  In this manner, simulation scripts are **not** 
limited to SST functionality.  All normal Python syntatic operations may 
be utilized as with a traditional Python script.  

The next section includes the SST Core Options.  These options drive the 
top-level simulation configuration that are utilized across components.  Each 
of the core program options are set with the Python function 
`sst.setProgramOptions("Option", "Value)`.  The two most commonly 
used program options are the `timebase` and `stop-at` options.  The 
`timebase` option sets the baseline global clock cadence by which 
SST will govern the core simulation components.  Each component has the ability 
to set higher or lower fidelity clocks governing local event timing, but the 
global clock drives the top-level simulation cadence.  Further, the `stop-at` 
program option determines when the simulation will complete reletive 
to the core timebase clock frequency.  In our example below we set a 
`timebase` of 1 picosecond and inform SST to stop the simulation at 10000 
seconds of simulated time.  Users may also specify a value of `0` 
for the `stop-at` program option.  This tells the SST core to wait until 
all components and sub-components have signaled completion to the core 
before stopping the simulation.  Note that for these options, 
SST automatically interprets the labeled units.  In this manner, users 
may utilize traditional labels such as `ns`, `us`, `ms` and `s` to specify 
timing values in nanoseconds, microseconds, milliseconds and seconds, 
respectively.  This is also true for specifying traditional measures 
of capacity (`GB`,`KB`, etc).  

The full list of permissible global program options are listed as follows:

|  **Option**  | **Description** | **Values** |
|:-|:-|:-|
| `timebase`          | The baseline clock cadence | Time (`ps`,`ns`,etc) |
| `stop-at`           | Simulation termination point | Simulated time (`ms`, `s`, etc) |
| `debug-file`        | Specifies the file for debug outptu | Filename |
| `partitioner`       | Select the graph partitioner to use | `lib.partitionerName` |
| `verbose`           | Print information about the core runtime | Boolean (`True`, `False`)|
| `dump_partition`    | Output the SST paritioned graph info | Boolean (`True`, `False`) |
| `dump_config_graph` | Output the SST config graph | Boolean (`True`, `False`) |
| `model_options`     | Provide options to the Python script | String |
| `run-mode`          | Set the run mode | `init`, `run` or `both` |

The final required section in the SST simulation script involves the definition 
and configuration of one or more components.  The process of defining a component 
involves two steps: Defining the component object and adding parameters 
to the object. The first requires the use of the `sst.Component` Python 
function.  This function requires two arguments, one to specify the 
internal name for the component object and the second to specify the 
component library to load for the target object.  In the example below, 
we create an object instance `comp_clocker0` with the internal SST name 
of `clocker0` using the `coreTestElement.simpleClockerComponent` component 
library.  As we elicited above, you can utilize the `sst-info` command line 
tool to output the available libraries, their components, subcomponents 
and associated parameters.  For example, using the following command 
will show the contents of the `coreTestElement` element library, including 
the `simpleClockerComponent` component (Component 1).

```
$> sst-info coreTestElement

PROCESSED 1 .so (SST ELEMENT) FILES FOUND IN DIRECTORY(s) /scratch/sst/sst-11.1.0/lib/sstcore:/scratch/sst/s
st-11.1.0/elements/lib/sst-elements-library
Filtering output on Element = "coreTestElement"
================================================================================
ELEMENT 0 = coreTestElement ()
Num Components = 10
      Component 0: SubComponentLoader
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 1
            STATISTIC 0 = totalSent [# of total messages sent] () Enable level = 1
         NUM PORTS = 1
            PORT 0 = port%d (Sending or Receiving Port(s))
         NUM SUBCOMPONENT SLOTS = 1
            SUB COMPONENT SLOT 0 = mySubComp (Test slot) [SST::CoreTestSubComponent::SubCompInterface]
         NUM PARAMETERS = 3
            PARAMETER 0 = clock (Clock Rate) [1GHz]
            PARAMETER 1 = unnamed_subcomponent (Unnamed SubComponent to load.  If empty, then a named subcom
ponent is loaded) []
            PARAMETER 2 = num_subcomps (Number of anonymous SubComponents to load.  Ignored if using name Su
bComponents.) [1]
    SubComponentLoader: Demonstrates subcomponents
    Using ELI version 0.9.0
    Compiled on: Nov  2 2021 08:52:35, using file: ../../../src/sst/core/testElements/coreTest_SubComponent.
h
      Component 1: coreTestClockerComponent
      CATEGORY: UNCATEGORIZED COMPONENT
         NUM STATISTICS = 0
         NUM PORTS = 0
         NUM SUBCOMPONENT SLOTS = 0
         NUM PARAMETERS = 2
            PARAMETER 0 = clock (Clock frequency) [1GHz]
            PARAMETER 1 = clockcount (Number of clock ticks to execute) [100000]
    coreTestClockerComponent: Clock Benchmark Component
    Using ELI version 0.9.0
    Compiled on: Nov  2 2021 08:52:35, using file: ../../../src/sst/core/testElements/coreTest_ClockerCompon
ent.h
[the remainder of the output was truncated]
```

Now that we have our component object created using the `sst.Component` interface, 
the second step in the component creation process is to configure parameters 
for the component object.  The SST Python interface provides a convenient function 
for initializing component parameters in the form.  A single call to `addParams` 
with a list of component option and value pairs can be peformed.  Each option 
name is a quoted string and each parameter value can be a quoted string, an integer 
value or a boolean.  In our example, we provide two parameters, `clockcount` and 
`clock` for the `comp_clocker0` object.  Also note from our `sst-info` example 
above, we can see that the `coreTestElement.simpleClockerComponent` contains 
the two target component parameters alongside their default values (in brackets).  

### Example Python Simulation Script
```
# Automatically generated SST Python input
import sst

# Define SST core options
sst.setProgramOption("timebase", "1 ps")
sst.setProgramOption("stop-at", "10000s")

# Define the simulation components
comp_clocker0 = sst.Component("clocker0", "coreTestElement.simpleClockerComponent")
comp_clocker0.addParams({
      "clockcount" : "100000000",
      "clock" : "1MHz"
})

# End of generated output.
```
*Note that this simulation script can be found in the SST-Core source tree at:
~/tests/test_ClockerComponent.py*

## Collecting Data From SST <a name="CollectingData"></a>

Now that we have a basic ability to create functional simulation input 
scripts, we can add additional script options to generate useful output data. 
The output data created from SST is generated from the SST *statistics*.  These 
statistics values are generated for each instantiated component object and output 
to a chosen destination at the end of a simulation (console, file, etc).  The 
statistics output data can be utilized to analyze the simulation results.  Each 
component has the ability to define and output a unique set of statistics values 
based upon the functionality implemented in the target component.  For example, 
network components may output statistics such as the number of bytes transfered 
or the average size of a network packet.  Conversely, cpu components may track 
the number of instructions retired, or the number of specific instructions execute. The
output formats provided by the SST core utilities provide convenient file 
formats such as *csv* files in order to make downstream data analysis more efficent.

The process of enabling and utilizing statistics in an SST simulation script requires 
two steps as follows:
* **Statistic Load Level** : Setting the statistic load level involves enabling 
the statistics functions inside the SST core to collect and disseminate data 
to the target output medium.
* **Statistic Output** : Setting the statistic output information initializes 
the target data output medium after the simulation has completed.  

The first step in utilizing the statistics requires that the user specify 
the statistic *load level*.  The load level specifies different levels of 
statistics to be enabled for data collection within the target element objects.  
Generally speaking, the higher the level, the more data will be generated.  Setting 
the level to "0" will disable the statistics (which is also the default action).  Setting 
the statistic load level is a simple processs of adding a single sst function call 
to the Python input script as follows.  This is generally added before creating 
any element objects.

```
sst.setStatisticLoadLevel(7)
```

Once you have enabled the statistics within the simulation script, you may 
now configure the output options for the generated statistics.  By default, 
the statistics will be output to the console.  This can be explicitly 
set using the `sst.statOutputConsole()` function in the simulation script. However, 
users often prefer to output the data to a useful text file that can be parsed 
for downstream analysis.  The commonly utilized output file format is *csv*, or 
comma separated value files.  The SST python interfaces provide a set of functions
to enable the CSV (or other output formats) and set the required parameters.  Each 
output enabler function is formatted as `sst.statOutputFORMAT` where `FORMAT` 
is one of `CONSOLE`, `TXT`, `CSV`, `TXTgz` and `CXVgz`.  Each of these functions 
can take one or more parameters that define things such as the file path, 
the output header information and the target MPI rank for which to output 
the data from.  In our example below, we set the output to utilize *csv* 
files, the file path to `TestOutput.csv` and the CSV separator character to 
a comma.  As with the componenet parameters, statistics parameters can be 
specified using standard Python syntax.  The full list of potential options 
by output type is listed as follows:

|  **Output Type**  | **Parameter** | **Description** |
|:-|:-|:-|
| `sst.statOutputConsole` | | Outputs to the console |
| `sst.statOutputTxt`     | | Outputs to a text file |
|                         | *filepath* | Path to the output file |
|                         | *outputtopheader* | If set, output a header at start of simulation providing name of each unique registered field of all statistics. Default = 1. |
|                         | *outputinlineheader* | If set, output field names inline with the data. Default = 1. |
|                         | *outputsimtime* | If set, output simulation time as a field. Default = 1. |
|                         | *outputrank* | If set, output rank a field. Default = 1 |
| `sst.statOutputCSV`     | | Outputs to a CSV file |
|                         | *separator* | The separator between fields. Default is “, “ |
|                         | *filepath* | Path to the output file |
|                         | *outputtopheader* | If set, output a header at start of simulation providing name of each unique registered field of all statistics. Default = 1. |
|                         | *outputsimtime* | If set, output simulation time as a field. Default = 1. |
|                         | *outputrank* | If set, output rank a field. Default = 1 |
| `sst.statOutputTXTgz`   | | Outputs to a compressed txt file |
|                         | *filepath* | Path to the output file |
|                         | *outputtopheader* | If set, output a header at start of simulation providing name of each unique registered field of all statistics. Default = 1. |
|                         | *outputinlineheader* | If set, output field names inline with the data. Default = 1. |
|                         | *outputsimtime* | If set, output simulation time as a field. Default = 1. |
|                         | *outputrank* | If set, output rank a field. Default = 1 |
| `sst.statOutputCSVgz`   | | Outputs to a compressed CXV file |
|                         | *separator* | The separator between fields. Default is “, “ |
|                         | *filepath* | Path to the output file |
|                         | *outputtopheader* | If set, output a header at start of simulation providing name of each unique registered field of all statistics. Default = 1. |
|                         | *outputsimtime* | If set, output simulation time as a field. Default = 1. |
|                         | *outputrank* | If set, output rank a field. Default = 1 |

Now that we've enabled the statistic levels and the output formats, we can enable specific 
statistics for specific component objects.  The SST core Python infrastructure 
provides a simple interface for all components.  The `enableStatistics` function 
provides users the ability to enable specific components and their associated 
parameters.  The SST core also provides interfaces to enable all statistics 
on certain components, enable statistics on specific classes of component and 
all statistics on all components.  Note that each of the functions can only 
be called *after* the component objects are actually created.  In this manner, 
the statistics functions are *applied* to one or more of the created component 
objects.  Each of the statistics that reside within each component may have 
optional parameters enabled to in order to enable/disable or modify how/when 
the statistics are generated.  However, all statistics have a default set of 
parameters that are enabled across all components.  These include the following:

|  **Parameter**  | **Description** | **Default Value** |
|:-|:-|:-|
| `rate`   | Identifies the output rate of the component | 0 |
| `type`   | Identifies the type of the statistic | `sst.AccumulatorStatistic` |
| `startat`| Identifies a time in the simulation when to enable the statistic. | `0ns` (enabled at startup) |
| `stopat` | Identifies a time in the simulation when to disable the statistic.| `0ns` (not disabled until sim completion) |

For the `type` parameter, SST provides four unique collection types.  Each 
collection type stores and reports data in a unique manner.  The most 
commonly used type is the `sst.AccumulatorStatistic` type that accumulates 
data into a single value.  The `sst.HistogramStatistic` creates a histogram of 
the collected data.  The `sst.UniqueCountStatistic` counts unique entries 
from the collected data and the `sst.NullStatistic` effectively does nothing.  Additional 
parameters for each of the statistics types is provided as follows:

|  **Type**  | **Parameter** | **Description** |
|:-|:-|:-|
| `sst.AccumulatorStatistic` | | |
| | *rate* | User defined. if not provided or 0, output at end of simulation. |
| | *startat* | User defined. If set to a time value, the statistic will start disabled, and then be enabled at the set time. If not provided or 0, statistic will start enabled. |
| | *stopat* | User defined. If set to a time value, the statistic will be disabled at the set time. If not provided or 0, then no change in enable will occur. |
| `sst.HistogramStatistic`   | | |
| | *rate* | User defined. if not provided or 0, output at end of simulation. |
| | *startat* | User defined. If set to a time value, the statistic will start disabled, and then be enabled at the set time. If not provided or 0, statistic will start enabled. |
| | *stopat* | User defined. If set to a time value, the statistic will be disabled at the set time. If not provided or 0, then no change in enable will occur. |
| | *minvalue* | The lowest value to be binned. |
| | *binwidth* | How wide the bins are. |
| | *numbins* | The number of bins desired. |
| | *dumpbinsonoutput* | Output the bin counts during an output (default = on) |
| | *includeoutofbounds* | Provide a Out of Bounds Low Bin and Out of Bounds High Bin for data that is out of normal bin range. |
| `sst.UniqueCountStatistic` | | |
| | *rate* | User defined. if not provided or 0, output at end of simulation. |
| | *startat* | User defined. If set to a time value, the statistic will start disabled, and then be enabled at the set time. If not provided or 0, statistic will start enabled. |
| | *stopat* | User defined. If set to a time value, the statistic will be disabled at the set time. If not provided or 0, then no change in enable will occur. |
| `sst.NullStatistic`        | | |
|                            | none | |


Examples of the interfaces are provided below:

```
compA.enableAllStatistics({"param_1" : "value_1",
                           "param_n" : "value_n" })

compA.enableStatistics(["statistic_name_1",
                        "statistic_name_2",
                        "statistic_name_n"],
                       {"param_1" : "value_1",
                        "param_n" : "value_n" })
sst.enableAllStatisticsForAllComponents({"param_1" : "value_1", 
                                         "param_n" : "value_n" })

sst.enableAllStatisticsForComponentName("Component_Name",
                                       {"param_1" : "value_1", 
                                        "param_n" : "value_n" })

sst.enableStatisticForComponentName("Component_Name",
                                    "Statistic_Name", 
                                   {"param_1" : "value_1", 
                                    "param_n" : "value_n" })

sst.enableAllStatisticsForComponentType("Component_Type",
                                       {"param_1" : "value_1", 
                                        "param_n" : "value_n" })

sst.enableStatisticForComponentType("Component_Type",
                                    "Statistic_Name", 
                                   {"param_1" : "value_1", 
                                    "param_n" : "value_n" })
```

A full example of utilizing the SST statistics interfaces is provided below.

### Example Python Simulation Script with Statistics

```
import sst

# Define SST core options
sst.setProgramOption("timebase", "1 ps")
sst.setProgramOption("stopAtCycle", "200ns")

# Set the Statistic Load Level; Statistics with Enable Levels (set in
# elementInfoStatistic) lower or equal to the load can be enabled (default = 0)
sst.setStatisticLoadLevel(7)

# Set the desired Statistic Output (sst.statOutputConsole is default)
sst.setStatisticOutput("sst.statOutputCSV", {"filepath" : "./TestOutput.csv",
			                                  "separator" : ", "
                                            })

# Define the simulation components
StatExample0 = sst.Component("simpleStatisticsTest0", "coreTestElement.simpleStatisticsComponent")
StatExample0.setRank(0)

# Set Component Parameters
StatExample0.addParams({
      "rng" : """marsaglia""",
      "count" : """100""",   # Change For number of 1ns clocks
      "seed_w" : """1447""",
      "seed_z" : """1053"""
})

# Enable Individual Statistics for the Component with separate rates
StatExample0.enableStatistics([
      "stat1_U32"], {
      "type":"sst.HistogramStatistic",
      "minvalue" : "10",
      "binwidth" : "10",
      "numbins"  : "41",
      "IncludeOutOfBounds" : "1",
      "rate":"5 ns"})

StatExample0.enableStatistics([
      "stat2_U64"], {
      "type":"sst.HistogramStatistic",
      "minvalue" : "1000",
      "binwidth" : "1000",
      "numbins"  : "17",
      "IncludeOutOfBounds" : "1",
      "rate":"10 ns"})

StatExample0.enableStatistics([
      "stat3_I32"], {
      "type":"sst.HistogramStatistic",
      "minvalue" : "-200",
      "binwidth" : "50",
      "numbins"  : "8",
      "IncludeOutOfBounds" : "1",
      "rate":"25 events"})

StatExample0.enableStatistics([
      "stat4_I64"], {
      "type":"sst.HistogramStatistic",
      "minvalue" : "-9000",
      "binwidth" : "1000",
      "numbins"  : "18",
      "IncludeOutOfBounds" : "1",
      "rate":"50 ns"})

StatExample0.enableStatistics([
      "stat5_U32"], {
      "type":"sst.AccumulatorStatistic",
      "rate":"5 ns"
      })

StatExample0.enableStatistics([
      "stat6_U64"], {
      "type":"sst.AccumulatorStatistic",
      "rate":"10 ns",
      "startat":"35ns",
      "stopat":"65ns"
      })
```

*Note that this simulation script can be found in the SST-Core source tree at:
~/tests/test_StatisticsComponent.py*

## Links to Documentation <a name="LinksToDocumentation"></a>
* [SST Release Page](http://sst-simulator.org/SSTPages/SSTMainDownloads/)
* [SST-Core Github](https://github.com/sstsimulator/sst-core)
* [SST-Elements Github](https://github.com/sstsimulator/sst-elements)
* [GraphViz](https://graphviz.org/)
* [SST Python Interfaces](http://sst-simulator.org/SSTPages/SSTUserPythonFileFormat/)
