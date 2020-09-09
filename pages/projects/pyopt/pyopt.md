---
title: PyOpt
nav_order: 3
parent: Projects
---

# PyOpt
{:.no_toc}

* TOC
{:toc}

## NOTE

This page was taken from a writeup I did while in the Campbell Muscle Lab at the University of Kentucky. I have not adapted the writeup to be independent of the framework I was using at the time (a heart muscle simulation) so much of this may be confusing. I'm sorry if this causes any inconvenience. If you would like to know more about PyOpt, please reach out and email me by clicking the link at the top right of the page. Thank you!


## Overview

This demo shows the process of fitting muscle force output by a FiberSim simulation to some target data using an optimization technique called particle swarm optimization. Usually this target data would come from experiment, but we're going to simulate some target data to demonstrate the concept and validate that our optimization framework is behaving like we think it should.

This is the fourth demo in the series and if you haven't completed the previous demos, we highly recommend starting at [Demo A](../demo_a/demo_a.md) as this is a gentler introduction to the FiberSim software.

## Background on Particle Swarm Optimization

Particle swarm optimization is an evolutionary algorithm that optimizes a candidate solution (the model we are fitting) to some target data (we're simulating this here, but its usually experimental data) using a population of candidate solutions, a swarm of particles. Each particle is subject to cognitive influence (based on the particle's previous best position), social influence (based on the swarm's best know particle position), and momentum influence (based on the previous velocity of the particle). This can be represented mathematically according to the following equation:

```
v_i1 = c_1 * rand() * (p_best - p_present) + c_2 * rand() * (g_best - p_present) + w * v_i0
```

where `v_i1` is the particle's new velocity, `v_i0` is the particle's previous velocity, `c_1` is the cognitive parameter (between (0, 1)), `c_2` is the social parameter (between (0, 1)), `rand()` is a function that generates a pseudo-random number between [0, 1), `p_best` is the particle's best known position over the optimization history, `p_present` is the particle's current position, `g_best` is the best known particle position across the entire swarm of particles, and `w` is the intertial parameter (between (0, 1)).

To control the behavior of the particle swarm optimization algorithm, we can tune `c_1`, `c_2`, and `w`.

An excellent visualization of a swarm of particles searching a complex parameter-space can be seen below.

![](https://upload.wikimedia.org/wikipedia/commons/e/ec/ParticleSwarmArrowsAnimation.gif)

Attribution: Ephramac [CC BY-SA 4.0 (https://creativecommons.org/licenses/by-sa/4.0)]

## Creating the Directory Structure for Your Simulation

As with the other demos, we're going to create a subdirectory of the `FiberSim_demos` directory called, `demo_d`.


## Dissecting the Python Code

Since this demo is a bit more involved than the previous demos, we're going to dissect the code and go over what each section does. If you would like to skip right to downloading the script, see [this link](simulation_driver.py).

### Importing Necessary Packages and Modules

As with the last demos, we need some standard and third-party modules like `os` and `numpy` as well as some local modules like `util.run` and `util.protocol`. We're going to be using a new third-party module for the particle swarm optimization called `pyswarms` and we're going to be using a new local module and package, `util.instruct` and `optimize`.

`util.instruct` includes routines for reading/manipulating model files. We're going to use a routine located here to modify three rate parameters for simulating some target data that we'll then fit our model to.

`optimize` is a local library that contains classes for optimization and model parameter manipulation. We'll be using the `worker` class for our particles and the `master_worker` class for updating the particles' velocities at each optimization iteration.

The code for this section is as follows:

```python
import sys
import os
import json

import numpy as np

# Store where our FiberSim Python modules are located.
FIBERSIM_MODULES_PATH = "/users/campbell/dylan/github/models/fibersim/python_files"

# Add the location of the FiberSim modules to our file path.
sys.path.append(FIBERSIM_MODULES_PATH)

# Import the utility modules. Note that we're adding the "instruct" module. We're going to be
# using that to modify a model file.
from util import run, protocol, instruct

# Import the new "optimize" module that we'll use to run our optimization job along with the
# pyswarms particle swarm optimization module.
import optimize
import pyswarms as ps
```

### Setting File Names

As with the other demos, we must inform Python where our FiberSim executable and instruction files are. However, with optimization, we must specify some new file paths. We can break these down into files that we'll manually create and files that will be programatically created.

Files we will create:
+ Optimization template. The optimization template is the file that tells the optimizer what parameters it can change in candidate solution model files (in hopes of a better fit). The creation of this is discussed in detail in the ["Optimization Template"](../../instruction_file_creation/instruction_file_creation.md#optimization-template-file) section of the "Instruction File Creation" section of the documentation.

Files that are programatically created:
+ Best model. The best model file is saved whenever a candidate solution reaches a new minimum error in the fit to the target data.
+ Modified model. The modified model is what we use to generate the simulated target data. It is our goal to optimize the parameters that we indicate in the optimization template to be sufficiently close to the values we modify them to in this model file.
+ Simulated target data. This file is where we will write our formatted simulated target data.

```python
# Get the location of this file on your computer.
root = os.path.dirname(__file__)

# Store the locations of the instruction files and the executable like we've done previously.
options_file_path = os.path.join(root, "sim_options.json")
model_file_path = os.path.join(root, "original_model.json") # Note the name change here.
protocol_file_path = os.path.join(root, "protocol.txt")
output_file_path = os.path.join(root, "results")
fibersim_file_path = os.path.join(root, "..", "FiberSim.exe")

# Now we also need some new files.
# We need an optimization template to tell the optimizer what parameters to change to fit the
# target data we're trying to produce.
optimization_template_path = os.path.join(root, "optimization_template.json")
# We need a "best" model file that is overwritten anytime a new minimum error is found.
best_model_file_path = os.path.join(root, "best_model.json")
# We need a modified model file to generate the simulated target data for us.
modified_model_file_path = os.path.join(root, "modified_model.json")
# Finally, we choose a place to store our structured simulated target data.
target_data_dir = os.path.join(root, "simulated_target_data")
simulated_target_data_path = os.path.join(target_data_dir, "simulated_target_data.txt")
```

### Writing the Modified Model for Simulating Target Data

In this section we read in the original model file, use the `instruct` module to modify some cross-bridge transition rate parameters, and finally write the model file to the path we previously designated.

Here we are modifying the first and second parameters of the third rate equation and the first parameter of the fifth rate equation.

```python
# We're going to read in a copy of the original model that we have and modify it so it will produce 
# a new force trace for us to optimize to.
with open(model_file_path, 'r') as f:
  modified_model = json.load(f)

# We're going to use the "instruct" module to modify some cross-bridge transition rate
# parameters.
instruct.set_rate_param("3@1", 300.0, modified_model["kinetics"]["rate_equations"])
instruct.set_rate_param("3@2", 27.0, modified_model["kinetics"]["rate_equations"])
instruct.set_rate_param("5@1", 3.0, modified_model["kinetics"]["rate_equations"])

# Now we write the modified model.
with open(modified_model_file_path, 'w') as f:
  json.dump(modified_model, f)
```

### Generating the Protocol and Target Data

Here we generate an isometric constant pCa protocol like we've done previously using the `protocol` module and run the simulation using the modified model file using the `run` module.

```python
# Store some information for the protocol generation.
time_step = 0.001  # Time step in seconds.
time_steps_for_optimizer_to_skip = 10
no_of_time_steps = 40
pCa = 5.5
Pi_conc = 7e-4  # [M]
ADP_conc = 3e-5  # [M]
ATP_conc = 4e-3  # [M]

# Generate the protocol file for the simulated target data.
protocol.generate_isometric_pCa_protocol(time_step, no_of_time_steps, pCa, 
  protocol_file_path, Pi_conc, ADP_conc, ATP_conc)

# Turn off the option for FiberSim to print out the simulation information continuously. This can
# be cumbersome to print everything to the prompt when running many simulations.
continuous_output = False

# Use the run module to simulate the target data for us.
run.fibersim_simulation(fibersim_file_path, options_file_path, modified_model_file_path, 
  protocol_file_path, target_data_dir, continuous_output=continuous_output)
```

### Restructuring the Simulated Target Data to a Standard Format

FiberSim outputs the target force data in a format that is not supported by the optimizer. To circumvent this, we read in and modify the structure of the target data such that it contains a column for the simulation time and a column for the force produced by the half-sarcomere. We then write this at the file path we described earlier.

```python
# Once the simulated target data is done running, we need to load that in. The most convenient way
# to do that is using numpy's loadtxt function.
target_data_file_path = os.path.join(target_data_dir, "forces.txt")
target_force_data = np.loadtxt(target_data_file_path, skiprows=1)[:, -1]

# We also need the simulation time for the target data. We can get that through the protocol module.
target_sim_times = np.asarray(protocol.get_sim_time(protocol_file_path))

# We have to make sure the target times start at zero, so we subtract off the time we've indicated 
# it takes for the model to reach steady state.
target_sim_times -= target_sim_times[time_steps_for_optimizer_to_skip]

# Now we combine the data together into one array.
target_data = np.stack((target_sim_times, target_force_data), axis=-1)

# And finally, we need to cut off the number of time steps we're wanting to skip. Typically this 
# would be used for allowing the model to reach pseudo-steady state.
target_data = target_data[time_steps_for_optimizer_to_skip:, :]

# We now write the target data so each worker can read it in.
np.savetxt(simulated_target_data_path, target_data)
```

### Setting up the Optimization Structures and Running the Optimization

Now that we have the prerequisite target data generated and structured correctly, we can construct our `master_worker` for driving the optimization and begin the job. We start by forming a dictionary that tells the `master_worker` all of the file paths and options it will need. Then we construct the `master_worker`. After this, we set the cognitive, social, and intertial parameters the particle swarm optimizer will use to determine the particles' velocities. Finally, we create an instance of the particle swarm optimizer and begin the job.

Once we begin the optimization, a display of the optimization's progress will show on screen and will update at each iteration. We'll let this run until it reaches a sufficiently close fit to the target data.

```python
# Now that we have the target data generated, we can make the objects for optimization.
# First, we have to set up the dictionary for the optimization job.
template_dict = {
  "fibersim_file_string": fibersim_file_path,
  "protocol_file_string": protocol_file_path,
  "options_file_string": options_file_path,
  "fit_mode": "time",
  "fit_variable": "muscle_force",
  "original_json_model_file_string": model_file_path,
  "best_model_file_string": best_model_file_path,
  "optimization_json_template_string": optimization_template_path,
  "output_dir_string": output_file_path,
  "target_data_string": simulated_target_data_path,
  "time_steps_to_steady_state": time_steps_for_optimizer_to_skip,
  "compute_rolling_average": False
}
n_workers = 20
no_params_to_optimize = 3
# We're going to make a MasterWorker object that drives all of the "particles" in our particle
# swarm optimization.
master_worker = optimize.master_worker.from_dict(template_dict, n_workers=n_workers)

# Now we have to define the "hyperparameters" that tell the particle swarm optimization routine
# how to behave. There are three parameters that we must define.
pso_options = {
  "c1": 0.5,  # Cognitive parameter, tells particle how fast to move relative to its previous best 
              # position.
  "c2": 0.5,  # Social parameter, tells particle how fast to move relative to best particle 
              # location.
  "w": 0.4    # Intertial parameter, how fast the particle moves relative to previous velocity.
}

# Start an instance of the particle swarm optimizer.
pso_optimizer = ps.single.GlobalBestPSO(n_particles=n_workers, dimensions=no_params_to_optimize,
  options=pso_options)

# Perform the optimization.
max_iters = 100
pso_optimizer.optimize(master_worker.drive_workers, iters=max_iters)
```

## Running the Simulation Driver

Now we can run the simulation driver in the same manner as we have done previously. The optimization should proceed in a similar fashion to the movie below. Note that it will not proceed in the same fashion though, as the particle swarm optimization routine incorporates the use of pseudo-random numbers.

![](optimization_plot.gif)




