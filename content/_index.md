---
title: "MTKNeuralToolkit.jl"
description: "Computational Neural Network Modeling in Julia"
type: "page"
layout: "single"
menu: main
---

# MTKNeuralToolkit.jl

[MTKNeuralToolkit.jl on GitHub](https://github.com/your-repo/MTKNeuralToolkit.jl)

1. [Installation](#installation)
2. [Quick Start](#quick-start)
3. [Neuron Models](#neuron-models)
4. [Synapses & Connectivity](#synapses)
5. [Examples](#examples)
6. [API Reference](#api-reference)
7. [Development Notes](#development)
8. [Performance](#performance)
9. [Contributing](#contributing)
10. [References](#references)

---

## Installation {#installation}


## Quick Start {#quick-start}

Build a simple two-neuron network:

```julia
using MTKNeuralToolkit
using OrdinaryDiffEq
using ModelingToolkitStandardLibrary.Blocks: TimeVaryingFunction

# Create neurons
@named input_stim = TimeVaryingFunction(f=t -> sin(t))
neurons = Dict(
    "pre" => build_HH(input_stim; name=:pre),
    "post" => build_HH(; name=:post)
)

# Define connections
connections = Dict(
    ("pre", "post") => [(type=:Exc, weight=0.5)]
)

# Build and simulate
network = build_network(connections, neurons)
prob = ODEProblem(network, [], (0.0, 100.0))
sol = solve(prob, TRBDF2())

# Extract voltages
voltages = parse_sol_for_membrane_voltages(sol)
```

[→ Detailed Quick Start Tutorial](quickstart/)

## Neuron Models {#neuron-models}

Three main neuron types available:

- **Hodgkin-Huxley**: `build_HH()` - Fast, 4-variable model
- **Liu/Prinz**: `build_Liu(), build_Prinz()` - 8-channel model with calcium dynamics  
- **IF/LIF**: `build_IF(), build_LIF()` - Fast, spiking neuron we all love.

Each model accepts custom configuration and optional input stimuli.

[→ Complete Neuron Models Guide](neuron-models/)

## Synapses & Connectivity {#synapses}

Chemical synapses with preset types:
- `:Exc` - Excitatory (E=0mV)
- `:Inh` - Inhibitory (E=-70mV)
- `:Chol` - Cholinergic (E=-80mV)
- `:Glut` - Glutamatergic (E=-70mV)

Custom synapses supported with user-defined kinetics.

[→ Synapses and Connectivity Guide](synapses/)

## Examples {#examples}

Real-world usage examples:

- Single neuron dynamics and channel behavior
- Small networks (2-25 neurons) with specific connectivity
- STG circuit - biological central pattern generator
- Custom components and gap junctions

[→ Examples and Use Cases](examples/)

## API Reference {#api-reference}

Complete function documentation:

- **Public API**: User-facing functions for building neurons and networks
- **Internal API**: Implementation details 

[→ Public API Reference](api-public/)  
[→ Internal API Reference](api-internal/)

## Development Notes {#development}

🚧 **Development Status**: This package is under active development. Features and APIs may change between versions.

Key considerations: Buggy -_-'
---

*Built with [ModelingToolkit.jl](https://github.com/SciML/ModelingToolkit.jl) and the Julia scientific computing ecosystem.*