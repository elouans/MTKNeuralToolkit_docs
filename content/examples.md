---
title: "Examples and Use Cases"
description: "Real-world examples from single neurons to complex networks"
featured_image: ""
menu:
  main:
    parent: "Examples"
weight: 10
---

# Examples and Use Cases

Real-world examples demonstrating MTKNeuralToolkit capabilities, from single neurons to complex networks.

## Example Categories

- **[Single Neuron Models](#single-neurons)**: Basic dynamics and channel behavior
- **[Small Networks](#small-networks)**: 2-5 neuron circuits with specific connectivity  
- **[Biological Circuits](#biological-circuits)**: STG and other well-characterized networks
- **[Custom Components](#custom-components)**: Building novel synapse and neuron types

---

## Single Neurons {#single-neurons}

### Hodgkin-Huxley Dynamics

Classic model showing action potential generation and channel kinetics.

```julia
using MTKNeuralToolkit, OrdinaryDiffEq, Plots
using ModelingToolkitStandardLibrary.Blocks: TimeVaryingFunction

# Create stimulus and neuron
@named stimulus = TimeVaryingFunction(f=t -> sin(t))
neuron = build_HH(stimulus; name=:neuron)
neuron = structural_simplify(neuron)

# Simulate
prob = ODEProblem(neuron, [], (0.0, 200.0))
sol = solve(prob, Tsit5())

# Plot voltage and channel states
p = plot(layout=(4,1), size=(800,600))
plot!(p, sol, idxs=[neuron.Na.conductance.m_gate, neuron.Na.conductance.h_gate], 
      subplot=1, title="Na Channel Gates")
plot!(p, sol, idxs=[neuron.K.conductance.n_gate], 
      subplot=2, title="K Channel Gate")  
plot!(p, sol, idxs=[neuron.soma.v], 
      subplot=3, title="Membrane Voltage")
```

**Key Features:**
- Action potential generation
- Na/K channel dynamics
- Phase relationships between gates
- Fast simulation with Tsit5 solver

### Liu Model with Calcium Dynamics

Detailed model showing calcium-dependent processes and multiple channel types.

```julia
# Liu neuron with complex stimulus
@named stimulus = TimeVaryingFunction(f=t -> exp(sin(t)*sin(t)))
neuron = build_Liu(stimulus; name=:neuron)
neuron = structural_simplify(neuron)

# Simulate with stiff solver
prob = ODEProblem(neuron, [], (0.0, 400.0))
sol = solve(prob, TRBDF2(), maxiters=1e9)

# Plot comprehensive channel analysis
p = plot(layout=(6,1), size=(1000,1200))
plot!(p, sol, idxs=[neuron.Na.conductance.m, neuron.Na.conductance.h], 
      subplot=1, title="Na Channel")
plot!(p, sol, idxs=[neuron.KCa.conductance.m], 
      subplot=2, title="Ca-activated K")
plot!(p, sol, idxs=[neuron.CaS.conductance.m, neuron.CaS.conductance.h], 
      subplot=3, title="Slow Ca Channel")
plot!(p, sol, idxs=[neuron.soma.v], 
      subplot=4, title="Membrane Voltage")
plot!(p, sol, idxs=[neuron.soma.Ca], 
      subplot=5, title="Ca Concentration")
plot!(p, sol, idxs=[neuron.soma.ca.i], 
      subplot=6, title="Ca Current")
```

**Key Features:**
- 8-channel model complexity
- Calcium dynamics and feedback
- Multiple timescale interactions  
- Requires TRBDF2 stiff solver

---

## Small Networks {#small-networks}

### Two-Neuron Excitatory Coupling

Demonstrates synaptic transmission and network building basics.

```julia
# Create connected HH neurons
@named stimulus = TimeVaryingFunction(f=t -> sin(t))
neurons = Dict(
    "driver" => build_HH(stimulus; name=:driver),
    "follower" => build_HH(; name=:follower)
)

connections = Dict(
    ("driver", "follower") => [(type=:Exc, weight=0.5)]
)

# Build and simulate network
network = build_network(connections, neurons)
prob = ODEProblem(network, [], (0.0, 100.0))
sol = solve(prob, TRBDF2())

# Plot both neurons
voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages, labels=["Driver" "Follower"])
```

**Demonstrates:**
- Basic network assembly
- Synaptic transmission delays
- Voltage trace extraction
- Connection specification format

### Mutual Inhibition Network

Two neurons with reciprocal inhibitory connections, showing competitive dynamics.

```julia
# Competing inhibitory neurons
neurons = Dict(
    "neuron1" => build_Prinz(; name=:neuron1),
    "neuron2" => build_Prinz(; name=:neuron2)
)

connections = Dict(
    ("neuron1", "neuron2") => [(type=:Inh, weight=2.0)],
    ("neuron2", "neuron1") => [(type=:Inh, weight=2.0)]
)

network = build_network(connections, neurons)
prob = ODEProblem(network, [], (0.0, 500.0))
sol = solve(prob, TRBDF2())

voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages, labels=["Neuron 1" "Neuron 2"])
```

**Demonstrates:**
- Competitive network dynamics
- Reciprocal inhibition
- Winner-take-all behavior
- Prinz model usage

---

## Biological Circuits {#biological-circuits}

### Stomatogastric Ganglion (STG) Circuit

Three-neuron central pattern generator producing realistic pyloric rhythm.

```julia
using MTKNeuralToolkit.Config

# STG neurons with realistic parameters
syn_cf = 0.254      # Synapse scaling
prinz_cf = 159.2    # Prinz model scaling

@named tonic_drive = TimeVaryingFunction(f=t -> 1.0)  # Constant input

neurons = Dict(
    "AB" => build_Prinz(tonic_drive; name=:AB, config=PrinzConfig(
        V0=-60.0, Na_g=100.0*prinz_cf, CaS_g=6.0*prinz_cf, 
        CaT_g=2.5*prinz_cf, H_g=0.01*prinz_cf, K_g=50.0*prinz_cf, 
        KCa_g=5.0*prinz_cf, DRK_g=100.0*prinz_cf, Leak_g=0.0*prinz_cf)),
    "PY" => build_Prinz(; name=:PY, config=PrinzConfig(
        V0=-55.0, Na_g=100.0*prinz_cf, CaS_g=2.0*prinz_cf, 
        CaT_g=2.4*prinz_cf, H_g=0.05*prinz_cf, K_g=50.0*prinz_cf, 
        KCa_g=0.0*prinz_cf, DRK_g=125.0*prinz_cf, Leak_g=0.01*prinz_cf)),
    "LP" => build_Prinz(; name=:LP, config=PrinzConfig(
        V0=-65.0, Na_g=100.0*prinz_cf, CaS_g=4.0*prinz_cf, 
        CaT_g=0.0*prinz_cf, H_g=0.05*prinz_cf, K_g=20.0*prinz_cf, 
        KCa_g=0.0*prinz_cf, DRK_g=25.0*prinz_cf, Leak_g=0.03*prinz_cf))
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

voltages = parse_sol_for_membrane_voltages(sol)
plot(sol, idxs=voltages, labels=["AB" "PY" "LP"])
```

**Biological Significance:**
- Central pattern generator for rhythmic behavior
- Well-characterized lobster neural circuit
- Multiple synapse types (cholinergic + glutamatergic)
- Realistic conductance scaling
- Demonstrates `build_network_split()` for performance

---

## Custom Components {#custom-components}

### Gap Junction Implementation

Bidirectional electrical coupling with voltage-dependent gating.

```julia
@mtkmodel GapJunction begin
    @extend v_pre, v_post, i_post, i_pre = twoport = BiDirectionalTwoPort()
    @parameters begin
        g_gap, [description = "Gap junction conductance"]
        V_threshold = -20.0, [description = "Voltage threshold for gating"]
        k_gate = 10.0, [description = "Gating sensitivity"]
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

prob = ODEProblem(network, [], (0.0, 500.0))
sol = solve(prob, TRBDF2())
```

**Features:**
- Bidirectional current flow
- Voltage-dependent gating  
- Custom MTK model definition

### Custom Chemical Synapse

Modifying synapse parameters for specific applications.

```julia
# Custom synapse with specific kinetics
custom_syn = put_synapse(
    pre_neuron, post_neuron, :Custom, weight=1.5;
    E=-65.0,        # Intermediate reversal potential
    Vth=-30.0,      # Lower activation threshold
    k_=0.05,        # Slower kinetics
    sigma=3.0,      # Gentler activation curve
    name=:modulatory_synapse
)
```


## Example Files in Repository

The MTKNeuralToolkit repository contains several example scripts:

- **`plot_HH.jl`**: Basic Hodgkin-Huxley neuron simulation
- **`plot_Liu.jl`**: Detailed Liu model with calcium dynamics
- **`plot_stg.jl`**: Complete STG circuit implementation
- **`plot_HH_network.jl`**: Multi-neuron HH network examples
- **`plot_custom_synapse.jl`**: Gap junction implementation
- **`tests_400hh.jl`**: Large-scale network performance test


---

**Ready to explore the API?** Check out the [Public API Reference](api_public/) for complete function documentation.