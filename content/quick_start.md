---
title: "Quick Start Guide"
description: "Get started with MTKNeuralToolkit"
featured_image: ""
menu:
  main:
    parent: "Tutorials"
weight: 10
---

# Quick Start Guide

Get up and running with MTKNeuralToolkit in minutes.

## Prerequisites

- Julia 1.6+ (1.9+ recommended)
- Basic familiarity with differential equations
- Understanding of neuronal membrane dynamics (helpful but not required)

### Optional Background
- Hodgkin-Huxley formalism
- ModelingToolkit.jl concepts
- OrdinaryDiffEq.jl solving methods

## Installation

```julia
using Pkg
Pkg.add("MTKNeuralToolkit")

# Development dependencies for plotting
Pkg.add(["OrdinaryDiffEq", "Plots", "ModelingToolkitStandardLibrary"])
```

## Your First Neuron

Let's create and simulate a single Hodgkin-Huxley neuron:

```julia
using MTKNeuralToolkit
using OrdinaryDiffEq
using ModelingToolkitStandardLibrary.Blocks: TimeVaryingFunction
using Plots

# Create input stimulus
@named stimulus = TimeVaryingFunction(f=t -> sin(t))

# Build HH neuron with stimulus
neuron = build_HH(stimulus; name=:neuron)

# Simplify the system
neuron_simplified = structural_simplify(neuron)

# Create and solve ODE problem
prob = ODEProblem(neuron_simplified, [], (0.0, 50.0))
sol = solve(prob, Tsit5())

# Plot membrane voltage
plot(sol, idxs=[neuron.soma.v], xlabel="Time (ms)", ylabel="Voltage (mV)")
```

This creates a Hodgkin-Huxley neuron with sinusoidal input current and plots the resulting membrane voltage oscillations.

## Building a Simple Network

Create a two-neuron network with synaptic coupling:

```julia
# Define neurons
neurons = Dict(
    "exciter" => build_HH(stimulus; name=:exciter),
    "target" => build_HH(; name=:target)
)

# Define synaptic connections
connections = Dict(
    ("exciter", "target") => [(type=:Exc, weight=0.8)]
)

# Build network
network = build_network(connections, neurons)

# Simulate
prob = ODEProblem(network, [], (0.0, 100.0))
sol = solve(prob, TRBDF2())

# Plot all membrane voltages
voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages, labels=["Exciter" "Target"])
```

The excitatory synapse (`:Exc`) allows the driven neuron to influence the target neuron's activity.

## Synapse Types

Four common synapse types are available:

- **`:Exc`** - Excitatory (E=0mV)
- **`:Inh`** - Inhibitory (E=-70mV) 
- **`:Chol`** - Cholinergic (E=-80mV)
- **`:Glut`** - Glutamatergic (E=-70mV)

## Typical Workflow

1. **Define inputs**: Create stimulus using `TimeVaryingFunction`
2. **Build neurons**: Use `build_HH()`, `build_Liu()`, or `build_Prinz()`
3. **Define connectivity**: Create connections dictionary
4. **Assemble network**: Call `build_network()`
5. **Create ODE problem**: Use `ODEProblem()`
6. **Solve**: Choose appropriate solver (Tsit5 for HH, TRBDF2 for Liu/Prinz)
7. **Extract results**: Use `parse_sol_for_membrane_voltages()`

## Example: Configuring Neuron Parameters

```julia
# Custom HH neuron parameters
using MTKNeuralToolkit.Config
config = HHConfig(
    Na_g=120.0,    # Sodium conductance
    K_g=36.0,      # Potassium conductance
    Leak_g=0.3,    # Leak conductance
    C=1.0          # Capacitance
)

neuron = build_HH(stimulus; name=:custom_neuron, config=config)
```

## Next Steps

- Learn about [Neuron Models](/neuron-models/) in detail
- Explore [Synapse Types](/synapses/) and their parameters
- See [Examples](/examples/) for realistic networks
- Check the [API Reference](/api-public/) for complete function documentation

---

**Ready to build more complex networks?** Check out our [STG circuit example](/examples/stg/) to see a three-neuron central pattern generator in action.