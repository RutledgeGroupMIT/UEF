# USER-UEF
A LAMMPS package for molecular dynamics under diagonal flow fields

<img src="https://github.mit.edu/danich/USER-UEF/blob/master/img/uniaxial_box.gif?raw=true" width=300 />

## Introduction
USER-UEF is a LAMMPS package for non-equilibrium molecular dynamics (NEMD) under diagonal flow fields, including uniaxial and biaxial flows. With this package, simulations under flow may be carried out for an indefinite amount of time, extending the functionality of LAMMPS to include steady-state diagonal flow fields. It is an implementation of the boundary conditions developed by [Matthew Dobson](http://arxiv.org/abs/1408.7078), and also uses numerical lattice reduction as was proposed by [Thomas Hunt](http://arxiv.org/abs/1310.3905). The lattice reduction algorithm used was developed by [Igor Semaev](http://link.springer.com/chapter/10.1007%2F3-540-44670-2_13). The package is intended for simulations of homogeneous flows, and integrates the SLLOD equations of motion. 

## Installation
The implementation has been tested in the Jul. 30, 2016 stable version of LAMMPS. Older versions may not be compatible.

The USER-UEF package is compiled alongside LAMMPS, and doesn't require any additional libraries. Installation will change some of the LAMMPS source files, but these changes will not affect the base functionality of LAMMPS.

To install the package from a LAMMPS distribution located at `lammps/`

1. Clone this repository and copy the `USER-UEF/` subdirectory into the `lammps/src/` directory. 
2. Move into `lammps/src/USER-UEF/` and run the `initialize.sh` script. This script will make necessary changes to source files in LAMMPS. Original source files are saved within `lammps/src/USER-UEF/` with a `.orig` suffix and may be restored with the `revert.sh` script if necessary. 
3. Move into `lammps/src/` then run `make yes-USER-UEF`. 
4. At this point LAMMPS is ready to be compiled. See the LAMMPS [documentation](http://lammps.sandia.gov/doc/Section_start.html#start-2) for compilation instructions.

The following commands will install the package in a fresh version of the current stable LAMMPS package to the current directory.
```
wget http://lammps.sandia.gov/tars/lammps-stable.tar.gz
tar -xvf lammps-stable.tar.gz
git clone [LINK TO PUBLIC REPO HERE]
cp -r USER-UEF/USER-UEF/ lammps-*/src
cd lammps-*/src/
[compile LAMMPS (e.g. make mpi)]
```

## Usage

The package defines `fix nvt/uef` and `fix npt/uef` for constant volume and stress-controlled simulations respectively, `compute pressure/uef` to compute the pressure tensor, and `dump cfg/uef` for outputting coordinates in the reference frame of the flow field.
### fix nvt/uef
* `fix ID all nvt/uef temp Tstart Tstop Tdamp erate eps_x eps_y keyword values ...`
  * ID = name for the fix
  * Tstart,Tstop = external temperature at start/end of run
  * Tdamp = temperature damping parameter (time units)
  * eps_x = strain rate in x dimension 1/(time units) 
  * eps_y = strain rate in y dimension 1/(time units)<br><br>
Additional keywords:
  * strain = initial level of strain (default=0). Use of this keyword is not recommended, but may be recessary when resuming a run with data file. This keyword is not needed when restart files are used.<br>
  * The following additional keywords from [`fix nvt`](http://lammps.sandia.gov/doc/fix_nh.html) can be used with this fix: tchain, tloop, drag<br><br>
Examples: 
  * Uniaxial flow<br>`fix f1 all nvt/uef temp 400 400 100 erate 0.00001 -0.000005`
  * Biaxial flow<br>`fix f2 all nvt/uef temp 400 400 100 erate 0.000005 0.000005`

### fix npt/uef
* `fix ID all npt/uef temp Tstart Tend Tdamp erate eps_x eps_y keyword value...`
  * ID = name for the fix
  * Tstart,Tstop = external temperature at start/end of run
  * Tdamp = temperature damping parameter (time units)
  * Pstart,Pstop = external pressure at start/end of run
  * Pdamp = pressure damping parameter (time units)
  * eps_x = strain rate in x dimension 1/(time units) 
  * eps_y = strain rate in y dimension 1/(time units)<br><br>
Additional keywords: 
  * x or y or z = Pstart Pstop Pdamp
  * iso = Pstart Pstop Pdamp
  * strain = initial level of strain (default=0). Use of this keyword is not recommended, but may be recessary when resuming a run with data file. This keyword is not needed when restart files are used.
  * ext = x or y or z or xy or xz or yz or xyz (default=xyz). These are "external" dimensions used in pressure control. For example, for uniaxial extension in the z direction, x and y correspond to free surfaces. The setting xy will only control (P_xx+P_yy)/2 to the target external pressure.    
  * The following additional keywords from [`fix nvt`](http://lammps.sandia.gov/doc/fix_nh.html) can be used with this fix: couple, tchain, pchain, tloop, ploop, drag<br><br>
Examples: 
  * Uniaxial flow<br>`fix f1 all npt/uef temp 400 400 300 iso 1 1 3000 erate 0.00001 -0.000005 ext yz`
  * Biaxial flow<br>`fix f2 all npt/uef temp 400 400 300 z 1 1 3000 erate 0.000005 0.000005`

### compute pressure/uef
Note: It is generally not necessary to use this command since a `fix nvt/uef` or `fix npt/uef` will contain an instance of this compute. See the note below for details.
* `compute ID all pressure/uef temp-ID`
  * ID = name for the compute
  * temp-ID = ID of compute that calculates temperature<br><br>
Additional keywords: 
  * One of the following additional keywords from [`compute pressure`](http://lammps.sandia.gov/doc/compute_pressure.html) may be used with this fix: ke or pair or bond or angle or dihedral or improper or kspace or fix or virial<br><br>
 Example:
  * `compute c1 all pressure/uef thermo_temp`

#### Usage notes
*  The uef fixes use the SLLOD equations of motion, which lead to an instability in the center of mass velocity. A `fix momentum` should be used to regularly reset the linear momentum.
*  The uef fixes store the peculiar velocities rather than the total velocities, in contrast to the LAMMPS [`fix nvt/sllod`](http://lammps.sandia.gov/doc/fix_nvt_sllod.html?highlight=sllod) command.
*  When the strain keyword is unset, or set to zero, the initial simulation box must be cubic and have style triclinic. If the box is initially of type ortho, use the command `change box all triclinic` before invoking the fix.
* Each fix defines a `compute pressure/uef` and `compute temp` that can be be accessed at `c_ID_press` and `c_ID_temp` respectively for scalar values, or `c_ID_press[i]` and `c_ID_temp[i]` for the pressure and kinetic energy tensors. The symmetric tensors are stored as vectors, and the values of i from 1 to 6 correspond to components xx, yy, zz, xy, xz, yz. This is the recommended method for obtaining the pressure tensor. The scalar value for the pressure will correspond to the average stress only in the directions indicated by the ext keyword if it is set. The kinetic energy tensor will not correspond to the coordinate system of the flow field.
* The uef fixes can be used with `write_restart` and `read_restart`. 
* The uef fixes are compatible with `run_style respa`
* The uef fixes are compatible with `fix modify`, however custom pressure computes must be of type `pressure/uef`. 
* Any trajectories written when one of the above fixes is used will not be in the same coordinate system as the flow field unless `dump cfg/uef` is used.
* Only the atomic positions will be in the reference frame of the flow field when `dump cfg/uef` is used. If the atomic velocities are specified as an output, for example, their values will correspond a coordinate system with an upper triangular simulation box.
* Vector or tensor-valued quantities computed, except for the pressure tensor from `compute pressure/uef`, will not be in the same coordinate system as the flow field. 

## Implementation Details

The simulation box used in the boundary conditions developed by Hunt and Dobson does not have a consistent alignment relative to the applied flow field. This can be seen in the video of the simulation box under uniaxial extension at the top of the page. LAMMPS utilizes an upper-triangular simulation box, making it impossible to express the evolving simulation box in the same coordinate system as the flow field. The USER-UEF package keeps track of two coordinate systems: the flow frame, and the upper triangular LAMMPS frame. The coordinate systems are related to each other through the QR decomposition, as is illustrated in the image below.

<img src="https://github.mit.edu/danich/USER-UEF/blob/master/img/frames.jpg?raw=true" width=300 />

During most molecular dynamics operations, the system is represented in the LAMMPS frame. Only when the positions and velocities are updated is the system rotated to the flow frame, and it is rotated back to the LAMMPS frame immediately afterwards. For this reason, all vector-valued quantities (except for the pressure tensor from `compute pressure/uef`) will be computed in the LAMMPS frame. Rotationally invariant scalar quantites like the temperature and hydrostatic pressure, on the other hand, will be computed correctly. Additionally, the system is in the LAMMPS frame during all of the output steps, and therefore trajectory files made using the `dump` command will be in the LAMMPS frame. 

## Examples

Sample input scripts and data files for uniaxial and biaxial extension of a WCA fluid can be found in the `examples/` subdirectory. Please see `examples/README` for further information.

## Error and Warning Messages

The methods described with inherit error/warning messages from `fix npt/nvt`, `compute pressure`, and `dump cfg`. Additional error messages associated with this package are below:

* "Illegal fix nvt/npt/uef command" - The fix was not called correctly. Check the usage notes.
* "Keyword erate must be set for fix nvt/npt/uef command" - Self-explanatory.
* "Simulation box must be triclinic for fix/nvt/npt/uef" - Self-explanatory.
* "Only normal stresses can be controlled with fix/nvt/npt/uef" - The keywords xy xz and yz cannot be used for pressure control.
* "All controlled stresses must have the same value in fix/nvt/npt/uef" - Stress control is only possible when the stress specified for each dimension is the same 
* "Dimensions with controlled stresses must have same strain rate in fix/nvt/npt/uef" - Stress-controlled dimensions with the same strain rate must have the same target stress.
* "Can't use another fix which changes box shape with fix/nvt/npt/uef" - The `fix npt/nvt/uef` command must have full control over the box shape. You cannot use a simultaneous `fix deform` command, for example.
* "Pressure ID for fix/nvt/uef doesn't exist" - The `compute pressure` introduced via `fix_modify` does not exist
* "Using fix nvt/npt/uef without a compute pressure/uef" - The `compute pressure` introduced via `fix_modify` is not a `compute pressure/uef`. The computed pressure tensor will not be in the coordinate system of the flow field and the ext keyword will not take effect.
* "Initial box is not close enough to the expected uef box" - The initial box does not correspond to the shape required by the value of the strain keyword. If the default strain value of zero was used, the initial box is not cubic.
* "Can't use compute pressure/uef without defining a fix nvt/npt/uef" - Self-explanatory.
* "Can't use dump cfg/uef without defining a fix nvt/npt/uef" - Self-explanatory.
* "Temperature control must be used with fix nvt/uef" - Self-explanatory.
* "Pressure control can't be used with fix nvt/uef" - Self-explanatory.
* "Temperature control must be used with fix npt/uef" - Self-explanatory.
* "Pressure control must be used with fix npt/uef" - Self-explanatory.

## TODO
*  Define new computes that will allow the user to create `dump custom` commands in the flow frame.
*  Extend functionality to include all diagonalizeable flow fields
