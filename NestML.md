# block:
    <block_type> [<args>]:
      ...
    end

# Block types
* `neuron` *`<name>`* - The top-level block of a neuron model called `<name>`. The content will be translated into a single neuron model that can be instantiated in PyNEST using `nest.Create(<name>)`. All following blocks are contained in this block.
* `parameters` - This block is composed of a list of variable declarations that are supposed to contain all variables which remain constant during the simulation, but can vary among different simulations or instantiations of the same neuron. These variables can be set and read by the user using `nest.SetStatus(<gid>, <variable>, <value>)` and  `nest.GetStatus(<gid>, <variable>)`.
* `state` - This block is composed of a list of variable declarations that are supposed to describe parts of the neuron which may change over time.
* `initial_values` - This block describes the initial values of all stated differential equations. Only variables from this block can be further defined with differential equations. The variables in this block can be recorded using a `multimeter`.
* `internals` - This block is composed of a list of implementation-dependent helper variables that supposed to be constant during the simulation run. Therefore, their initialization expression can only reference parameters or other internal variables.
* `equations` - This block contains shape definitions and differential equations. It will be explained in further detail [later on in the manual](#equations).
* `input` - This block is composed of one or more input ports. It will be explained in further detail [later on in the manual](#input).
* `output` *`<event_type>`* - Defines which type of event the neuron can send. Currently, only `spike` is supported. No `end` is necessary at the end of this block.
* `update` - Inside this block arbitrary code can be implemented using the internal programming language. The `update` block defines the runtime behavior of the neuron. It contains the logic for state and equation [updates](#equations) and [refractoriness](#concepts-for-refractoriness). This block is translated into the `update` method in NEST.

 __The following blocks are mandataroy: input, output and update__




# Example:
    neuron iaf_neuron:
        state:
            y0, y1, y2, y3, V_m mV [V_m >= -99.0]
            # Membrane potential
            alias V_rel mV = V_m + E_L
        end
        function set_V_rel(v mV):
            y3 = v - E_L
        end
        parameter:
            # Capacity of the membrane.
            C_m pF = 250 [C_m > 0]
        end
        internal:
            h ms = resolution()
            P11 real = exp(-h / tau_syn)
            ...
            P32 real = 1 / C_m * (P33 - P11)
            / (-1/tau_m - -1/tau_syn)
        end
        input:
            spikeBuffer <- inhibitory
            excitatory spike
            currentBuffer <- current
        end
        output: spike
        dynamics timestep(t ms):
            if r == 0: # not refractory
            V_m = P30 * (y0 + I_e) + P31 *
            y1 + P32 * y2 + P33 * V_m
            else:
            r = r - 1
            end
            # alpha shape PSCs
            V_m = P21 * y1 + P22 * y2
            y1 = y1 * P11
            y0 = currentBuffer.getSum(t);
        end
    end

##    state:
Contains the variables of the dynamic state of the neuron. An example for a state
variable is the membrane potential of a neuron (V_m). An alias variable describes
the dependency between variables using an expression (V_rel). For setting a value
on an alias a setter function is required (set_V_rel), as the defining expression
cannot be inverted automatically for the general case. Plausibility constraints can
be added in square brackets after the variable definition (V_m >= -99.0). These are
useful for debugging and during the development phase of the model and can be
removed in the production version for better performance

##   function set_V_rel(v mV):
Functions allow the convenient reuse of code. Their definition starts
with the keyword function followed by the function name and a list of zero or more
function parameters in parentheses. Just like declaring a variable, a parameter is declared
by first stating its name and then its type. Multiple parameters are separated by a comma.
The parameter list is followed by an optional return type

##   parameter:
Contains attributes that do not change over time, but may vary among neuron
instances. Examples are the length of the refractory period or the membrane
capacitance (C_m). To ensure that values are in a sensible range, it is possible to define
guards which are evaluated every time a parameter is changed by the user. The
syntax is the same as for the plausibility constraints in the state block.

##    internal:
Contains values that depend on the parameters, but can be precalculated once
or auxiliary variables needed for the implementation.

##    input:
Several named inputs can be declared using the name of the buffer that should
receive the specified input during simulation. The input type can specified as spike
or current. A spike input can further be inhibitory, excitatory or both. Depending
on the sign of the input, incoming spikes are routed to the corresponding
sub-buffer. If no such modifier is given the buffer receives all spikes

##    output: 
Each neuron in NEST can just send one type of event during simulation.
NESTML supports spike or current output, which is specified after the keyword
output.

##    dynamics timestep(t ms):
The definition of the dynamics of a neuron is similar to that of a function. It starts with the keyword dynamics followed by the type of the dynamics. Depending
on the type, the function is called once per update step (timestep) or just once
per minimum delay interval in the simulated network (minDelay). A list of parameters
can be defined in parentheses
