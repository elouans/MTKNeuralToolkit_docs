---
title: "Neuron Models"
description: "Detailed guide to HH, Liu, and Prinz neuron models"
featured_image: ""
menu:
  main:
    parent: "Tutorials"
weight: 20
---

# Neuron Models

MTKNeuralToolkit natively implements three major neuron model families, each with different levels of biophysical detail and computational requirements:
Hodgkin-Huxley neurons.
Liu/Prinz neurons.
Integrate-and-Fire neurons.

The intention here is not to cover the extent of what you will be using, but instead to give building blocks and sample code covering a wide enough span of complexity and functionality to allow you to implement your own neuron models. If you only need these natively implemented neurons, then great, your job is much much easier :D.

That being said:


## Hodgkin-Huxley Model

The classic four-variable model with sodium, potassium, and leak conductances. Ideal for learning and fast simulations.

### Mathematical Description

**Membrane equation:**
```
C dV/dt = -I_Na - I_K - I_leak + I_ext
```

**Channel currents:**
- `I_Na = g_Na * m³ * h * (V - E_Na)`
- `I_K = g_K * n⁴ * (V - E_K)`  
- `I_leak = g_leak * (V - E_leak)`

### Basic Usage

```julia
# Basic HH neuron
neuron = build_HH(; name=:neuron)

# With custom parameters
using MTKNeuralToolkit.Config
config = HHConfig(Na_g=120.0, K_g=36.0, Leak_g=0.3)
neuron = build_HH(; name=:neuron, config=config)
```

### With Input Stimulus

```julia
# With time-varying input
@named stimulus = TimeVaryingFunction(f=t -> 5*sin(2π*t/20))
neuron = build_HH(stimulus; name=:neuron)
```

### Default Parameters

- **Na_g**: 120.0 mS/cm² (sodium conductance)
- **K_g**: 36.0 mS/cm² (potassium conductance)
- **Leak_g**: 0.3 mS/cm² (leak conductance)
- **Na_E**: 50.0 mV, **K_E**: -77.0 mV, **Leak_E**: -54.4 mV
- **C**: 1.0 μF/cm² (membrane capacitance)

All parameters are modifiable via `HHConfig()`.

## Liu Model

Eight-channel model with detailed calcium dynamics. Suitable for studies requiring calcium-dependent processes.

### Channel Types

- **Na**: Fast sodium (spiking)
- **K**: Delayed rectifier potassium  
- **CaS**: Slow calcium (persistent)
- **CaT**: Transient calcium (low-threshold)
- **KCa**: Calcium-activated potassium
- **H**: Hyperpolarization-activated cation
- **Leak**: Passive leak

### Calcium Dynamics

```
dCa/dt = (1/τ) * (-Ca + Ca∞ + flux_multiplier * I_Ca/C)
```

Calcium concentration affects KCa channels and calcium reversal potentials.

### Usage

```julia
# Liu neuron with calcium dynamics
neuron = build_Liu(; name=:neuron)

# Custom configuration
config = LiuConfig(
    CaS_g=6.0,     # Slow calcium
    CaT_g=2.5,     # Transient calcium
    KCa_g=5.0,     # Ca-activated K
    H_g=0.01       # H-current
)
neuron = build_Liu(; name=:neuron, config=config)
```

**Warning**: Liu models require stiff solvers due to multiple timescales. Use stiff solvers such as TRBDF2() for stability and performance.
## Prinz STG Model

Stomatogastric ganglion neuron model optimized for central pattern generator networks. Similar channel complement to Liu but with different kinetics.

### STG Context

Originally developed for the lobster stomatogastric ganglion, these models excel at producing realistic rhythmic activity in small networks.

**Typical STG neurons:**
- **AB** (Anterior Burster)
- **PD** (Pyloric Dilator) 
- **LP** (Lateral Pyloric)
- **PY** (Pyloric)

### Usage

```julia
# Single Prinz neuron
neuron = build_Prinz(; name=:neuron)

# STG-style burster with input
@named tonic_input = TimeVaryingFunction(f=t -> 1.0)  # Constant drive
ab_neuron = build_Prinz(tonic_input; name=:AB)
```

### STG-Realistic Configuration

```julia
# Neuron with STG-realistic parameters
config = PrinzConfig(
    Na_g=100.0 * 159.2,    # Scaled conductances
    CaS_g=6.0 * 159.2,
    H_g=0.01 * 159.2,
    V0=-60.0               # Resting potential
)
neuron = build_Prinz(; name=:AB, config=config)
```


## Custom Configuration

All neuron types accept configuration structs to modify default parameters:

```julia
# Create custom configuration
custom_config = PrinzConfig(
    Na_g=80.0,           # Reduced sodium
    K_g=40.0,            # Reduced potassium
    CaS_g=10.0,          # Increased slow calcium
    Leak_g=0.05,         # Small leak
    C=2.0,               # Larger capacitance
    V0=-70.0             # Hyperpolarized rest
)

# Apply to neuron
neuron = build_Prinz(; name=:custom, config=custom_config)
```


## Troubleshooting

### Common Issues

- **Simulation fails to converge**: Switch to TRBDF2 solver for Liu/Prinz models
- **Unrealistic spiking behavior**: Check conductance scaling and reversal potentials
- **Calcium concentrations explode**: Verify flux_multiplier and tau parameters in calcium dynamics

---

**Next:** Learn about [synaptic connections](/synapses/) to build multi-neuron networks.