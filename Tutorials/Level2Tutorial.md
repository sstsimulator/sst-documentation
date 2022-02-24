# Level 2 Tutorial

## Table of Contents
1. [Parallel SST Runs](#ParallelSSTRuns)
2. [Advanced Python Scripting](#AdvPythonScripting)
3. [Advanced JSON Scripting](#AdvJSONScripting)
4. [Loading Parallel Graphs](#LoadingParallelGraphs)
5. [Links to External SST Components](#ExternSSTComp)

## Parallel SST runs: <a name="ParallelSSTRuns"></a>

### Introduction and Environment

One of the key features of SST is the ability to execute parallel 
simulations that utilize the MPI (Message Passing Interface) to span 
multiple cores and/or multiple nodes in a cluster.  The SST Core interfaces 
are natively built to utilize MPI in order to perform communication between 
individual components across multiple MPI "ranks."  This enables users to 
build and execute large scale simulations without fear of exhausting a single 
system's resources.  When a user requests a parallel simulation (using MPI 
or threading), the SST Core infrastructure will automatically partition the 
requested components and subcomponents across the target number of parallel 
units.  This partitioning considers the requested simulation components, the 
connectivity of the components and the latency of the links between components 
in order to co-locate components and subcomponents that are expected to frequently 
exchange events.

The MPI parallelism present in SST can only be utilized if the SST Core 
infrastructure was built with MPI enabled.  Generally speaking, this is enabled 
by default (MPI must be explicitly disabled when configuring the SST Core 
build using the `--disable-mpi` configuration option).  Further, while there 
are a number of available open source and proprietary MPI packages available, 
the OpenMPI implementation is preferred for the SST Core environment.  For the 
remainder of this section, ensure that the OpenMPI `mpirun` utility is in 
your standard `PATH` environment.

One important item to note when building and utilizing SST simulations is that 
the parallel runtime functionality (MPI parallelism and threading) are explicitly 
hidden from the SST components.  Components *do not* have access to the MPI 
runtime environment directly.  This allows developers to build composable 
SST components that have the ability to execute in a sequential environment 
and a parallel environment without the need to rebuild the component from source.

### Executing Parallel Simulations

Executing a parallel simulation with MPI is a simple process that does not 
require any modifications to your simulation input file or the SST Core infrastructure.  
Assuming that the SST Core was built with MPI parallelism enabled (default), 
executing a parallel MPI simulation is analogous to executing any other 
parallel MPI application.  The MPI runtime environment provides an execution 
script, `mpirun`, that provides the necessary facilities to setup and tear down 
a parallel job that executes across multiple nodes of a cluster.  Different MPI 
implementations may utilize different `mpirun` launch command options.  For 
the purpose of this tutorial, we will rely upon the standard OpenMPI implementation 
of `mpirun`.

The OpenMPI implementation of `mpirun` requires two fundamental arguments in order 
to configure the parallel environment.  First, `mpirun` requires that the user 
specify the number of MPI processes to start.  This is analogous to the number 
of MPI "ranks."  The `-n` option accepts an integer that specifies the number 
of ranks to execute.  For each rank, a unique process will be spawned by the 
MPI runtime for parallel execution.  Second, `mpirun` requires the user 
to provide a list of permissible hosts to execute each MPI rank.  This hostfile 
contains a set of entries, one per line, where each entry is the hostname of 
a target compute node.  The number of entries in the hostfile must match 
the number of requested ranks.  For example, if you seek to execute two 
ranks, then there must be at least two entries in the hostfile.  If you seek to 
execute multiple MPI ranks on a single compute node, then the host name of the 
target compute node can be duplicated.

One important item to note is that many high performance computing clusters are 
configured with a "batch scheduling" tool.  This tool provides time and resource 
sharing interfaces such that users can safely share the available computational 
resources without fear of colliding with one another.  We have introduced the basic 
MPI runtime features using hostfiles in this section.  We will briefly describe 
additional facilities below for which to integrate the SST MPI parallel runtime 
environment with the [Slurm](https://slurm.schedmd.com/documentation.html) 
resource manager.  Please consult your system administrators 
for more information regarding how to utilize the resource management facilities in 
your respective environment.

An example of executing MPI parallel simulations is listed as follows:

```
# Execute a simulation with two ranks
$> mpirun --hostfile file.txt -n 2 sst /path/to/sim.py

# Execute a simulation with ten ranks
$> mpirun --hostfile file.txt -n 10 sst /path/to/sim.py
```

The following are examples of MPI hostfiles utilized for running 
different configurations of a five rank parallel simulation:

```
# Sample host file: 5 ranks across 5 nodes
node001
node003
node002
node004
node005
```

```
# Sample host file: 5 ranks across 1 node
node001
node001
node001
node001
node001
```

### Executing Parallel Simulations with Threading

While MPI parallelism is the preferred method of executing parallel simulations 
with SST, the SST Core infrastructure also provides thread-level parallelism.  Users 
can execute SST locally using thread parallelism and/or utilize thread-level 
parallelism with MPI parallelism.  Much in the same manner as MPI parallelism, 
executing using thread parallelism (and no MPI parallelism) enables SST to 
automatically partition the requested simulation components across multiple 
threads within a single compute node.  By default, SST utilizes a single 
thread of execution.  When using MPI, the default behavior is one thread 
per MPI rank.  SST provides a runtime command line argument to request 
multiple threads.  The `-n` option requires an integer argument that specifies 
the number of requested threads or threads per rank.

An example of requesting two threads of execution without MPI is as follows:

```
$> sst -n 2 /path/to/sim.py
```

An example of request two threads for each of four MPI ranks is shown below.
Note that the total parallelism is this case is (4 ranks x 2 threads), or 
eight parallel units.

```
$> mpirun --hostfile file.txt -n 4 sst -n 2 /path/to/sim.py
```

### Integrating with the Slurm Scheduler

As mentioned above, many high performance computing environments are configured 
to utilize a resource management tool in order to provide fair and safe resource 
sharing of compute and network resources.  As an example, many compute clusters 
are configured to utilize the Slurm resource manager.  Generally speaking, 
Slurm requires that the user encapsulate a parallel application in a "job script" 
in order to sufficiently reserve and utilize compute resources.  These job 
scripts are a convenient way to build the execution environment, reserve some number 
of compute nodes and execute the application in parallel.

From our example above, the Slurm job script encapsulates two things.  First, 
the script contains the necessary facilities by which to generate an appropriate 
hostfile for the required number of MPI ranks.  The `srun` command will 
automatically create a text file formatted in the correct way for the 
complementary `mpirun` command.  Second, the job script contains the actual 
`mpirun` command used for launching the target parallel application.  Please 
consult the Slurm documentation or the documentation for the resource 
manager in your target environment for more information on generating 
and utilizing job scripts.

The following is an example of a Slurm batch script
for executing SST across four MPI ranks:

```
#!/bin/bash
#
# Usage: sbatch -N4 slurm.sh
#

# Create the hostfile
srun hostname > output.txt

# Execute SST
mpirun --hostfile output.txt -n 4 sst /path/to/sim.py
```

## Advanced Python Scripting: <a name="AdvPythonScripting"></a>

### Introduction

The Python SST model interfaces provide users a wide array of options 
for constructing scalable simulations using composable interfaces.  The benefit 
to using Python as the basis for the simulation input is that users may utilize 
all the standard Python loop constructs, flow control and function interfaces. The 
standard Python syntax combined with the SST Python classes enable powerful 
features for creating simulations.  The remainder of this section provides an overview 
into a number of advanced Python features as well as the SST Python interfaces.  THis 
section assumes that the user has basic knowledge of Python syntax and data 
structures.

### Program Options

The SST Python interfaces provide users the ability to to retrieve and set global 
program options.  These options allow users to control global options that 
are analogous to those passed on the command line.  This is often useful when 
buidling large, complex simulations that need to be utilized across multiple users 
and systems while maintaining the same runtime environment.  It also reduces the 
overhead to maintain complex shell scripts and command line options.

The SST function `sst.getProgramOptions()` can be utilized to retrieve a full 
dictionary of global program options set from the command line.  The options 
presented can then be utilized to adjudicate how to futher initialize 
simulation options. The following example queries the `verbose` option 
and prints all the global program options if verbosity is set.

```
# Query Global Program Options
import os
import sst

Opts = sst.getProgramOptions()

if Opts["verbose"] == 1 :
  # print all the program options
  print(Opts)
```

Similarly, we can enable certain global program options using the 
`sst.setProgramOption` interface.  This interface accepts both single name:value 
pairs as well as full dictionaries of name:value pairs.  In this example, 
if the verbose flag is set from the command line, we also ask SST to output 
the simulation configuration to a file.  This will ask the SST Core to output the 
simulation graph, as it interpreted the configuration, back to the file `config.py`.

```
# Query & Set Global Program Options
import os
import sst

Opts = sst.getProgramOptions()

if Opts["verbose"] == 1 :
  # output the configuration to a file
  sst.setProgramOption("output-config", "config.py")
```


### Python Model Options

The SST Python interfaces also provide the ability to retrieve 
command line arguments from the user.  This is often helpful when creating 
simulation input scripts that have the ability to generate multiple configuration 
graphs depending upon what the user inputs from the command line.  The command 
line options are passed to the Python interfaces using the SST `--model-options` 
argument to the `sst` command.

A simple example of a simulation where this may be useful is creating a single 
script that drives a multi-CPU simulation, where the number of CPUs created 
in the simulation graph is runtime dependent.  Note that the `sys.argv` 
Python function works similar to the `ARGC`/`ARGV` construct in traditional 
C/CXX applications.  The first entry in the array (0) is the executable name.  
In this case, `sstsim.x` is the actual executable.  Each entry into the 
`model-options` command line argument is then inserted into each array entry, where 
entries are separated by spaces in the actual `mode-options` command line syntax.

```
# Query the command line options and create the target number of components
import sst
import sys

# argv[0] = 'sstsim.x'
# argv[1] = '--cpus'
# argv[2] = 'INTEGER'
print(sys.argv)

CPUS = int(sys.argv[2])

for i in range(0, CPUS):
  print("Building CPU Component " + str(i))
```

Executing this script would resemble the following:
```
$> sst --model-options="--cpus 10" sim.py
```

This would yield the following output:
```
['sstsim.x', '--cpus', '10']
Building CPU Component 0
Building CPU Component 1
Building CPU Component 2
Building CPU Component 3
Building CPU Component 4
Building CPU Component 5
Building CPU Component 6
Building CPU Component 7
Building CPU Component 8
Building CPU Component 9
```

### Building Complex Simulations

Now that we have the ability to query the environment, set 
program set and retrieve command line arguments via the SST Python 
environment, we can start building more complex simulation scripts 
using all of the aforementioned features coupled to the SST Python 
function interfaces.

Before we begin our next example, we need to 
introduct several new SST Python functions, `pushNamePrefix` and 
`popNamePrefix`.  As mentioned in the Level 1 tutorial material, 
each component must retain a unique name internal to SST.  This ensures 
that subcomponents and links are attached to the correct components in the 
prescribed manner.  When constructing simulation scripts that utilize 
a large number of similar components, its often difficult to manually 
create and track the unique names when creating components, links and 
subcomponents.  the `pushNamePrefix` function accepts a string argument.  
This string is inserted as a prefix string to any subsequently created 
component.  In this manner, when the user pushes a string of "FOO" and 
creates a component with a name of "BAR", the fully qualified component 
name becomes "FOO.BAR".  Conversely, the `popNamePrefix` drops the name 
previously pushed for the prefix string.

Next, once you have created the components, you can retrieve their fully 
qualified name as a string using the `component.getFullName()` function.  
For the example above, this will return "FOO.BAR".  Finally, if we find 
errors during the configuration phases of our simulation script, we may 
utilize the `sst.exit()` function.  This function forces the script to 
exit during interpretation before actually attempting to execute the 
script within SST.

For our example, we will create a variable number of 
`coreTestElement.coreTestClockerComponent` components and execute 
each one at 1Mhz.  We seek to create a script that queries the local 
SST program options environment and constructs a variable number of 
clock domains based upon the user's input.  For this, we will utilize 
the SST `--model-options` command line structure to handle a command 
line option from the user (`--clocks INTEGER`) in order to specify the 
target number of clock domains.  Further, if the user selects the 
`-v` verbose runtime option, then the sript will enable the configuraton 
output and echo the Python configuration back to the `config.py` file.  
Finally, for each requested clock domain, we will create a 
`coreTestClockerComponent` with a name prefix of `DOMAINi`, where `i` 
is 0 to CLOCKS-1.  Our script needs to be robust for external users.  If 
the user fails to specify the `--clocks` model options input, then the 
script will display and error and exit gracefully.  Our full example 
is as follows:

```
# Complex Simulation Example
import os
import sst
import sys

# argv[0] = 'sstsim.x'
# argv[1] = '--clocks'
# argv[2] = 'INTEGER'
if len(sys.argv) < 3:
  print("ERROR : Simulation model requires --model-options=\"--clocks INT\"")
  print(sys.argv)
  sst.exit()

# retrieve the number of clock components to create
CLOCKS = int(sys.argv[2])

# retrieve the global program options
Opts = sst.getProgramOptions()

if Opts["verbose"] == 1 :
  # output the configuration to a file
  sst.setProgramOption("output-config", "config.py")
  # print the program options
  print(sst.getProgramOptions())

# create all the clock components
for i in range(0, CLOCKS):
  sst.pushNamePrefix("DOMAIN"+str(i))
  clocker = sst.Component("clocker", "coreTestElement.coreTestClockerComponent")
  clocker.addParams({
      "clockcount" : "100000000",
      "clock" : "1MHz"
  })
  print("Created clock component " + str(i) + ": " + clocker.getFullName())
  sst.popNamePrefix()

# EOF
```

If we execute this script with `$> sst sim.py`, we see that our 
script correctly displays an error:
```
ERROR : Simulation model requires --model-options="--clocks INT"
['sstsim.x']
```

Executing the same script with the correct set of arguments 
will display output similar to the following:
```
$> sst -v --model-options="--clocks 1" sim.py
Created clock component 0: DOMAIN0.clocker
WARNING: Building component "DOMAIN0.clocker" with no links assigned.
Clock is configured for: 1MHz
REGISTER CLOCK #2 at 5 ns
REGISTER CLOCK #3 at 15 ns
  CLOCK #2 - TICK Num 1; Param = 222
  CLOCK #2 - TICK Num 2; Param = 222
  CLOCK #3 - TICK Num 1; Param = 333
  CLOCK #2 - TICK Num 3; Param = 222
  CLOCK #2 - TICK Num 4; Param = 222
  CLOCK #2 - TICK Num 5; Param = 222
  CLOCK #3 - TICK Num 2; Param = 333
  CLOCK #2 - TICK Num 6; Param = 222
  CLOCK #2 - TICK Num 7; Param = 222
  CLOCK #2 - TICK Num 8; Param = 222
  CLOCK #3 - TICK Num 3; Param = 333
  CLOCK #2 - TICK Num 9; Param = 222
  CLOCK #2 - TICK Num 10; Param = 222
  CLOCK #2 - TICK Num 11; Param = 222
  CLOCK #3 - TICK Num 4; Param = 333
  CLOCK #2 - TICK Num 12; Param = 222
  CLOCK #2 - TICK Num 13; Param = 222
  CLOCK #2 - TICK Num 14; Param = 222
  CLOCK #3 - TICK Num 5; Param = 333
  CLOCK #2 - TICK Num 15; Param = 222
  CLOCK #3 - TICK Num 6; Param = 333
  CLOCK #3 - TICK Num 7; Param = 333
  CLOCK #3 - TICK Num 8; Param = 333
  CLOCK #3 - TICK Num 9; Param = 333
  CLOCK #3 - TICK Num 10; Param = 333
  CLOCK #3 - TICK Num 11; Param = 333
  CLOCK #3 - TICK Num 12; Param = 333
  CLOCK #3 - TICK Num 13; Param = 333
  CLOCK #3 - TICK Num 14; Param = 333
  CLOCK #3 - TICK Num 15; Param = 333
Simulation is complete, simulated time: 100 s
```

If we increase the number of generated clocks on the command line, 
we see the following.  Further, if we enable verbosity, we also 
find that SST echoes a valid Python script back out to config.py

```
$> sst -v --model-options="--clocks 2" test2.py
SSTPythonModel: Creating config graph for SST using Python model...
{'debug-file': '/dev/null', 'stop-at': '0 ns', 'heartbeat-period': 'N', 'timebase': '1 ps', 'partitioner': 'sst.linear', 'verbose': 1, 'output-partition': '', 'output-config': 'config.py', 'output-dot': '', 'numRanks': 1, 'numThreads': 1, 'run-mode': 'both'}
Created clock component 0: DOMAIN0.clocker
Created clock component 1: DOMAIN1.clocker
WARNING: Building component "DOMAIN0.clocker" with no links assigned.
Clock is configured for: 1MHz
REGISTER CLOCK #2 at 5 ns
REGISTER CLOCK #3 at 15 ns
WARNING: Building component "DOMAIN1.clocker" with no links assigned.
Clock is configured for: 1MHz
REGISTER CLOCK #2 at 5 ns
REGISTER CLOCK #3 at 15 ns
 SST Core: # Starting main event loop
 SST Core: # Start time: 2022/02/24 at: 15:04:51
  CLOCK #2 - TICK Num 1; Param = 222
  CLOCK #2 - TICK Num 1; Param = 222
  CLOCK #2 - TICK Num 2; Param = 222
  CLOCK #2 - TICK Num 2; Param = 222
  CLOCK #3 - TICK Num 1; Param = 333
  CLOCK #3 - TICK Num 1; Param = 333
  CLOCK #2 - TICK Num 3; Param = 222
  CLOCK #2 - TICK Num 3; Param = 222
  CLOCK #2 - TICK Num 4; Param = 222
  CLOCK #2 - TICK Num 4; Param = 222
  CLOCK #2 - TICK Num 5; Param = 222
  CLOCK #2 - TICK Num 5; Param = 222
  CLOCK #3 - TICK Num 2; Param = 333
  CLOCK #3 - TICK Num 2; Param = 333
  CLOCK #2 - TICK Num 6; Param = 222
  CLOCK #2 - TICK Num 6; Param = 222
  CLOCK #2 - TICK Num 7; Param = 222
  CLOCK #2 - TICK Num 7; Param = 222
  CLOCK #2 - TICK Num 8; Param = 222
  CLOCK #2 - TICK Num 8; Param = 222
  CLOCK #3 - TICK Num 3; Param = 333
  CLOCK #3 - TICK Num 3; Param = 333
  CLOCK #2 - TICK Num 9; Param = 222
  CLOCK #2 - TICK Num 9; Param = 222
  CLOCK #2 - TICK Num 10; Param = 222
  CLOCK #2 - TICK Num 10; Param = 222
  CLOCK #2 - TICK Num 11; Param = 222
  CLOCK #2 - TICK Num 11; Param = 222
  CLOCK #3 - TICK Num 4; Param = 333
  CLOCK #3 - TICK Num 4; Param = 333
  CLOCK #2 - TICK Num 12; Param = 222
  CLOCK #2 - TICK Num 12; Param = 222
  CLOCK #2 - TICK Num 13; Param = 222
  CLOCK #2 - TICK Num 13; Param = 222
  CLOCK #2 - TICK Num 14; Param = 222
  CLOCK #2 - TICK Num 14; Param = 222
  CLOCK #3 - TICK Num 5; Param = 333
  CLOCK #3 - TICK Num 5; Param = 333
  CLOCK #2 - TICK Num 15; Param = 222
  CLOCK #2 - TICK Num 15; Param = 222
  CLOCK #3 - TICK Num 6; Param = 333
  CLOCK #3 - TICK Num 6; Param = 333
  CLOCK #3 - TICK Num 7; Param = 333
  CLOCK #3 - TICK Num 7; Param = 333
  CLOCK #3 - TICK Num 8; Param = 333
  CLOCK #3 - TICK Num 8; Param = 333
  CLOCK #3 - TICK Num 9; Param = 333
  CLOCK #3 - TICK Num 9; Param = 333
  CLOCK #3 - TICK Num 10; Param = 333
  CLOCK #3 - TICK Num 10; Param = 333
  CLOCK #3 - TICK Num 11; Param = 333
  CLOCK #3 - TICK Num 11; Param = 333
  CLOCK #3 - TICK Num 12; Param = 333
  CLOCK #3 - TICK Num 12; Param = 333
  CLOCK #3 - TICK Num 13; Param = 333
  CLOCK #3 - TICK Num 13; Param = 333
  CLOCK #3 - TICK Num 14; Param = 333
  CLOCK #3 - TICK Num 14; Param = 333
  CLOCK #3 - TICK Num 15; Param = 333
  CLOCK #3 - TICK Num 15; Param = 333


------------------------------------------------------------
Simulation Timing Information:
Build time:                      0.041296 seconds
Simulation time:                 5.919092 seconds
Total time:                      5.960415 seconds
Simulated time:                  100 s

Simulation Resource Information:
Max Resident Set Size:           27.452 MB
Approx. Global Max RSS Size:     27.452 MB
Max Local Page Faults:           0 faults
Global Page Faults:              0 faults
Max Output Blocks:               0 blocks
Max Input Blocks:                8 blocks
Max mempool usage:               8.98022 MB
Global mempool usage:            8.98022 MB
Global active activities:        2 activities
Current global TimeVortex depth: 4 entries
Max TimeVortex depth:            5 entries
Max Sync data size:              0 B
Global Sync data size:           0 B
------------------------------------------------------------
```

Generated config.py script:
```
# Automatically generated by SST
import sst

# Define SST Program Options:
sst.setProgramOption("timebase", "1 ps")
sst.setProgramOption("stopAtCycle", "0 ns")

# Define the SST Components:
comp_DOMAIN0_clocker = sst.Component("DOMAIN0.clocker", "coreTestElement.coreTestClockerComponent")
comp_DOMAIN0_clocker.addParams({
     "clock" : "1MHz",
     "clockcount" : "100000000"
})
comp_DOMAIN0_clocker.setCoordinates(0, 0, 0)
comp_DOMAIN1_clocker = sst.Component("DOMAIN1.clocker", "coreTestElement.coreTestClockerComponent")
comp_DOMAIN1_clocker.addParams({
     "clock" : "1MHz",
     "clockcount" : "100000000"
})
comp_DOMAIN1_clocker.setCoordinates(0, 0, 0)
# Define SST Statistics Options:
sst.setStatisticOutput("sst.statOutputConsole")
# End of generated output.
```

### Python Classes

The following section projects an outline of the available SST Python 
function interfaces.  We have utilized many of the interfaces throughout the 
Level 1 and Level 2 tutorials.  Please refer back to the various samples 
for examples on utilizing the interfaces.

#### Component/Subcomponent Class

* *Function* : `sst.Component(name,type)`
  * *Description* : A Component is created by instancing an `sst.Component` class
  * *Parameters*
    * `name`: name of the Component. The name can be used to get a 
    handle to the Component later in the python code. The name is 
    also available to the c++ implementation of the Component
    * `type`: type of the Component in the lib.element format
    (for example, coreTestElement.coreTestClockerComponent)
* *Function* : `setSubComponent(slot_name, type, slot_index = 0)`
  * *Description* : Inserts a subcomponent of the specified type into the indicated slot name and index.
  * *Parameters* :
    * `slot_name` : name of the slot in which the SubComponent should be loaded.
    * `type` : type of the SubComponent in the lib.element format 
    (for example, merlin.linkcontrol)
    * `slot_index` : slot index in which the SubComponent should be inserted. 
    This defaults to 0 and is not required if only one SubComponent is being 
    loaded into the specified slot. Each SubComponent must be loaded into a 
    unique slot_index and some (Sub)Components will require the indices to be 
    linear with no gaps.
* *Function* : `addParam(key, value)`
  * *Description* : Adds a parameter to the Params object for the Component/SubComponent
  * *Parameters* :
    * `key` : name of the parameter
    * `value` : value of the parameter. This can be almost any python object 
    and the __str__ method will be called to get a string representation. 
    A list can be passed to this call when the find_array function is used in 
    the C++ class to retrieve the parameters.
* *Function* : `addParams(params)`
  * *Description* : Adds multiple parameters to the Params object for the Component/SubComponent
  * *Parameters* :
    * `params` : a python dict of key, value pairs. See addParam description 
    for information about how key and value are used.
* *Function* : `addLink(link, port, latency=link_default)`
  * *Description* : Connects a link to the specified port with the 
  specified latency on the link. If a default latency is specified on the link, 
  then the latency parameter is optional
  * *Parameters* :
    * `link` : sst.Link object that will be connected to the port
    * `port` : name of the port to connect the link to
    * `latency` : latency of the link from the perspective of this 
    Component/SubComponent sending an event. This parameter is optional and 
    the call will use the default latency set on the link if it's not specified 
    in the call. If provided, must include units (e.g., "ns").
* *Function* : `getFullName()`
  * *Description* : Returns the full name of the Component/SubComponent. For 
  Components, the name is the one supplied by the user at the time the 
  Component was created. For SubComponents, the name is automatically generated 
  from the parent Component and slot name. At each level, the name is generated 
  as the parent's name plus the slot name, separated by a colon.  Names are in the form: 
  `Parent:slot[index]` or `Parent:slot[index]:slot[index]`
  * *Parameters* : none
* *Function* : `getType()`
  * *Description* : Returns the type of the component in lib.element format. 
  This is simply the type supplied to either the Component constructor or setSubComponent() call.
  * *Parameters* : none
* *Function* : `setStatisticLoadLevel(level, apply_to_children=False)`
  * *Description* : Sets the load level for this statistic. This overrides 
  the default load level. Load levels are not used for statistics that are explicitly enabled
  * *Parameters* : Enables all statistics for the Component/SubComponent on which the call is made.
    * `level` : statistic load level for the component
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistics on all currently instanced SubComponent descendants. 
    SubComponents created after this call will not have their load level set
* *Function* : `enableAllStatistics(stat_params_dict, apply_to_children=False)`
  * *Description* : Enables all statistics for the Component/SubComponent on which the call is made. 
  * *Parameters* :
    * `stat_params_dict` : Python dictionary that specified the statistic 
    parameters. All statistics will get the same set of parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistics on all currently instanced SubComponent descendants. SubComponents 
    created after this call will not have their statistics enabled.
* *Function* : `enableStatistics(stat_list, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables a list of statistics for the component on which the call is made.
  * *Parameters* :
    * `stat_list` : list of statistics to be enabled. If only one stat is to 
    be enabled, you may pass a single string instead of a list.
    * `stat_params_dict` : Python dictionary that specified the statistic 
    parameters. All statistics will get the same set of parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistics on all currently instanced SubComponent descendants. 
    SubComponents created after this call will not have their statistics enabled

The following functions only apply to `component` objects:
* *Function* : `setRank(mpi_rank, thread = 0)`
  * *Description* : Sets the rank the Component should be assigned to.
  * *Parameters* :
    * `mpi_rank` : MPI rank the Component should be assigned to
    * `thread` : thread the Component should be assigned to
* *Function* : `setWeight(weight)`
  * *Description* : Sets the weight of the Component. The weight is used by 
  some partitioners to help balance Components across ranks.
  * *Parameters* :
    * `weight` : weight of the Component specified as a float. 
    Weights can have any value, but should be meaningful in the 
    context of the overall simulation.

#### Link Class
* *Function* : `Link(name, latency=None)`
  * *Description* : The Link object is used to connect Component/SubComponents 
  together to form the simulation
  * *Parameters* :
    * `name` : Name of the link
    * `latency` : default latency for the link, optional. Must include units 
    (e.g., "ns"). This will be used if no latency is specified in calls to 
    Link.connect() or (Sub)Component.addLink()
* *Function* : `connect( (comp1, port1 latency1=default), (comp2, port2, latency2=default)`
  * *Description* : Connects two ports using the link object. 
  If a default latency is specified on the Link, then the latency parameters 
  are optional, however, if no default latency is set on the Link, the latency 
  parameters to connect are required.  The actual parameters are two tuples 
  representing the information for the ports to be connected. The fields in the 
  tuple are (comp, port, latency) as describe below.
  * *Parameters* :
    * `comp` : Component/SubComponent object that the port is a part of
    * `port` : Port to connect to
    * `latency` : latency of link from the perspective of the corresponding 
    Component/SubComponent sending an event. This is optional, and if not 
    specified, the default latency of the link will be used. If no latency is 
    set, either in the call or as a default, the call will fatal.
* *Function* : `setNoCut()`
  * *Description* : Tell the simulator that this link should not be "cut" by a 
  partition boundary. In effect, it will guarantee that the two Components 
  connected by this link will be on the same rank. This must be used with care, 
  as you can easily get into a situation where too many components are on the same rank.
  * *Parameters* : none

#### Global Functions
* *Function* : `setProgramOption(option,value)`
  * *Description* : Sets the specified program option for the simulation. These 
  mirror the options available on the sst command line. Parameters set in the 
  python file will be overwritten by options set on the command line. 
  Use `sst –help` to get a list of available options.
  * *Parameters* :
    * `option` : configuration option to be set
    * `value` : value to set for the target option
* *Function* : `setProgramOptions(options)`
  * *Description* : Sets the specified program options for the simulation. These 
  mirror the options available on the sst command line. Parameters set in the 
  python file will be overwritten by options set on the command line. Use 
  `sst –help` to get a list of available options.
  * *Parameters* :
    * `options` : dictionary the contents specify the option as dictionary keys 
    with the options value being specified by the corresponding dictionary value.
* *Function* : `getProgramOptions()`
  * *Description* : Returns a python a dictionary with the current values of 
  the program options. This will include all program options, not just those 
  set in the python file.
  * *Parameters* : none
* *Function* : `getMPIRankCount()`
  * *Description* : Returns the number of physical MPI ranks in the simulation
  * *Parameters* : none
* *Function* : `getSSTThreadCount()`
  * *Description* : Returns the threads per rank specified for the simulation
  * *Parameters* : none
* *Function* : `setSSTThreadCount(threads)`
  * *Description* : Sets the number of threads per rank for the simulation. 
  This value is overwritten if -n is specified on the command line.
  * *Parameters* :
    * `threads` : number of threads per MPI rank to use in the simulation
* *Function* : `pushNamePrefix(prefix)`
  * *Description* : Pushes a name prefix onto the name stack. This prefix will 
  be added on the names of all Components and Links. The names in the stack 
  are separated by a period. Example, if pushNamePrefix("base") and 
  pushNamePrefix("next") were called in that order, the prefixed name would be 
  "base.next". Prefixes can be popped from the stack using popNamePrefix().
  * *Parameters* :
    * `prefix` : prefix to add to the name stack
* *Function* : `popNamePrefix()`
  * *Description* : Pops a prefix from the name stack. 
  See `pushNamePrefix` for how name stacks are used.
  * *Parameters* : none
* *Function* : `exit()`
  * *Description* : Causes the simulation to exit
  * *Parameters* : none

#### Statistics
* *Function* : `enableAllStatisticsForAllComponents(stat_params_dict)`
  * *Description* : Enables all statistics for all Components in the simulation 
  that have already been instanced
  * *Parameters* :
    * `stat_params_dict` : Python dictionary that specified the statistic 
    parameters. All statistics will get the same set of parameters
* *Function* : `enableAllStatisticsForComponentName(name, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables all statistics for the Component named in the call. 
  This call works for both Components and SubComponents
  * *Parameters* :
    * `name` : name of the Component or SubComponent on which to enable all 
    statistics. The name for SubComponents is described above. Slot indexes are 
    optional in cases where only one SubComponent has been added to a slot, but 
    you can also use [0] in all cases, even when the actual name will not 
    display this way. If a component with the provided name is not found, the 
    function will call fatal()
    * `stat_params_dict` : Python dictionary that specifies the statistic 
    parameters. All statistics will get the same set of parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistic on all SubComponent descendants
* *Function* : `enableStatisticForComponentName(name, stat, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables a statistic for the component on which the call is made.
  * *Parameters* :
    * `name` : name of the Component or SubComponent on which to enable all 
    statistics. The name for SubComponents is described above. Slot indexes are 
    optional in cases where only one SubComponent has been added to a slot, but 
    you can also use [0] in all cases, even when the actual name will not display 
    this way. If a component with the provided name is not found, the 
    function will call fatal()
    * `stat` : statistic to be enabled
    * `stat_params_dict` : Python dictionary that specified the statistic parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistic on all SubComponent descendants
* *Function* : `enableStatisticsForComponentName(name, stat, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables a list of statistics for the component on which the call is made
  * *Parameters* :
    * `name` : name of the Component or SubComponent on which to enable the 
    statistics. The name for SubComponents is described above. Slot indexes are 
    optional in cases where only one SubComponent has been added to a slot, but 
    you can also use [0] in all cases, even when the actual name will not display 
    this way. If a component with the provided name is not found, the function will call fatal()
    * `stat_list` : list of statistics to be enabled. If only one stat is to be 
    enabled, you may pass a single string instead of a list
    * `stat_params_dict` : Python dictionary that specified the statistic parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistics on all SubComponent descendants
* *Function* : `enableAllStatisticsForComponentType(type, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables all statistics for all previously instanced 
  Components/SubComponents of the type specified in the call. This call works 
  for both Components and SubComponents
  * *Parameters* :
    * `type` : type of the Component or SubComponent on which to enable all 
    statistics. All previously instanced elements of this type will have their 
    statistics enabled
    * `stat_params_dict` : Python dictionary that specified the statistic 
    parameters. All statistics will get the same set of parameters
    * `apply_to_children` : If set to True, will recursively enable all 
    statistics on all SubComponent descendants.
* *Function* : `enableStatisticForComponentType(type, stat, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables a list of statistics for the component on which the call is made.
  * *Parameters* :
    * `type` : type of the Component or SubComponent on which to enable all 
    statistics. All previously instanced elements of this type will have their 
    statistics enabled
    * `stat` : statistic to be enabled
    * `stat_params_dict` : Python dictionary that specified the statistic parameters
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistic on all SubComponent descendants
* *Function* : `enableStatisticsForComponentType(type, stat_list, stat_params_dict, apply_to_children=False)`
  * *Description* : Enables a list of statistics for the component on which the call is made.
  * *Parameters* :
    * `type` : type of the Component or SubComponent on which to enable all 
    statistics. All previously instanced elements of this type will have their 
    statistics enabled.
    * `stat_list` : list of statistics to be enabled. If only one stat is to be 
    enabled, you may pass a single string instead of a list.
    * `stat_params_dict` : Python dictionary that specified the statistic parameters
    * `apply_to_children` :  If set to True, will recursively enable specified 
    statistic on all SubComponent descendants.
* *Function* : `setStatisticLoadLevelForComponentName(name, level, apply_to_children=False)`
  * *Description* : Sets the statistic load level for the named component.
  * *Parameters* :
    * `name` : name of the Component or SubComponent on which to enable all 
    statistics. The name for SubComponents is described above. Slot indexes are 
    optional in cases where only one SubComponent has been added to a slot, but 
    you can also use [0] in all cases, even when the actual name will not 
    display this way. If a component with the provided name is not found, the 
    function will call fatal()
    * `level` : statistic load level for the component
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistic on all SubComponent descendants.
* *Function* : `setStatisticLoadLevelForComponentType(type, level, apply_to_children=False)`
  * *Description* : Sets the statistic load level for all components of the specified type
  * *Parameters* :
    * `type` : type of the Component or SubComponent on which to enable all 
    statistics. All previously instanced elements of this type will have their 
    level set.
    * `level` : statistic load level for the components
    * `apply_to_children` : If set to True, will recursively enable specified 
    statistic on all SubComponent descendants.
* *Function* : `setStatisticLoadLevel(level)`
  * *Description* : Set the global statistic load level. This level is used if 
  individual load levels are not set. Also, the load level is only used for 
  statistics not specifically enabled (i.e., not enabled using one of the 
  enableAllStatistics variants).
  * *Parameters* :
    * `level` : value to set load level to
* *Function* : `getStatisticLoadLevel()`
  * *Description* : Return the global load level
  * *Parameters* : none
* *Function* : `setStatisticOutput(stat_output_module)`
  * *Description* :
  * *Parameters* :
* *Function* : `setStatisticOutputOption(option,value)`
  * *Description* : Set the specified option for the StatisticOutput object.
  * *Parameters* :
    * `option` : option to set
    * `value` : value to set option to
* *Function* : `setStatisticOutputOptions(options)`
  * *Description* : Set the specified options for the StatisticOutput object
  * *Parameters* :
    * `options` : dictionary of option (key), value pairs

## Advanced JSON Scripting: <a name="AdvJSONScripting"></a>

One of the unique features of the latest versions of SST is the ability 
to specify simulation models using Python or JSON input files.  The JSON 
simulation inputs provide users the ability to rapidly generate complex 
simulations using external tools that generate human-readable JSON files.
Currently, the SST JSON simulation loader is *automatically* invoked 
when a JSON file is specified on the command line.  This is analogous 
to the following:

```
$> sst /path/to/simulation.json
```

The SST JSON input files are organized in a very specific manner 
such that global program options, components, subcomponents and 
links are specified in unique blocks.  These blocks are subsequently 
linked together in order to form a simulation graph.  Currently, 
the SST JSON model loader accepts all of the standard program options, 
component attributes, recursively nested subcomponents (also with attributes) 
and link configuration data.  The SST JSON loader does not currently handle 
global statistics options.

In the following example, we have a sample JSON input file with a global
program options block that include `verbose`, `stop-at` and `timeVortex` 
parameters.  The syntax for the global parameters is identical to that 
of the Python scripting format.  We also have a `components` block 
that includes four components.  For each component, we define 
the `name`, `type` (specifying the component model) and a set of 
`params`.  Notice for `component.1` we also specify a set of 
`subcomponents`. Finally, we specify a `links` block that includes 
the `left` and `right` links, their respective ports, components and 
latency parameters.

```
{
  "program_options":{
    "verbose": "1",
    "stop-at": "25us",
    "timeVortex": "sst.timevortex.priority_queue"
  },
  "components":[
    {
      "name": "component.0",
      "type": "coreTestElement.coreTestComponent",
      "params": {
        "workPerCycle": "1000",
        "commSize": "100",
        "commFreq": "1000"
      }
    },
    {
      "name": "component.1",
      "type": "coreTestElement.coreTestComponent",
      "params": {
        "workPerCycle": "1000",
        "commSize": "100",
        "commFreq": "1000"
      },
      "subcomponents": [
        {
          "slot_name": "generator",
          "slot_number": 0,
          "type": "miranda.Stencil3DBenchGenerator",
          "params": {
            "verbose": "0",
            "nx": "30",
            "ny": "20",
            "nz": "10"
          }
        }
      ]
    },
    {
      "name": "component.2",
      "type": "coreTestElement.coreTestComponent",
      "params": {
        "workPerCycle": "1000",
        "commSize": "100",
        "commFreq": "1000"
      }
    },
    {
      "name": "component.3",
      "type": "coreTestElement.coreTestComponent",
      "params": {
        "workPerCycle": "1000",
        "commSize": "100",
        "commFreq": "1000"
      }
    }
  ],
  "links": [
    {
      "name": "link_s_0_10",
      "left": {
        "component": "component.0",
        "port": "Nlink",
        "latency": "10000ps"
      },
      "right": {
        "component": "component.1",
        "port": "Slink",
        "latency": "10000ps"
      }
    },
    {
      "name": "link_s_0_90",
      "left": {
        "component": "component.2",
        "port": "Slink",
        "latency": "10000ps"
      },
      "right": {
        "component": "component.3",
        "port": "Nlink",
        "latency": "10000ps"
      }
    }
  ]
}
```

In addition to writing files directly in the JSON model format, 
users may also utilize SST to create JSON input files with existing 
Python input files.  This offers a convenient migration path from 
existing simulations written in the original Python scripting 
form to a more human-readable JSON file.
This can be done using the following command:

```
$> sst --run-mode=init --output-json=/path/to/simulation.json simulation.py
```

As an example of using this feature, we have generated a set of 
sample JSON inputs using existing SST test cases.  These are listed in 
the table below:

|  **Description**  | **Python Input** | **JSON Input** |
|:-|:-|:-|
| Test Component | [Test Component Python](samples/test_Component.py) | [Test Component JSON](samples/test_Component.json)|
| Miranda Stencil3D | [Miranda 3D Python](samples/stencil3dbench.py)| [Miranda3D JSON](samples/stencil3dbench.json)|
| Merlin 256 Node FatTree | [Merlin 256 Python](samples/fattree_256_test.py)| [Merlin 256 JSON](samples/fattree_256_test.json)|


## Loading Parallel Graphs: <a name="LoadingParallelGraphs"></a>

With the latest updates to SST, users now have the ability to build and execute 
simulation inputs in parallel.  Historically, SST would assign rank-0 to 
load, connect, validate and distribute the simulation graph data to all 
the subsequent MPI ranks.  While this path works well for reasonably sized 
graphs, very large graph inputs require a significant amount of memory 
during graph construction, thus placing a large burden on rank-0.  When 
using the parallel graph loader, users have the ability to split a simulation 
graph into `N` files for `N` ranks of a simulation whereby each `N-th` rank
independently loads only its portion of the simulation graph.  SST will handle 
the hard portion of wiring up the graph across MPI ranks.  However, the input 
files for each rank *must* contain the same `program_options`.  In this manner, 
each rank is configured to operate under the same global options.

Executing SST with the parallel graph loader requires that the user create 
a single input file for each rank of the parallel simulation.  As a result, 
if the user seeks to utilize `4` ranks, then `4` input files must be present.  
The files are named using a monotonically increasing integer in the following 
form (for `n` ranks):
```
FILE0.json
FILE1.json
...
FILE{n-1}.json
```

When executing using the parallel graph loader, we need additional options 
and file prefixes on the command line.  First, we must specify the 
`--parallel-output` option in order to trigger the SST core to load 
the per-rank simulation files.  Next, we must specify the base file 
prefix as the input to the simulation.  Using our example above (`FILE0.json`) 
would be specified as `FILE.json`.  An example of doing so resembles the following:

```
$> mpirun --hostfile output.txt -n2 sst --parallel-output=1 FILE.json
```

In addition to the aforementioned parallel graph loading, the latest development 
source for SST also supports converting large simulation graphs into their 
parallel form.  This allows users to easily port their existing Python-based 
simulation inputs to parallel JSON inputs.  Users may accomplish this similar to the 
basic JSON convertor outlined above by adding the `--output-json=FILE.json` 
to the parallel run.  Note that the file name specified in the `--output-json` flag 
must contain a `.json` file extension.  SST will automatically split the input 
Python file into the appropriate `N` output JSON files in the same monotonically 
increasing form as outlined above.

```
$> mpirun --hostfile output.txt -n2 sst --parallel-output=1 --output-json=FILE.json FILE.py
```

The following are several candidate examples of parallel JSON input files using 
standard SST tests.

|  **Description**  | **Ranks** | **Rank Files** |
|:-|:-:|:-|
| Miranda Stencil3D | 2 | [Rank0](samples/parallel/stencil3dbench0.json) [Rank1](samples/parallel/stencil3dbench1.json)|
| Merlin 256 Node FatTree | 4 | [Rank0](samples/parallel/fattree_256_test_parallel0.json) [Rank1](samples/parallel/fattree_256_test_parallel1.json) [Rank2](samples/parallel/fattree_256_test_parallel2.json) [Rank3](samples/parallel/fattree_256_test_parallel3.json)|

## Links to External SST Components: <a name="ExternSSTComp"></a>
* [HBM DRAMSIm2](https://github.com/tactcomplabs/HBM/releases/tag/sst-8.0.0-release)
* [DRAMSim2](https://github.com/dramninjasUMD/DRAMSim2/archive/v2.2.2.tar.gz)
* [DRAMSim3](https://github.com/umd-memsys/dramsim3)
* [NVDIMMSim](https://github.com/jimstevens2001/NVDIMMSIM/archive/v2.0.0.tar.gz)
* [RISC-V Rev](https://github.com/tactcomplabs/rev)
