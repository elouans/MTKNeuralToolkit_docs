---
title: "Public API Reference"
description: "Complete reference for user-facing functions"
featured_image: ""
menu:
  main:
    parent: "Reference"
weight: 10
---

# Public API Reference

Complete reference for all functions intended for direct user interaction. These constitute the stable public API of MTKNeuralToolkit.

---

## Neuron Building Functions

### `build_HH`

```julia
build_HH(input=nothing; name=:soma, config=HHConfig())
```

Build a Hodgkin-Huxley neuron with Na, K, and leak channels.

**Parameters:**
- `input`: Optional stimulus (TimeVaryingFunction, Constant, etc.)
- `name`: Symbol for neuron identification
- `config`: HHConfig struct with channel parameters

**Returns:** ODESystem representing the complete neuron model

**Example:**
```julia
neuron = build_HH(stimulus; name=:neuron1)
```

---

### `build_Liu`

```julia
build_Liu(input=nothing; name=:soma, config=LiuConfig())
```

Build a Liu neuron model with calcium dynamics and 8 channel types.

**Parameters:**
- `input`: Optional stimulus input
- `name`: Symbol for neuron identification  
- `config`: LiuConfig struct with channel parameters

**Returns:** ODESystem with calcium-sensitive neuron model

**Example:**
```julia
neuron = build_Liu(; name=:ca_neuron, config=LiuConfig(CaS_g=6.0))
```


---

### `build_Prinz`

```julia
build_Prinz(input=nothing; name=:soma, config=PrinzConfig())
```

Build a Prinz STG neuron model optimized for central pattern generator networks.

**Parameters:**
- `input`: Optional stimulus input
- `name`: Symbol for neuron identification
- `config`: PrinzConfig struct with STG-realistic parameters

**Returns:** ODESystem optimized for network simulations

**Example:**
```julia
ab_neuron = build_Prinz(drive; name=:AB, config=PrinzConfig(V0=-60.0))
```


---

## Network Assembly Functions

### `build_network`

```julia
build_network(connections::Dict, neurons::Union{Vector,Dict})
```

Assemble a network from neurons and synaptic connections.

**Parameters:**
- `connections`: Dict mapping (pre, post) tuples to synapse specifications
- `neurons`: Vector or Dict of string->neuron

**Returns:** Structurally simplified ODESystem representing the complete network

**Example:**
```julia
network = build_network(connections, neurons)
```

**Performance:** Recommended for networks <50 neurons

---

### `build_network_split`

```julia
build_network_split(connections::Dict, neurons::Union{Vector,Dict})
```

Build network with individual component simplification for possibly better performance.

**Parameters:**
- `connections`: Dict mapping (pre, post) tuples to synapse specifications
- `neurons`: Dict of string->neuron or Vector of neurons

**Returns:** Structurally simplified ODESystem with pre-optimized components

**Example:**
```julia
network = build_network_split(connections, neurons)
```


---

### `put_synapse`

```julia
put_synapse(pre, post, synapse_type::Symbol, weight::Float64; kwargs...)
```

Create and connect a synapse between two neurons.

**Parameters:**
- `pre`: Presynaptic neuron ODESystem
- `post`: Postsynaptic neuron ODESystem  
- `synapse_type`: `:Exc`, `:Inh`, `:Chol`, `:Glut`, or `:Custom`
- `weight`: Synaptic conductance (positive float)
- `name`: Optional synapse identifier
- `E`, `Vth`, `k_`, `sigma`: Custom parameters for `:Custom` type

**Returns:** ODESystem with connected synapse

**Example:**
```julia
syn = put_synapse(n1, n2, :Exc, 0.5; name=:connection)
```

---

## Solution Analysis Functions

### `parse_sol_for_membrane_voltages`

```julia
parse_sol_for_membrane_voltages(sol::ODESolution)
```

Extract unique membrane voltage traces from network simulation results.

**Parameters:**
- `sol`: ODESolution from network simulation

**Returns:** Vector of voltage state variables, one per neuron (avoiding duplicates)

**Example:**
```julia
voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages)
```

**Warning:** Handles multiple synapse connections to same neuron correctly

---

### `parse_sol_for_voltage`

```julia
parse_sol_for_voltage(state_vars)
```

Internal helper for voltage extraction from state variable list.

**Parameters:**
- `state_vars`: Vector of symbolic variables from ODESystem

**Returns:** Vector of voltage state variables

**Warning:** Typically called by `parse_sol_for_membrane_voltages()`

---

## Debugging Tools

### `inspect_network`

```julia
inspect_network(prob::Union{ODEProblem,ODESystem})
```

Display network structure including neurons and synaptic connections.

**Parameters:**
- `prob`: ODEProblem or ODESystem to analyze

**Returns:** NamedTuple with `(neurons, neuron_synapses, all_states)`

**Example:**
```julia
result = inspect_network(network)
println("Neurons: ", length(result.neurons))
```

**Use Case:** Debugging network topology and verifying connections

---

## Channel and Component Building

### `build_channel`

```julia
build_channel(conductance, reversal; name)
```

Build ion channel from conductance and reversal potential components.

**Parameters:**
- `conductance`: Gate dynamics component (e.g., NaGates, KGates)
- `reversal`: Reversal potential component (e.g., FixedReversal)
- `name`: Symbol for channel identification

**Returns:** ODESystem representing complete ion channel

**Example:**
```julia
na_channel = build_channel(HH.NaGates(;g=120), FixedReversal(;E=50); name=:Na)
```

---

### `build_neuron`

```julia
build_neuron(soma, input=nothing; channels)
```

Assemble neuron from soma and channel components.

**Parameters:**
- `soma`: Neuron soma component (BasicSoma, CalciumSensitiveNeuron)
- `input`: Optional input stimulus
- `channels`: Vector of channel ODESystems

**Returns:** Complete neuron ODESystem

**Example:**
```julia
neuron = build_neuron(soma, stimulus; channels=[na_ch, k_ch, leak_ch])
```

---

### `add_synapse`

```julia
add_synapse(channel, pre_neuron, post_neuron; debug=false)
```

Connect synapse channel between two neurons.

**Parameters:**
- `channel`: Synapse component
- `pre_neuron`: Presynaptic neuron ODESystem
- `post_neuron`: Postsynaptic neuron ODESystem
- `debug`: Print connection information

**Returns:** ODESystem with synaptic connection

**Warning:** Lower-level function; prefer `put_synapse()` for typical use

---

## Configuration Structures

### `HHConfig`

Configuration parameters for Hodgkin-Huxley neurons.

**Fields:**
- `Na_g`, `K_g`, `Leak_g`: Channel conductances (mS/cm²)
- `Na_E`, `K_E`, `Leak_E`: Reversal potentials (mV)
- `C`: Membrane capacitance (μF/cm²)
- `V0`: Initial voltage (mV)

**Example:**
```julia
config = HHConfig(Na_g=120.0, K_g=36.0, C=1.0)
```

---

### `LiuConfig`

Configuration parameters for Liu neurons with calcium dynamics.

**Fields:**
- `Na_g`, `K_g`, `KCa_g`, `DRK_g`, `H_g`, `Leak_g`: Channel conductances
- `CaS_g`, `CaT_g`: Calcium channel conductances
- Reversal potentials for all channels
- `C`: Capacitance, `V0`/`Ca0`: Initial conditions

**Example:**
```julia
config = LiuConfig(CaS_g=6.0, KCa_g=5.0, H_g=0.01)
```

---

### `PrinzConfig`

Configuration parameters for Prinz STG neurons.

**Fields:**
- Same structure as LiuConfig
- Optimized defaults for STG network simulations
- Scaling factors typically applied (×159.2 for conductances)

**Example:**
```julia
config = PrinzConfig(V0=-60.0, Na_g=100.0*159.2)
```

---

## Type Constants

### `SYNAPSE_TYPES`

Vector of valid synapse type symbols: `[:Exc, :Inh, :Chol, :Glut, :Custom]`


**Next:** Explore [Internal Functions](/api_internal.md/) for advanced customization and development.