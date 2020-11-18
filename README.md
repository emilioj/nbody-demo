CODE and original text by Fabio Baruffa
https://github.com/fbaru-dev/nbody-demo

Adapted text by me, Emilio J. Padrón.

This is an example code based on a simple N-body simulation of a
distribution of point masses placed at location r_1,...,r_N and have
masses m_1,...,m_N. The position of the particles after a specified
time is computed using a finite difference methods for ordinary
differential equation.

## Implementation

For each particle the position, the velocity, the acceleration and the
mass is stored in a C-like structure and for an N particles case, an
array of this structure is allocated. This is the simple
data-structure which is very close to the physical representation of a
particle mass.

The file `Particle.hpp` contains the implementation of such
data-structure.

For each particle indexed by i, the accelearation is computed a_i =
G*mj*(ri-rj)/|ri-rj|^3, which value is used to update the velocity and
position using the Euler integration scheme.  Furthermore the total
energy of the particles' group is computed.

The file `GSimulation.cpp` contains the implementation of the
algorithm.

## Directory structure of the Demo

The demo consists of several directories, which correspond to the
different optimization steps to take to enabling vectorization and
OpenMP multi-threding of the code.

Each directory has its onw makefile to compile and run the test case.
To compiler the code type `make` and the run the simulation type `make
run`.

As benhmark, the simulation starts with 16000 particles and 10
integration steps. One can change the default giving the number of
particles and the number of integration steps using the command line
argument: `./nbody.x < # of particles> < # of integration>`

Try to change the number of particles and observe how the performance
changes.

## Different versions
To start the demo, go to the folder `ver0`, compile and run the test.

### Intial version: ver0
The typical output of the simulation is:
```
Run the default test case on CPU:
./nbody.x
===============================
 Initialize Gravity Simulation
 nPart = 16000; nSteps = 10; dt = 0.1
------------------------------------------------
 s       dt      kenergy     time (s)    GFlops
------------------------------------------------
 1       0.1     26.405      1.7966      4.1324
 2       0.2     313.77      1.5309      4.8498
 3       0.3     926.56      1.5311      4.8489
 4       0.4     1866.4      1.5313      4.8484
 5       0.5     3135.6      1.5315      4.8479
 6       0.6     4737.6      1.5309      4.8497
 7       0.7     6676.6      1.5312      4.8487
 8       0.8     8957.7      1.5311      4.849
 9       0.9     11587       1.5314      4.848
 10      1       14572       1.5309      4.8495

# Number Threads     : 1
# Total Time (s)     : 15.577
# Average Perfomance : 4.8488 +- 0.00062286
===============================

```

On output is printed some useful information. Colomnwise: s is the
number of steps; dt is the physical time taking into account the
physical time integration step; kenery is the kinetic energy of the
group of particles; time is the computational time taken till that
time step; GFlops is the number of giga flops per second.

N.B. The GFlops is an estimation done by looking into the code and
counting the number of math operations according to the
algorithm. This is used only as standard metric for comparison. More
realistic numbers can be measured in different way (Roofline model of
Intel® Advisor).

Following the five steps of code modernization,
https://software.intel.com/en-us/articles/what-is-code-modernization
we can improve the performance of the code.

- describe the Intel® Advisor result (other session, *not today*)
- generate the compiler report and describe the different options:
-  -qopt-report[=N]: default level is 2
-  -qopt-report-phase=<vec,loop,openmp,...>: default is all
-  -qopt-report-file=stdout | stderr | filename
-  -qopt-report-filter="GSimulation.cpp,130-204"

Then show how verbose is the compiler report and use filtering.

Try it also with GCC using Makefile_gnu:
```
make -f Makefile_gnu
```

In the different versions, try to compare to a gcc version!

### ver1
Solution of the ver0. The optimization are: -O2 -xAVX or higher.
The Makefile is the only difference. Here we generate higher vectorized code and
produce the compiler report.
One should run this version in the same way as before and:
- show the new performance numbers
- generate the compiler report and have a look
- compare with GCC
- disable vectorization in ICC with -no-vec and compare again!

### ver2
Solution of the ver1. The difference is in the GSimulation.cpp file where the consistent
computation with floats is made (constants and SQRT function).
One should run this version in the same way as before and:
- show the new performance numbers
- generate the compiler report and look at the auto-vectorization stuff

### ver3
Solution of the ver2. The differences are in:
- Particle.hpp: the new SoA data structure is implemented
- GSimulation.hpp: modified the data member according to SoA
- GSimulation.cpp: allocation and reference to SoA

One should run this version in the same way as before and:
- show the new performance numbers
- generate the compiler report and look at some of the vectorization remarks
- modify in the `Makefile` the CXXFLAGS adding the OMPFLAGS at line 8, recompile and run
  remark #15301: OpenMP SIMD LOOP WAS VECTORIZED
- at this point running the code shows wrong results (Warning with SIMD, be aware of the full control)
- try to use #pragma simd reduction (solution in the file GSimulation-simd.cpp)
  NB rember that the simd reduction is not allowed on `particles->acc_x[i]`
  Solution:
    - cp GSimulation.cpp GSimulation.cpp.bkp
    - cp GSimulation-simd.cpp GSimulation.cpp
  recompile and run
  remark #15301: OpenMP SIMD LOOP WAS VECTORIZED
- rerun and show that the result is now correct

### ver4
This is the clean solution of the ver3 after all modification done live in the
previous session.
One should run this version in the same way as before and:
- show the new performance numbers
- generate the compiler report and look at vectorization remarks regarding unaligned accesses

### ver5
This is the solution of the ver4, with all the allocations replaced by the memory
alignment allocation function.
Running this version allows to see that even modifing the memory allocation functions,
the data is not aligned. One needs to use the function `__assume_aligned(...)`.
Recompile the code adding the option: -DASALIGN.
One should run again this version with the alignment option and:
- show the new performance numbers
- generate the compiler report

This concludes the basic vectorization part of the demo.
At this point, only two topics are missing:
- advanced cache optimization (loop-tiling) (ver6)
- enabling OpenMP (ver7)

### ver6
This is the cache optimized version of the code, without OpenMP.
The performance depends on the size of the tile and the number or particles.
One should run again this version and:
- show the new performance numbers
- generate the compiler report

### ver7
This is the version of the code with OpenMP. Play with the number of threads,
openmp scheduling and threads affinity.

### ver8
This is the version of the code with OpenMP and cache tiling.
One can also play with the floating point model -fp-model fast=2, for example and
look for further performance improvements
