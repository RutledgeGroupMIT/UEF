# UEF
A LAMMPS package for molecular dynamics under extensional flow fields

<img src="https://github.com/danicholson/UEF/blob/master/img/uniaxial_box.gif?raw=true" width=300 />

UEF is a LAMMPS package for non-equilibrium molecular dynamics (NEMD) under diagonal flow fields, including uniaxial and biaxial flows. With this package, simulations under flow may be carried out for an indefinite amount of time, extending the functionality of LAMMPS to include steady-state diagonal flow fields. It is an implementation of the boundary conditions developed by [Matthew Dobson](http://arxiv.org/abs/1408.7078), and also uses numerical lattice reduction as was proposed by [Thomas Hunt](http://arxiv.org/abs/1310.3905). The lattice reduction algorithm used was developed by [Igor Semaev](http://link.springer.com/chapter/10.1007%2F3-540-44670-2_13). The package is intended for simulations of homogeneous flows, and integrates the SLLOD equations of motion. 

Authored by:
[David Nicholson](https://github.com/danicholson)<br>
Massachusetts Institute of Technology<br>
Initial commit: Oct 19, 2016<br><br>
Support provided via [issues](https://github.com/danicholson/UEF/issues) and/or [email](mailto:davidanich@gmail.com).

## Contents
* [Installation](#installation)
* [Usage](#usage)
 * [fix nvt/uef](#fix-nvtuef)
 * [fix npt/uef](#fix-nptuef)
 * [compute temp/uef](#compute-tempuef)
 * [compute pressure/uef](#compute-pressureuef)
 * [dump cfg/uef](#dump-cfguef)
* [Implementation Details](#implementation-details)
* [Examples](#examples)
* [Error and Warning Messages](#error-and-warning-messages)
* [Citing the UEF package](#citing-the-uef-package)


## Installation
The implementation has been tested in the Jul. 30, 2016 stable version of LAMMPS. Older versions may not be compatible.

The UEF package is compiled within LAMMPS, and doesn't require any additional libraries or modify the LAMMPS source code.

To install the package from a LAMMPS distribution located at `lammps/`

1. Clone this repository and copy the `USER-UEF/` subdirectory into the `lammps/src/` directory. 
2. Move into `lammps/src/` then run `make yes-USER-UEF`. 
3. Compile LAMMPS. See the LAMMPS [documentation](http://lammps.sandia.gov/doc/Section_start.html#start-2) for compilation instructions.

The following commands will install the package in a fresh version of the current stable LAMMPS package to the current directory.
```
wget http://lammps.sandia.gov/tars/lammps-stable.tar.gz
tar -xvf lammps-stable.tar.gz
git clone https://github.com/danicholson/UEF.git
cp -r UEF/USER-UEF/ lammps-*/src
cd lammps-*/src/
make yes-user-uef
[compile LAMMPS (e.g. make mpi)]
```

## Usage

The package defines `fix nvt/uef` and `fix npt/uef` for constant volume and stress-controlled simulations, `compute pressure/uef` and `compute temp/uef` to compute the pressure and kinetic energy tensors, and `dump cfg/uef` for outputting properly oriented atomic coordinates.

***

### fix nvt/uef
#### Syntax
`fix ID all nvt/uef temp Tstart Tstop Tdamp erate eps_x eps_y keyword values ...`
* ID = name for the fix
* Tstart,Tstop = external temperature at start/end of run
* Tdamp = temperature damping parameter (time units)
* eps_x = strain rate in x dimension 1/(time units) 
* eps_y = strain rate in y dimension 1/(time units)<br><br>Additional keywords: 
* strain = initial level of strain (default="0 0"). Use of this keyword is not recommended, but may be necessary when resuming a run with data file. This keyword is not needed when restart files are used.<br>
* The following additional keywords from [`fix nvt`](http://lammps.sandia.gov/doc/fix_nh.html) can be used with this fix: tchain, tloop, drag
 
#### Examples
 * Uniaxial flow<br>`fix f1 all nvt/uef temp 400 400 100 erate 0.00001 -0.000005`
 * Biaxial flow<br>`fix f2 all nvt/uef temp 400 400 100 erate 0.000005 0.000005`

#### Usage notes
Due to requirements of the boundary conditions, when the strain keyword is unset, or set to zero, the initial simulation box must be cubic and have style triclinic. If the box is initially of type ortho, use the command `change box all triclinic` before invoking the fix.

This fix integrates the SLLOD equations of motion, which lead to an instability in the center of mass velocity under extension. A [`fix momentum`](http://lammps.sandia.gov/doc/fix_momentum.html) should be used to regularly reset the linear momentum. Additionally, this fix stores the peculiar velocity of each atom, defined as the velocity relative to the streaming velocity. This is in contrast to the LAMMPS [`fix nvt/sllod`](http://lammps.sandia.gov/doc/fix_nvt_sllod.html?highlight=sllod) command.

This fix defines a `compute pressure/uef` and `compute temp/uef` that can be accessed at `c_ID_press` and `c_ID_temp` respectively for scalar values, or `c_ID_press[i]` and `c_ID_temp[i]` for the pressure and kinetic energy tensors.

When this fix is applied, any orientation-dependent vector or tensor-valued quantities computed, except for the tensors from `compute pressure/uef`/`compute temp/uef` and coordinates from `dump cfg/uef`, will not be in the same coordinate system as the flow field. See the [implementation details](#implementation-details) for further information.

This fix can be used with `write_restart` and `read_restart`, `run_style respa`, and `fix modify`, however custom pressure and temperature computes must be of type `pressure/uef` and `temp/uef`. When resuming from restart files, you may need to use `box tilt large` since LAMMPS does not always agree that the simulation box is fully reduced.

***

### fix npt/uef
#### Syntax
 `fix ID all npt/uef temp Tstart Tend Tdamp erate eps_x eps_y keyword value...`
* ID = name for the fix
* Tstart,Tstop = external temperature at start/end of run
* Tdamp = temperature damping parameter (time units)
* Pstart,Pstop = external pressure at start/end of run
* Pdamp = pressure damping parameter (time units)
* eps_x = strain rate in x dimension 1/(time units) 
* eps_y = strain rate in y dimension 1/(time units)<br><br>Additional keywords: 
* x or y or z = Pstart Pstop Pdamp
* iso = Pstart Pstop Pdamp
* strain = initial level of strain (default=0). Use of this keyword is not recommended, but may be necessary when resuming a run with data file. This keyword is not needed when restart files are used.
* ext = x or y or z or xy or xz or yz or xyz (default=xyz). These are "external" dimensions used in pressure control. For example, for uniaxial extension in the z direction, x and y correspond to free surfaces. The setting xy will only control (P_xx+P_yy)/2 to the target external pressure.    
* The following additional keywords from [`fix nvt`](http://lammps.sandia.gov/doc/fix_nh.html) can be used with this fix: couple, tchain, pchain, tloop, ploop, drag<br><br>
  
#### Examples 
* Uniaxial flow<br>`fix f1 all npt/uef temp 400 400 300 iso 1 1 3000 erate 0.00001 -0.000005 ext yz`
* Biaxial flow<br>`fix f2 all npt/uef temp 400 400 300 z 1 1 3000 erate 0.000005 0.000005`

#### Usage notes
The usage notes for `fix nvt/uef` apply to `fix npt/uef` as well. There are two ways to control the pressure using this fix. The first method involves using the`ext` keyword along with the `iso` pressure style. With this method, the pressure is controlled by scaling the simulation box isotropically to achieve the average stress in the directions specified by `ext`. 

For example, this command will control the hydrostatic pressure under uniaxial tension:

`fix f1 all npt/uef temp 0.7 0.7 0.5 iso 1 1 5 erate -0.5 -0.5 ext xyz`

This command will control the average stress in compression directions under uniaxial tension:

`fix f1 all npt/uef temp 0.7 0.7 0.5 iso 1 1 5 erate -0.5 -0.5 ext xy`

The second method involves setting the normal stresses using the `x`, `y` , and/or `z` keywords. When using this method, the same pressure must be specified via `Pstart` and `Pstop` for all dimensions controlled. Any choice of pressure conditions that would cause LAMMPS to compute a deviatoric stress are not permissible and will result in an error. Additionally, all dimensions with controlled stress must have the same applied strain rate. The `ext` keyword must be set to the default value (`xyz`) when using this method. 

For example, the following commands will work:
```
fix f1 all npt/uef temp 0.7 0.7 0.5 x 1 1 5 y 1 1 5 erate -0.5 -0.5
fix f1 all npt/uef temp 0.7 0.7 0.5 z 1 1 5 erate 0.5 0.5
```

The following commands will not work:
```
fix f1 all npt/uef temp 0.7 0.7 0.5 x 1 1 5 z 1 1 5 erate -0.5 -0.5
fix f1 all npt/uef temp 0.7 0.7 0.5 x 1 1 5 z 2 2 5 erate 0.5 0.5
```

***

### compute temp/uef
#### Syntax
`compute ID all temp/uef`
* ID = name for the compute
  
#### Examples
`compute c1 all temp/uef`

#### Usage notes
This compute requires a `fix nvt/uef` or `fix npt/uef`. It computes the kinetic energy tensor in the reference frame of the flow field.

See the documentation for [`compute temp`](http://lammps.sandia.gov/doc/compute_pressure.html) for further details on output.

***

### compute pressure/uef
#### Syntax
`compute ID all pressure/uef temp-ID`
* ID = name for the compute
* temp-ID = ID of compute that calculates temperature<br><br>Additional keywords: 
* The following additional keywords from [`compute pressure`](http://lammps.sandia.gov/doc/compute_pressure.html) may be used with this fix: ke or pair or bond or angle or dihedral or improper or kspace or fix or virial
 
#### Examples
`compute c1 all pressure/uef c_1_temp`

#### Usage notes
This compute requires a`fix nvt/uef` or `fix npt/uef`. It computes the pressure tensor in the reference frame of the flow field.

The pressure tensor computed from `compute pressure/uef` is only accurate if its temperature compute, specified by `temp-ID`, is a `compute temp/uef`. 

See the documentation for [`compute pressure`](http://lammps.sandia.gov/doc/compute_pressure.html) for further details on output.

***

### dump cfg/uef
#### Syntax
`dump ID all cfg/uef N file mass type xs ys zs keyword value`
* N =  dump every this many timesteps
* file = name of file to write dump info to<br><br>Additional keywords: 
* See the documentation for [`dump cfg`](http://lammps.sandia.gov/doc/dump.html) for additional keywords.
  
#### Examples
`dump d1 all cfg/uef 100 dump.*.cfg mass type xs ys zs`

#### Usage notes
This command requires a `fix nvt/uef` or `fix npt/uef`. It outputs the atomic positions in the reference frame of the flow field. Only the positions are in the proper reference frame; if the atomic velocities are specified as an output, for example, they will not be in the flow field reference frame.

See the [`dump cfg`](http://lammps.sandia.gov/doc/dump.html) documentation for further information on writing trajectories with cfg files.

***

## Implementation Details

The simulation box used in the boundary conditions developed by Hunt and Dobson does not have a consistent alignment relative to the applied flow field. This can be seen in the video of the simulation box under uniaxial extension at the top of the page. LAMMPS utilizes an upper-triangular simulation box, making it impossible to express the evolving simulation box in the same coordinate system as the flow field. The UEF package keeps track of two coordinate systems: the flow frame, and the upper triangular LAMMPS frame. The coordinate systems are related to each other through the QR decomposition, as is illustrated in the image below.

<img src="https://github.com/danicholson/UEF/blob/master/img/frames.jpg?raw=true" width=300 />

During most molecular dynamics operations, the system is represented in the LAMMPS frame. Only when the positions and velocities are updated is the system rotated to the flow frame, and it is rotated back to the LAMMPS frame immediately afterwards. For this reason, all vector-valued quantities (except for the tensors from `compute pressure/uef` and `compute temp/uef`) will be computed in the LAMMPS frame. Rotationally invariant scalar quantities like the temperature and hydrostatic pressure, on the other hand, will be computed correctly. Additionally, the system is in the LAMMPS frame during all of the output steps, and therefore trajectory files made using the `dump` command will be in the LAMMPS frame unless the `dump cfg/uef` command is used. 

## Examples

Sample input scripts and data files for uniaxial and biaxial extension of a WCA fluid can be found in the `examples/` subdirectory. Please see `examples/README` for further information.

## Error and Warning Messages

The methods described with inherit error/warning messages from `fix npt/nvt`, `compute pressure`, and `dump cfg`. Additional error messages associated with this package are below:

* "Illegal fix nvt/npt/uef command" - The fix was not called correctly. Check the usage notes.
* "Keyword erate must be set for fix nvt/npt/uef command" - Self-explanatory.
* "Simulation box must be triclinic for fix/nvt/npt/uef" - Self-explanatory.
* "Only normal stresses can be controlled with fix/nvt/npt/uef" - The keywords xy xz and yz cannot be used for pressure control.
* "The ext keyword may only be used with iso pressure control" - Self-explanatory
* "All controlled stresses must have the same value in fix/nvt/npt/uef" - Stress control is only possible when the stress specified for each dimension is the same 
* "Dimensions with controlled stresses must have same strain rate in fix/nvt/npt/uef" - Stress-controlled dimensions with the same strain rate must have the same target stress.
* "Can't use another fix which changes box shape with fix/nvt/npt/uef" - The `fix npt/nvt/uef` command must have full control over the box shape. You cannot use a simultaneous `fix deform` command, for example.
* "Pressure ID for fix/nvt/uef doesn't exist" - The `compute pressure` introduced via `fix_modify` does not exist
* "Using fix nvt/npt/uef without a compute pressure/uef" - The `compute pressure` introduced via `fix_modify` is not a `compute pressure/uef`.
* "Initial box is not close enough to the expected uef box" - The initial box does not correspond to the shape required by the value of the strain keyword. If the default strain value of zero was used, the initial box is not cubic.
* "Can't use compute pressure/uef without defining a fix nvt/npt/uef" - Self-explanatory.
* "Can't use compute pressure/uef without defining a fix nvt/npt/uef" - Self-explanatory.
* "Can't use compute/temp without defining a fix nvt/npt/uef" - Self-explanatory.
* "Temperature control must be used with fix nvt/uef" - Self-explanatory.
* "Pressure control can't be used with fix nvt/uef" - Self-explanatory.
* "Temperature control must be used with fix npt/uef" - Self-explanatory.
* "Pressure control must be used with fix npt/uef" - Self-explanatory.

## Citing the UEF package

Coming soon. Please check back for the relevant citation.
