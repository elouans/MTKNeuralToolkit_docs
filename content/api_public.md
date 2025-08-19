---

---
# Public API Reference

Complete reference for all functions intended for direct user interaction. These constitute the stable public API of MTKNeuralToolkit.

* * *

## Neuron Building Functions

### `build_IF`

    build_IF(input=nothing; name=:IF)

Build an Integrate-and-Fire neuron with threshold dynamics and reset mechanism.

**Parameters:**

* `input`: Optional stimulus (TimeVaryingFunction, Constant, etc.)
* `name`: Symbol for neuron identification (default: `:IF`)

**Returns:** ODESystem representing IF neuron with event-driven spiking

**Implementation Details:**

* Uses BasicSoma with C=10μF capacitance
* Implements threshold detection and reset mechanism

**Example:**

    # Basic IF neuron
    neuron = build_IF(; name=:if_neuron)
    
    # With stimulus
    @named stimulus = TimeVaryingFunction(f=t -> ifelse((t > 10) & (t < 20), 100.0, 0.0))
    neuron = build_IF(stimulus; name=:driven_neuron)

**Solver Requirement:** Use Rodas5() or other event-aware solvers for proper spike detection

* * *

### `build_LIF`

    build_LIF(input=nothing; name=:LIF)

Build a Leaky Integrate-and-Fire neuron with membrane leak current.

**Parameters:**

* `input`: Optional stimulus input
* `name`: Symbol for neuron identification (default: `:LIF`)

**Returns:** ODESystem representing LIF neuron with leak dynamics

**Implementation Details:**

* Uses BasicSoma with C=10μF capacitance
* Includes membrane leak for realistic decay dynamics

**Example:**

    # Basic LIF neuron
    neuron = build_LIF(; name=:lif_neuron)
    
    # With input stimulus
    neuron = build_LIF(stimulus; name=:driven_lif)

**Use Case:** Large-scale spiking networks, population dynamics

* * *

### `build_HH(input=nothing; name=:soma, config=HHConfig())`

Build a Hodgkin-Huxley neuron with Na, K, and leak channels.

**Parameters:**

* `input`: Optional stimulus (TimeVaryingFunction, Constant, etc.)
* `name`: Symbol for neuron identification
* `config`: HHConfig struct with channel parameters

**Returns:** ODESystem representing the complete neuron model

**Channel Composition:**

* Na channel: Fast sodium with m³h gating
* K channel: Delayed rectifier with n⁴ gating
* Leak channel: Passive conductance

**Example:**

    # Basic HH neuron
    neuron = build_HH(; name=:neuron1)
    
    # With stimulus
    @named stimulus = TimeVaryingFunction(f=t -> sin(t))
    neuron = build_HH(stimulus; name=:driven_neuron)
    
    # With custom parameters
    config = HHConfig(Na_g=120.0, K_g=36.0, C=1.0)
    neuron = build_HH(stimulus; name=:custom_neuron, config=config)

**Performance:** Fast simulation with Tsit5() solver

* * *

### `build_Liu`

    build_Liu(input=nothing; name=:soma, config=LiuConfig())

Build a Liu neuron model with calcium dynamics and 8 channel types.

**Parameters:**

* `input`: Optional stimulus input
* `name`: Symbol for neuron identification (default: `:soma`)
* `config`: LiuConfig struct with channel parameters (default: LiuConfig())

**Returns:** ODESystem with calcium-sensitive neuron model

**Channel Composition:**

* Na: Fast sodium
* K: Delayed rectifier potassium
* CaS: Slow calcium (persistent)
* CaT: Transient calcium (low-threshold)
* KCa: Calcium-activated potassium
* H: Hyperpolarization-activated cation
* DRK: Delayed rectifier K (additional)
* Leak: Passive conductance
* Uses CalciumSensitiveNeuron soma with C=1μF

**Example:**

    # Basic Liu neuron
    neuron = build_Liu(; name=:ca_neuron)
    
    # With enhanced calcium channels
    config = LiuConfig(CaS_g=6.0, KCa_g=5.0, H_g=0.01)
    neuron = build_Liu(; name=:enhanced_ca, config=config)

**Solver Requirement:** Use TRBDF2() for stiff calcium dynamics

* * *

### `build_Prinz`

    build_Prinz(input=nothing; name=:soma, config=PrinzConfig())

Build a Prinz STG neuron model optimized for central pattern generator networks.

**Parameters:**

* `input`: Optional stimulus input
* `name`: Symbol for neuron identification (default: `:soma`)
* `config`: PrinzConfig struct with STG-realistic parameters (default: PrinzConfig())

**Returns:** ODESystem optimized for network simulations

**Channel Composition:** Same as Liu model but with STG-optimized kinetics

* Tuned for realistic bursting behavior
* Optimized for network synchronization
* Uses CalciumSensitiveNeuron soma with configurable C, Ca0, V0
* Scaling factors typically applied (×159.2 for conductances)

**Example:**

    # STG burster neuron
    @named drive = TimeVaryingFunction(f=t -> 1.0)  # Constant input
    ab_neuron = build_Prinz(drive; name=:AB, config=PrinzConfig(V0=-60.0))
    
    # Network-ready configuration
    prinz_cf = 159.2  # Standard STG scaling
    config = PrinzConfig(
        Na_g=100.0*prinz_cf, 
        CaS_g=6.0*prinz_cf, 
        H_g=0.01*prinz_cf
    )
    neuron = build_Prinz(; name=:STG_neuron, config=config)

**Use Case:** Central pattern generators, rhythmic networks

* * *

## Network Assembly Functions

### `build_network`

    build_network(connections::Dict, neurons::Union{Vector,Dict})

Assemble a network from neurons and synaptic connections.

**Parameters:**

* `connections`: Dict mapping (pre, post) tuples to synapse specifications
* `neurons`: Vector or Dict of string->neuron mappings

**Returns:** Structurally simplified ODESystem representing the complete network

**Connection Format:**

    connections = Dict(
        ("neuron1", "neuron2") => [(type=:Exc, weight=0.5)],
        ("neuron2", "neuron3") => [(type=:Inh, weight=1.2), (type=:Chol, weight=0.3)]
    )

**Example:**

    # Define neurons
    neurons = Dict(
        "driver" => build_HH(stimulus; name=:driver),
        "follower" => build_HH(; name=:follower)
    )
    
    # Define connections
    connections = Dict(
        ("driver", "follower") => [(type=:Exc, weight=0.8)]
    )
    
    # Build network
    network = build_network(connections, neurons)

**Performance:** Recommended for networks <50 neurons

* * *

## Channel and Component Building

### `build_neuron`

    build_neuron(soma, input=nothing; channels)

Assemble neuron from soma and channel components.

**Parameters:**

* `soma`: Neuron soma component (BasicSoma, CalciumSensitiveNeuron)
* `input`: Optional input stimulus
* `channels`: Vector of channel ODESystems

**Returns:** Complete neuron ODESystem with proper connectivity

**Connection Logic:**

* Connects all channels to soma membrane
* Handles calcium flux for calcium-sensitive channels
* Manages input current injection
* Establishes proper grounding

**Example:**

    # Build custom neuron
    soma = BasicSoma(; C=1.0, name=:soma)
    channels = [na_channel, k_channel, leak_channel]
    neuron = build_neuron(soma, stimulus; channels=channels)

## Configuration Structures

### `IFConfig`

Configuration parameters for Integrate-and-Fire neurons.

**Fields:**

* `V_reset`: Reset voltage after spike (default: -2.0 mV)
* `V_th`: Spike threshold voltage (default: 10.0 mV)
* `R`: Membrane resistance (default: 1.0 MΩ)
* `C`: Membrane capacitance (default: 10.0 μF)
* `E`: Reversal potential (default: 0.0 mV)

**Example:**

    # Fast-spiking IF neuron
    config = IFConfig(V_th=5.0, V_reset=-5.0, R=0.5, C=5.0)
    neuron = build_IF(; config=config)

* * *

### `LIFConfig`

Configuration parameters for Leaky Integrate-and-Fire neurons.

**Fields:**

* `V_reset`: Reset voltage after spike (default: -2.0 mV)
* `V_th`: Spike threshold voltage (default: 10.0 mV)
* `τ_m`: Membrane time constant (default: 1.0 ms)
* `R`: Membrane resistance (default: 1.0 MΩ)
* `C`: Membrane capacitance (default: 10.0 μF)
* `E`: Reversal potential (default: 0.0 mV)

**Example:**

    # Slow, leaky neuron
    config = LIFConfig(τ_m=5.0, V_th=15.0, R=2.0)
    neuron = build_LIF(; config=config)

* * *

Configuration parameters for Hodgkin-Huxley neurons.

**Fields:**

* `Na_g`, `K_g`, `Leak_g`: Channel conductances (mS/cm²)
* `Na_E`, `K_E`, `Leak_E`: Reversal potentials (mV)
* `C`: Membrane capacitance (μF/cm²)
* `V0`: Initial voltage (mV)

**Defaults:**

    HHConfig(
        V0=-65.0,
        Na_g=120.0, Na_E=50.0,
        K_g=36.0, K_E=-77.0,
        Leak_g=0.3, Leak_E=-54.4,
        C=1.0
    )

**Example:**

    # High-excitability configuration
    config = HHConfig(Na_g=150.0, K_g=30.0, C=0.8)
    neuron = build_HH(; config=config)

* * *

### `LiuConfig`

Configuration parameters for Liu neurons with calcium dynamics.

**Fields:**

* `Na_g`, `K_g`, `KCa_g`, `DRK_g`, `H_g`, `Leak_g`: Channel conductances
* `CaS_g`, `CaT_g`: Calcium channel conductances
* Reversal potentials for all channels
* `C`: Capacitance, `V0`/`Ca0`: Initial conditions

**Calcium-Specific Parameters:**

* `Ca0`: Initial calcium concentration
* Calcium reversal computed dynamically from Nernst equation

**Example:**

    # Enhanced calcium dynamics
    config = LiuConfig(
        CaS_g=8.0,      # Increased slow calcium
        KCa_g=7.0,      # Enhanced calcium-activated K
        H_g=0.02,       # Stronger h-current
        V0=-60.0        # Depolarized rest
    )
    neuron = build_Liu(; config=config)

* * *

### `PrinzConfig`

Configuration parameters for Prinz STG neurons.

**Fields:** Same structure as LiuConfig with STG-optimized defaults

**STG Scaling:** Typically multiply conductances by 159.2 for realistic STG behavior

    prinz_cf = 159.2
    config = PrinzConfig(
        V0=-60.0,
        Na_g=100.0*prinz_cf,
        CaS_g=6.0*prinz_cf,
        H_g=0.01*prinz_cf
    )

**Neuron-Specific Configurations:**

    # AB (Anterior Burster) neuron
    ab_config = PrinzConfig(V0=-60.0, H_g=0.01*prinz_cf)
    
    # PY (Pyloric) neuron  
    py_config = PrinzConfig(V0=-55.0, KCa_g=0.0*prinz_cf)
    
    # LP (Lateral Pyloric) neuron
    lp_config = PrinzConfig(V0=-65.0, CaT_g=0.0*prinz_cf)

* * *

## Type Constants and Validation

### `SYNAPSE_TYPES`

Vector of valid synapse type symbols:

    SYNAPSE_TYPES = [:Exc, :Inh, :Chol, :Glut, :Custom]

**Usage in validation:**

    is_valid_synapse(synapse_type) = synapse_type in SYNAPSE_TYPES

* * *

## Complete Example: STG Network

    using MTKNeuralToolkit, OrdinaryDiffEq
    using ModelingToolkitStandardLibrary.Blocks: TimeVaryingFunction
    
    # Scaling factors
    syn_cf = 0.254
    prinz_cf = 159.2
    
    # Create tonic drive
    @named tonic_drive = TimeVaryingFunction(f=t -> 1.0)
    
    # Define STG neurons with realistic parameters
    neurons = Dict(
        "AB" => build_Prinz(tonic_drive; name=:AB, config=PrinzConfig(
            V0=-60.0, Na_g=100.0*prinz_cf, CaS_g=6.0*prinz_cf, 
            H_g=0.01*prinz_cf, K_g=50.0*prinz_cf)),
        "PY" => build_Prinz(; name=:PY, config=PrinzConfig(
            V0=-55.0, Na_g=100.0*prinz_cf, CaS_g=2.0*prinz_cf, 
            KCa_g=0.0*prinz_cf)),
        "LP" => build_Prinz(; name=:LP, config=PrinzConfig(
            V0=-65.0, Na_g=100.0*prinz_cf, K_g=20.0*prinz_cf, 
            CaT_g=0.0*prinz_cf))
    )
    
    # STG connectivity pattern
    connections = Dict(
        ("AB", "LP") => [(type=:Chol, weight=30.0*syn_cf), (type=:Glut, weight=30.0*syn_cf)],
        ("AB", "PY") => [(type=:Chol, weight=3.0*syn_cf), (type=:Glut, weight=10.0*syn_cf)],
        ("LP", "AB") => [(type=:Glut, weight=30.0*syn_cf)],
        ("LP", "PY") => [(type=:Glut, weight=1.0*syn_cf)],
        ("PY", "LP") => [(type=:Glut, weight=30.0*syn_cf)]
    )
    
    # Build and simulate network
    network = build_network_split(connections, neurons)
    prob = ODEProblem(network, [], (0.0, 100.0))
    sol = solve(prob, TRBDF2())
    
    # Analyze results
    voltages = parse_sol_for_membrane_voltages(sol)
    plot(sol, idxs=voltages, labels=["AB" "PY" "LP"])
    
    # Network inspection
    result = inspect_network(network)
    println("STG network: $(length(result.neurons)) neurons")

This creates a complete three-neuron central pattern generator producing realistic pyloric rhythm dynamics.