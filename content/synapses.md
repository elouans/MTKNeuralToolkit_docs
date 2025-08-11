---
title: "Synapses and Connectivity"
description: "Chemical synapses, network connectivity, and synaptic transmission"
featured_image: ""
menu:
  main:
    parent: "Tutorials"
weight: 30
---

# Synapses and Connectivity

MTKNeuralToolkit implements voltage-gated chemical synapses with sigmoid activation and first-order kinetics.

## Synaptic Transmission Model

All chemical synapses follow the same general mathematical framework:

### General Equations

```
s_hat = 1 / (1 + exp((V_th - V_pre) / σ))
τ_s = (1 - s_hat) / k
ds/dt = (s_hat - s) / τ_s
I_post = g * s * (V_post - E_rev)
```

### Parameters

- **g**: Synaptic conductance (strength)
- **E_rev**: Reversal potential (determines exc/inh)
- **V_th**: Threshold voltage for activation  
- **k**: Kinetic rate parameter
- **σ**: Sigmoid steepness

## Preset Synapse Types

Four common synapse types are provided with biologically realistic parameters:

| Type | Symbol | Reversal | Kinetics | Description | Usage |
|------|--------|----------|----------|-------------|-------|
| **Excitatory** | `:Exc` | 0.0 mV | k = 0.025 | Fast excitatory synapse, typical of glutamatergic | General excitatory coupling |
| **Inhibitory** | `:Inh` | -70.0 mV | k = 0.01 | Slower inhibitory synapse, typical of GABAergic | General inhibitory coupling |
| **Cholinergic** | `:Chol` | -80.0 mV | k = 0.01 | Cholinergic synapse with hyperpolarizing reversal | Neuromodulatory effects, STG networks |
| **Glutamatergic** | `:Glut` | -70.0 mV | k = 0.025 | Fast glutamatergic synapse | Rapid excitatory transmission |

## Basic Connectivity

### Simple Connection

Create connections using a dictionary mapping neuron pairs to synapse specifications:

```julia
# Define neuron pairs and synapse types
connections = Dict(
    ("neuron1", "neuron2") => [(type=:Exc, weight=0.5)],
    ("neuron2", "neuron1") => [(type=:Inh, weight=0.3)]
)
```

Neurons can be referenced and passed to build_network through array indices or dict keys.

### Multiple Synapses Between Same Pair

```julia
# Multiple synapses between same pair
connections = Dict(
    ("AB", "LP") => [
        (type=:Chol, weight=30.0),
        (type=:Glut, weight=30.0)
    ]
)
```

### Connection Format

Each connection specifies:
- **Presynaptic neuron**: String identifier
- **Postsynaptic neuron**: String identifier
- **Synapse type**: Symbol (`:Exc`, `:Inh`, `:Chol`, `:Glut`)
- **Weight**: Conductance value (positive number)

## Building Connected Networks

### Standard Workflow

1. **Create neurons**, either as dicts or arrays.
2. **Define connectivity** as dictionary of connections
3. **Build network** using `build_network()`
4. **Simulate** with appropriate solver
5. **Take** a **Nap**

### Complete Example

```julia
using MTKNeuralToolkit, OrdinaryDiffEq
using ModelingToolkitStandardLibrary.Blocks: TimeVaryingFunction

# Step 1: Create neurons
@named stimulus = TimeVaryingFunction(f=t -> sin(t))
neurons = Dict(
    "driver" => build_HH(stimulus; name=:driver),
    "follower1" => build_Prinz(; name=:follower1),
    "follower2" => build_Prinz(; name=:follower2)
)

# Step 2: Define connectivity
connections = Dict(
    ("driver", "follower1") => [(type=:Exc, weight=1.0)],
    ("driver", "follower2") => [(type=:Exc, weight=0.8)],
    ("follower1", "follower2") => [(type=:Inh, weight=0.5)]
)

# Step 3: Build network
network = build_network(connections, neurons)

# Step 4: Simulate
prob = ODEProblem(network, [], (0.0, 100.0))
sol = solve(prob, TRBDF2())

# Extract and plot voltages
voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages, labels=["Driver" "Follower1" "Follower2"])
```

## Custom Synapse Parameters

Create synapses with custom biophysical parameters using `put_synapse()`:

```julia
# Create custom synapse between two neurons
custom_syn = put_synapse(
    pre_neuron, post_neuron, :Custom, weight=1.5;
    E=-65.0,        # Custom reversal potential
    Vth=-30.0,      # Custom threshold
    k_=0.05,        # Custom kinetics
    sigma=3.0,      # Custom steepness
    name=:my_synapse
)
```

### Parameter Effects

- **E_rev**: More positive = excitatory, more negative = inhibitory
- **V_th**: Lower threshold = easier activation
- **k_**: Higher k = faster kinetics
- **sigma**: Lower sigma = sharper activation curve


This is in progress, plans are to change this functionality soon to allow for integration with build_network. 

## Custom Synapses Altogether

Like for anything else in this package, you can also make your own synapses altogether:

```julia
@mtkmodel GapJunction begin
    @extend v_pre, v_post, i_post, i_pre = twoport = BiDirectionalTwoPort()
    @parameters begin
        g_gap, [description = "Gap junction conductance"]
        V_threshold = -20.0
        k_gate = 10.0
    end
    @equations begin
        i_post ~ g_gap / (1.0 + exp(k_gate * (abs(v_pre - v_post) - V_threshold))) * (v_pre - v_post)
        i_pre ~ -i_post
    end
end

# Usage in network
neurons = [build_HH(;name=:n1), build_HH(;name=:n2)]
connections = Dict(
    (1,2) => [() -> GapJunction(;g_gap=30.0, name=:gap)]
)
network = build_network(connections, neurons)
```

## Common Network Patterns

### Feedforward Network

Unidirectional excitatory chain:

```julia
connections = Dict(
    ("input", "hidden1") => [(type=:Exc, weight=1.0)],
    ("hidden1", "hidden2") => [(type=:Exc, weight=0.8)],
    ("hidden2", "output") => [(type=:Exc, weight=0.6)]
)
```

### Mutual Inhibition

Competitive inhibitory network:

```julia
connections = Dict(
    ("neuron1", "neuron2") => [(type=:Inh, weight=2.0)],
    ("neuron2", "neuron1") => [(type=:Inh, weight=2.0)]
)
```

### Central Pattern Generator

STG-style rhythmic network:

```julia
connections = Dict(
    ("AB", "LP") => [(type=:Chol, weight=30.0), (type=:Glut, weight=30.0)],
    ("AB", "PY") => [(type=:Chol, weight=3.0), (type=:Glut, weight=10.0)],
    ("LP", "PY") => [(type=:Glut, weight=1.0)],
    ("PY", "LP") => [(type=:Glut, weight=30.0)]
)
```

## Network Building Options

Two network building functions are available:

### Standard Assembly

```julia
network = build_network(connections, neurons)
```
- **Good for**: <50 neurons
- **Process**: Assemble then simplify entire network

### Split Assembly  

```julia
network = build_network_split(connections, neurons)
```
- **Good for**: >50 neurons
- **Process**: Simplify neurons first, then synapses, then network
- **Benefit**: Better performance for larger networks

## Debugging Connectivity

### Network Inspection

Examine network structure and connections:

```julia
# Inspect built network
result = inspect_network(network)
println("Neurons: ", length(result.neurons))
println("Synapses per neuron: ", result.neuron_synapses)
```

### Voltage Extraction

Extract clean voltage traces (avoids duplicates from multiple connections):

```julia
# Get one voltage trace per neuron
voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages)
```

## Example: STG Circuit

A complete three-neuron central pattern generator:

```julia
# STG neurons with realistic parameters
syn_cf = 0.254      # Synapse scaling
prinz_cf = 159.2    # Prinz model scaling

@named tonic_drive = TimeVaryingFunction(f=t -> 1.0)

neurons = Dict(
    "AB" => build_Prinz(tonic_drive; name=:AB, config=PrinzConfig(
        V0=-60.0, Na_g=100.0*prinz_cf, CaS_g=6.0*prinz_cf, 
        H_g=0.01*prinz_cf)),
    "PY" => build_Prinz(; name=:PY, config=PrinzConfig(
        V0=-55.0, Na_g=100.0*prinz_cf, CaS_g=2.0*prinz_cf)),
    "LP" => build_Prinz(; name=:LP, config=PrinzConfig(
        V0=-65.0, Na_g=100.0*prinz_cf, K_g=20.0*prinz_cf))
)

# STG connectivity pattern
connections = Dict(
    ("AB", "LP") => [(type=:Chol, weight=30.0*syn_cf), (type=:Glut, weight=30.0*syn_cf)],
    ("AB", "PY") => [(type=:Chol, weight=3.0*syn_cf), (type=:Glut, weight=10.0*syn_cf)],
    ("LP", "AB") => [(type=:Glut, weight=30.0*syn_cf)],
    ("LP", "PY") => [(type=:Glut, weight=1.0*syn_cf)],
    ("PY", "LP") => [(type=:Glut, weight=30.0*syn_cf)]
)

network = build_network_split(connections, neurons)
prob = ODEProblem(network, [], (0.0, 100.0))
sol = solve(prob, TRBDF2())
```

This produces realistic pyloric rhythm characteristic of the lobster STG.

---

**Next:** Explore complete [examples](/examples/) showing these concepts in action.