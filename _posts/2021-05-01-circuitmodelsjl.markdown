---
layout: post
title: CircuitModels.jl
category: development
---


# CircuitModels.jl
Circuit modeling and DAE setup in Julia. This code is largely based on the MAPP prototyping platform and the MNA equation engine written by Prof. Jaijeet Roychowdhury's group.
This package was originally written as a final project for EECS 219A at UC Berkeley in Spring 2021.

## Code Structure Summary
```
examples
 ┣ circuits                 - examples of how to build a circuit
 ┃ ┣ BJTDiffPair.jl
 ┃ ┣ NOTGate.jl
 ┃ ┣ RCLine.jl
 ┃ ┣ nStageRingOscillator.jl
 ┃ ┗ vsrc.jl
 ┣ mna                      - examples of how to make an MNA object from a circuit (each one line, structured for extensibility)
 ┃ ┣ BJTDiffPair.jl
 ┃ ┣ NOTGate.jl
 ┃ ┣ RCLine.jl
 ┃ ┣ nStageRingOscillator.jl
 ┃ ┗ vsrc.jl
 ┗ netlists                 - netlists to parse
 ┃ ┣ diffpair.clr
 ┃ ┣ mosfet_char.clr
 ┃ ┗ rc.clr

 src                            
 ┣ circuitsystems           - circuit building and modeling functionality 
 ┃ ┣ circuit.jl             - functions to build a circuit
 ┃ ┣ circuitsystems.jl      - submodule to include the other two files and expose them to the rest of the package
 ┃ ┗ mnasystem.jl           - functionality to do MNA from a circuit and run f/q
 ┣ devices                  - definitions of individual devices
 ┃ ┣ bjt.jl
 ┃ ┣ capacitor.jl
 ┃ ┣ devices.jl             - submodule to include all the device models and expose them to the rest of the package
 ┃ ┣ diode.jl
 ┃ ┣ inductor.jl
 ┃ ┣ mosfet.jl
 ┃ ┣ resistor.jl
 ┃ ┣ sources.jl
 ┃ ┣ switch.jl
 ┃ ┗ utils.jl               - smoothif and unit checking
 ┣ parser                   - netlist parsing functionality
 ┃ ┣ parse_devices.jl       - functions to parse individual devices from a line of text in the netlist
 ┃ ┣ parser.jl              - core parsing
 ┃ ┗ utils.jl               - order of magnitude and device object lookups
 ┣ systems
 ┃ ┣ analysis.jl            - build DAEs from Systems
 ┃ ┗ systems.jl             - include submodules that depend on the `System` object
 ┗ CircuitModels.jl         - include all submodules
 
 test
 ┣ runtests.jl              - runs the other three files
 ┣ test_circuit_builds.jl   - checks that all the circuits in "examples" can be conveted to MNA + f/q are callable
 ┣ test_devices.jl          - checks device constructors, unit validity, and that fe/qe are callable
 ┗ test_parser.jl           - checks that the netlists in "examples/netlists" can be parsed to circuits
.gitignore
LICENSE                     - MIT License, could be changed for public release
Manifest.toml               - Julia-generated file listing all dependencies (machine-generated)
Project.toml                - Julia-generated file listing direct dependencies (made via Julia REPL as packages are added)
README.md                   - this file
```

## Running the Code

This package requires Julia 1.6+, which can be downloaded from [https://julialang.org/downloads/].

Clone and `cd` into this repository, and run `julia` from the command line at the top level (the current working directory should be `CircuitModels.jl`.) Then, run `]activate .` (the `]` will become `@v1.6) pkg>`), then backspace so that the REPL reads `julia>` again and type `using CircuitModels`. This will precompile the package and all of its dependencies, which may take a while the first time it's run, although there should be a loading bar tracking the compiler's progress. It should look like this:

```
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.6.0 (2021-03-24)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

(@v1.6) pkg> activate .
  Activating environment at `~/projects/CircuitModels.jl/Project.toml`

julia> using CircuitModels
[ Info: Precompiling CircuitModels [cc1e6237-006d-4ed7-97b2-b8635ee6724d]

julia> 
```

(Note that this isn't the same procedure that's used if the package in question is public; in that case, it's possible to install a package by running `]add <package_name_or_link>` either from the name registered with the Julia package repository, or the GitHub link, similar to `pip` installation in Python.)

To run all the tests in the `test/` subdirectory, run `include("test/runtests.jl")`, or any of the individual files as desired. This checks that all the devices, example circuits, and example netlists are successfully constructed. (Upon running these, optionally try running a second time and observe the speedup due to JIT!)

In order to load the constructed circuits in `examples/circuits/`, run `include("examples/circuits/BJTDiffPair.jl")` (or any of the other files provided). This will load a `Circuit` object named `BJTDiffPair_circuit` (or `NOTGate_circuit`, etc.).

```
julia> include("examples/circuits/BJTDiffPair.jl")

julia> BJTDiffPair_circuit
Circuit(BJT differential pair, [:gnd, :nBL, :nE, :nCL, :VDD, :nCR])
```

Modified nodal analysis is then carried out by calling `MNASystem`, e.g.

```
julia> BJTDiffPair = MNASystem(BJTDiffPair_circuit)
Modified nodal analysis engine for BJT differential pair
```

Note that the standard MATLAB/IPython rules for suppressing output with semicolons applies here too.
The MNA objects can also be obtained by directly running `include("examples/mna/BJTDiffPair.jl")`, or similar for the others.

The MNA objects include the underlying circuit object (`BJTDiffPair.circuit`), information about the state variables (`BJTDiffPair.statenames`), the unknowns (`BJTDiffPair.unames_array`), and the equations being solved (`BJTDiffPair.eqn_names`).

```
julia> BJTDiffPair.circuit
Circuit(BJT differential pair, [:gnd, :nBL, :nE, :nCL, :VDD, :nCR])

julia> BJTDiffPair.statenames
6-element Vector{String}:
 "v_nBL"
 "v_nE"
 "v_nCL"
 "v_VDD"
 "v_nCR"
 "i_Vin@p"

julia> BJTDiffPair.unames_array
2-element Vector{String}:
 "Vin::V"
 "IE::i"

julia> BJTDiffPair.eqn_names
6-element Vector{String}:
 "KCL@nBL"
 "KCL@nE"
 "KCL@nCL"
 "KCL@VDD"
 "KCL@nCR"
 "fix V_Vin@1"
```

The functions `f` and `q` can then be called on the MNA objects, by running `f(BJTDiffPair, x, u)`. However, since the MNA equation engine is unit-aware, passing in vectors of unitless numbers will result in a unit error. To check the correct units for `x`, call `state_units(BJTDiffPair)`, and similarly for `u` call `input_units(BJTDiffPair)`. As is done in `test/test_circuit_builds.jl`, it is possible to construct a unit-aware vector from a vector of unitless numbers using the `.*` operator (multiplication broadcast over arrays), e.g.

```
julia> x = rand(6) .* state_units(BJTDiffPair)
6-element Vector{Quantity{Float64, D, U} where {D, U}}:
  0.8615763846864712 V
 0.04094620625121448 V
  0.4266211206166659 V
  0.7557126732061681 V
  0.9195226664448348 V
 0.04112044475339349 A
```
Further, the output of `f` should have the units in `output_units(BJTDiffPair)`, and the output of `q` should have those units multiplied by a unit of time.

```
julia> u = rand(2) .* input_units(BJTDiffPair)
2-element Vector{Quantity{Float64, D, U} where {D, U}}:
 0.7670477902394872 V
 0.3016086826622677 A

julia> f(BJTDiffPair, x, u)
6-element Vector{Quantity{Float64, D, U} where {D, U}}:
   0.5510543964231576 A
   -51.29411038942376 A
   50.482403209317724 A
 8.264077967541773e-5 A
 8.190499483430076e-5 A
 -0.09452859444698403 V

julia> q(BJTDiffPair, x, u)
6-element Vector{Quantity{Float64, D, U} where {D, U}}:
  1.2555854425050623e-9 A s
 -7.796839721840424e-10 A s
   -3.29526507853572e-7 A s
  1.6528155935083543e-7 A s
  1.6472951590511157e-7 A s
                    0.0 s V
```

The provided netlists can be parsed by calling `parse_netlist` with the filename, for example,
```
julia> parse_netlist("examples/netlists/rc.cir")
Circuit(RC circuit, [:n0, :n1, :n2])
```

Finally, this functionality is not fully developed yet, but it should be possible to call `dae_objective`, `make_dae_problem`, and `dcop` with an `MNASystem` passed in as the `System` object.

## Limitations

Problems and potential enhancements have mostly been logged in this repository under the GitHub Issues tracker and explained in more detail in the project report. In particular:

- Netlist parsing has some limitations in handling advanced parameters
- Some devices have not been implemented
- The controlled source implementations have not been tested
- Tests of MNA on netlist-parsed circuits (rather than the explicitly-defined circuits in `examples/circuits`) have not been carried out

While the last one is likely not an issue (the netlist-parsed VSRC is exactly the same as the user-defined one) it may be a source of additional errors when more advanced netlist parsing is added. 
