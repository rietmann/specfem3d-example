# Notes on running specfem3d "cartesian" on practical example #

This document will walk you through building and running examples with SPECFEM3D Cartesian (specfem3d).

## Building specfem3d ##

First we get specfem3d from its repository

    git clone --recursive https://github.com/geodynamics/specfem3d.git
    cd specfem3d

Specfem3d uses the autotools/configure/make build environment, which
I'll assume you are comfortable enough to make adjustments to. I'll assume you're mpif90 is already equipped with ifort (with compiled mpi module i.e., `use mpi` works).

    ./configure FC=ifort
    make

This step can be error prone, and the version I pulled over two years ago for my thesis work is now hopelessly out of date, so you will quickly know as much as I do about building specfem3d.

You should now have the following executables:

    bin/xdecompose_mesh
    bin/xgenerate_databases
    bin/xspecfem3D
    bin/xmeshfem3D

Info about the executables

* `xdecompose_mesh` loads a finite element mesh and partitions it into
  the user-supplied number of parts. Run serially (`./xdecompose_mesh
  <NPROC> ../MESH_LOCATION/ ../OUTPUT_FILES/DATABASES_MPI/`)
* `xgenerate_databases` takes the partitioned mesh and prepares it for
  a simulation. Run in parallel (`aprun -n <NPROC>
  ./xgenerate_databasese`)
* `xspecfem3d` runs the simulation.
* `xmeshfem3D` is a mostly defunct mesh generator that can be useful
  for weak-scaling tests as it can generate arbitrarily large
  meshes. `xdecompose_mesh` has some practical limitations for very
  large meshes on large numbers of processors. See the user guide (in
  `doc/` folder).

## Running an example ##

Lets assume you have a directory `CH_example`, containing the example meshes `MESH_CH_{7K,45K,360K,2_9M}`. I have included a range of sizes assuming you get far enough to do more serious performance evaluations on a multinode cluster (2.9M elements is reasonably large and would run well on 1024 cores using MPI only).

    cd CH_example
    cp -r specfem3d/bin .
    mkdir OUTPUT_FILES
    mkdir OUTPUT_FILES/DATABASES_MPI
    cp -r DATA_LOC/DATA .

The simulation is configured with:

* The file `DATA/Par_file` contains all runtime configuration
  parameters (e.g., number of processors `NPROC`, time-step size `DT`,
  and GPU capabilities `GPU_MODE`). You should only need to change the
  `NPROC` setting if you run on more than 1 CPU core.
* The file `Data/CMTSOLUTION` provides the "earthquake" source. It
  just needs to be within the domain.
* The file `STATIONS` provides the recording station names and
  locations, and you might like to add additional ones for your
  testing.

Assuming a single processor (NPROC=1), we run the proprocessing stage:
    
    ./bin/xdecompose_mesh 1 ../MESH_CH_45K ../OUTPUT_FILES/DATABASES_MPI
    mpirun -np 1 ./bin/xgenerate_databases
    
This creates a file `OUTPUT_FILES/output_mesher.txt`, which contains the line

    *** Maximum suggested time step =   0.308303535

A given mesh has a given minimum ratio (element size / pressure velocity), which requires you to adapt the time step `DT` in the `Par_file`. Specfem3d recommends a time-step size based on a heuristic, and for this example, set `DT = 0.3`.

The simulation is now prepared, and its length is adjusted by the parameter `NSTEP` in the `Par_file`. Logically, `Total time = (NSTEP)x(DT)`, so to get more simulated time, increase `NSTEP`.

Running a simulation is then

    mpirun -np 1 ./bin/xspecfem3d

It generates a log in `OUTPUT_FILES/output_solver.txt`. It also creates ascii-based "seismograms" in OUTPUT_FILES, which can easily be loaded into python (with NumPy support)

    from pylab import *
    s = loadtxt('/scratch/example_alex_heinecke/OUTPUT_FILES/CH.SENIN.MXZ.semd')
    plot(s[:,0],s[:,1])
    show()

## Brief Computational walkthrough ##

I'll assume a familiarity with the computational structure of the code, i.e., the time-stepping loop and the basics of the finite-element method used. If not, see my thesis for a more detailed overview ("Local Time Stepping on High Performance Computing Architectures", Universita della Svizzera italiana, Rietmann 2015).

The time-stepping logic is within the file: `iterate_time.F90`. As you might expect, there is a lot of I/O handling, but the stiffness-matrix computation is within the call

    ! 1. elastic domain w/ adjoint wavefields
    call compute_forces_viscoelastic()

This goes through the file `compute_forces_viscoelastic_calling_routine.F90`. Here the program chooses between the optimized routine `USE_DEVILLE_PRODUCTS` (set in `setup/constants.h`). The `compute_forces_viscoelastic_noDev()` routine may be a good place to start understanding the code, as it is quite a bit simpler than the _Dev routines. However, there has been some refactoring in the optimized routines setting them as small matrix-matrix routines, which I imagine is where you wish to try things. See `compute_forces_viscoelastic_Dev.F90`.

Good Luck!

