# bidirectional-ascent-example
## Motivation
Enable computational steering workflows within any [Ascent-instrumented](https://github.com/Alpine-DAV/ascent) simulation via function callbacks and shell commands.

## Setup
### Build Ascent
This work has not been merged into Ascent's develop branch yet, and so it has to be manually built.

#### Clone the Ascent repo
```bash
git clone git@github.com:Alpine-DAV/ascent.git
```

#### Navigate to ascent/scripts/build_ascent/
```bash
cd ascent/scripts/build_ascent/
```

#### Prepare a Python venv (how we get Ascent + Jupyter support)
It doesn't matter where the venv lives, but I will assume that it goes into the `build_ascent` directory. Here are the commands:
```bash
python -m venv env
source env/bin/activate
pip install -U pip setuptools wheel numpy mpi4py
```
While still in the `build_ascent` directory, navigate to:
```bash
cd ascent/src/libs/ascent/python/ascent_jupyter_bridge/
```
and execute:
```bash
pip install -r requirements.txt
pip install .
```
If this step fails, verify that you updated pip, setuptools, and wheel. Finally, navigate back to the `build_ascent` directory:
```bash
cd ../../../../../..
```

#### Create or modify a build script
There are several pre-made build scripts for different HPCs, or you can create your own script if building locally. In any case, the script you use must be modified to enable Python support.

Here's a simple example:
```bash
# Some of the dependencies will fail to compile on gcc/g++ 13
export CC=$(which gcc-12)
export CXX=$(which g++-12)
export FTN=$(which gfortran-12)

# Point this to the venv you created in the previous step
source env/bin/activate

# Add enable_python=ON
env enable_python=ON ./build_ascent.sh
```

Note: I like to add `build_jobs=16` when building locally to speed up the process. If you want CUDA or HIP support, look at the various HPC build scripts to see how to enable those.

#### Run the build script
If everything has been setup correctly, start the build:
```bash
./<my_script>.sh
```
This should take about an 30-60 minutes if everything succeeds, longer if building with CUDA or HIP support.

### Prepare the environment
Building Ascent with `enable_python=ON` will generate the Python modules for both Ascent and Conduit, we will need both. Unfortunately, neither are pip installable, and so we have to use a workaround:

#### Create a `sourceme` file
Technically you could add these lines to your `~/.bashrc`, but I find manual sourcing to be much cleaner. In this example I literally call the file `sourceme`, but you can name it anything:
```bash
# Load any modules here, for example:
module reset
module load cmake
module swap PrgEnv-nvhpc PrgEnv-gnu
module load cudatoolkit-standalone

# Add the necessary modules to PYTHONPATH
export PYTHONPATH=/change-to-your-env-path/env/lib/python3.6/site-packages:/change-to-your-ascent-path/ascent/scripts/build_ascent/install/ascent-develop/python-modules:/change-to-your-ascent-path/ascent/scripts/build_ascent/install/conduit-v0.8.8/python-modules

# Source the venv
source /change-to-your-env-path/env/bin/activate
```
Note: also change the Python venv directory to reflect the version of Python that you used.

#### Source our `sourceme` file
Then to modify our environment tempoarily:
```bash
source sourceme
```

## Examples
Assuming that all of the setup has been done correctly, we can start creating a bidirectional steering workflow within. If our desired simulation has not yet been instrumented with Ascent, do that first.

### Function callbacks
The custom branch of Ascent we built has a new top-level API function that we can invoke called `register_callback`. Callback registration does not require us to create an instance of Ascent first, and so we can perform this step at any time before executing Ascent actions.

The usage is:
```c++
ascent::register_callback("callback_name", function_pointer);
```

The two types of supported callback signatures are:
```c++
void void_callback(conduit::Node &params, conduit::Node &output);
bool bool_callback();
```

Note: we can assume that these callbacks will be executed by MPI, and write them accordingly.

### Void callbacks
Void callbacks are how we create general-purpose bidirectional functionality. Since they take two Conduit nodes, one for input parameters and one for output, we can effectively define a bidirectional API within the simulation code that we can access manually via Jupyter later.

Here's an example of changing the number of timesteps within a simulation:
```c++
void setNumTimesteps(Node &params, Node &output)
{
    // If a new timestep value was passed in, set it
    if (params.has_path("timesteps"))
    {
        // Where time_steps is a global variable
        time_steps = params["timesteps"].as_int64();

        // Re-init the simulation as needed
    }
}
```

Here's another example that outputs diagnostic data:
```c++
void getSimInfo(Node &params, Node &output)
{
    // Where all of these are global variables
    output["timesteps"] = time_steps;
    output["physical/density"] = physical_density;
    output["physical/speed"] = physical_speed;
    output["physical/length"] = physical_length;
    output["physical/viscosity"] = physical_viscosity;
    output["physical/time"] = physical_time;
    output["physical/freq"] = physical_freq;
    output["physical/reynolds_number"] = reynolds_number;
    output["simulation/dx"] = simulation_dx;
    output["simulation/dt"] = simulation_dt;
    output["simulation/speed"] = simulation_speed;
    output["simulation/viscosity"] = simulation_viscosity;
}
```

And finally, an example of using both `params` and `output`:
```c++
void testCallback(conduit::Node &params, conduit::Node &output)
{
    output["param_was_passed"] = false;
    if (params.has_path("example_param"))
    {
        // Do something
        output["param_was_passed"] = true;
    }
}
```

We can effectively make any change to the simulation at runtime with these callbacks, with the main downside being that we have to write the functionality ourselves.

### Bool callbacks
Bool callbacks are mostly useful for defining control flow schemes for Ascent.
```c++
bool computeSomething()
{
    return my_fancy_computation();
}
```
We can use bool callbacks directly with Ascent triggers like so:
```yaml
-
  action: "add_triggers"
  triggers:
    t1:
      params:
        callback: "computeSomething"
        actions_file : "custom_actions.yaml"
```
Then as Ascent executes these actions, `computeSomething` gets invoked and its result is used as the trigger condition.

### Full example

Consider the following `ascent_actions.yaml` file:
```yaml
-
  action: "add_commands"
  commands:
    c1:
      params:
        callback: "setStability"
        mpi_behavior: "all"
-
  action: "add_triggers"
  triggers:
    t1:
      params:
        callback: "isStable"
        actions_file : "stable_actions.yaml"
    t2:
      params:
        callback: "isUnstable"
        actions_file : "unstable_actions.yaml"
```
where these are the callback definitions:
```c++
void setStability(Node &params, Node &output)
{
    bool stable = lbm->checkStability();
    MPI_Allreduce(&stable, &all_stable, 1, MPI_UNSIGNED_CHAR, MPI_MAX, MPI_COMM_WORLD);
    if (!all_stable && rank == 0)
    {
        std::cerr << "Warning: simulation has become unstable (more time steps needed)" << std::endl;
    }
}

bool isStable()
{
    return all_stable;
}

bool isUnstable()
{
    return !all_stable;
}
```
These actions are processed sequentially, and so notice that Ascent will invoke `setStability` before doing anything else. When Ascent invokes a void callback directly, it will always pass null `params` and `output` Conduit nodes. In this case we also tell Ascent to execute the callback on all MPI ranks, which is the default behavior.

Here, `setStability` has each MPI rank execute a function which checks for local stability. If any of the MPI ranks have gone unstable, the `MPI_Allreduce` will communicate the instability to the other ranks. Then the bool callbacks simply return this stability values, which gets used as Ascent's trigger conditions.

Note: this is a little verbose since Ascent doesn't have an `else` clause for triggers right now.

Effectively, this allows Ascent to perform different sets of actions depending on the simulation state. We can imagine the stable state as not needing our attention, while the unstable state notifies us of a problem and gives the user a chance to correct the issue.

### Shell commands
Worth mentioning is that shell commands can also be invoked from an `ascent_actions.yaml` file. They can be defined as one-liners
```yaml
-
  action: "add_commands"
  commands:
    c1:
      params:
        shell_command: echo "Hello, this is a test"
        mpi_behavior: "all"
```
or can be used to call entire shell scripts:
```yaml
-
  action: "add_commands"
  commands:
    c1:
      params:
        shell_command: ./unstable.sh
        mpi_behavior: "root"
```
Whatever your terminal can execute, this can also execute. Shell commands cannot currently be used as Ascent trigger conditions, although we would like to add that functionality at some point. Shell commands are useful for anything that you might want to do externally to the simulation, for example, sending yourself an email if something goes wrong:
```bash
echo "Your simulation has become unstable! Access jupyter to investigate: http://localhost:8888/tree" | mail -s "Unstable Simulation Test" $email
```

### Invoking callbacks manually via Jupyter
First, start the Jupyter server. If you aren't currently sourced into the `sourceme` file that we created earlier, do that now:
```bash
source sourceme
jupyter notebook --ip 0.0.0.0 --port 8888
```
alternatively, Jupyter can be started in the background like so:
```bash
nohup jupyter notebook --ip 0.0.0.0 --port 8888 >/dev/null 2>&1 &
```

Second, we have to tell Ascent to perform a Jupyter extract. The mechanism for actually triggering the extract and yielding control back to the simulation will hopefully become more robust in the future. When Ascent performs a Jupyter extract, the simulation is paused completely to wait for the user to connect. There is no timeout implemented, so the simulation will wait forever if a user never connects.

Suppose we want to trigger a Jupyter extract once every 100 timesteps for exploratory purposes, we can use the following sets of actions:
```yaml
-
  action: "add_triggers"
  triggers:
    t1:
      params:
        condition: "cycle() % 100 == 0"
        actions_file : "jupyter_extract.yaml"
```
and
```yaml
-
  action: "add_extracts"
  extracts:
    e1:
      type: "jupyter"
```
With these 2 files, Ascent will prepare Jupyter for us once every 100 timesteps.

Once Ascent has performed the Jupyter extract and our Jupyter server is running, we should be able to connect via:
```bash
http://localhost:8888/tree
```

From here we can open or create an .ipynb file. Make sure that you have selected the "Ascent Bridge" kernel within Jupyter, otherwise this will not work. To connect our notebook to the simulation:
```python
%connect
```

From here you can access the simulation data, produce new renders, and invoke any simulation callbacks that were registered with Ascent.

Using our prior callback examples, here's how you might manually change the simulation timestep at runtime:
```python
params = conduit.Node()
params["timesteps"] = 1000
output = conduit.Node()
execute_callback("setNumTimesteps", params, output)
```
And if you wanted to get data out of the simulation, here's an example:
```python
params = conduit.Node()
output = conduit.Node()
execute_callback("getSimInfo", params, output),
print(output)
```

Finally, to yield control back to the simulation we invoke:
```python
%disconnect
```

### Conclusion
Since our bidirectional functionality is integrated directly within simulations, we could theoretically do things like:
- Manually insert new geometry or objects into a simulation at runtime.
- Change simulation parameters at runtime to help us find ideal settings.
- Find ideal camera angles or render settings within Jupyter.
- Etc.
