# PLUMED Masterclass 21.5: Simulations with multiple replicas

## Origin

This masterclass was authored by Giovanni Bussi on March 15, 2021

## Aims

In this Masterclass, we will discuss how to perform and analyze multi-replica simulations where the different replicas feel different bias potentials. We will also learn how to compute statistical errors on the computed quantities.

## Objectives

Once you have completed this Masterclass you will be able to:

- Use PLUMED and GROMACS to run multiple-replica simulations.
- Use WHAM to combine multiple simulations performed with different bias potentials.
- Calculate error bars on free energies and populations, taking into account correlations induced by replica exchanges.
 
## Setting up PLUMED

For this masterclass you will need versions of PLUMED and GROMACS that are compiled using the MPI library.  The versions used in some of the other masterclasses will thus not work properly.  In order to obtain the correct versions, please use the following commands:

````
conda install --strict-channel-priority -c plumed/label/masterclass-mpi -c conda-forge plumed
conda install --strict-channel-priority -c plumed/label/masterclass-mpi -c conda-forge gromacs
````

The `--strict-channel-priority` is necessary as it prevents your conda install from downloading packages from the `bioconda` channel. `bioconda` contains a version of GROMACS that is **not** patched with PLUMED and would not work here.  Similarly, the channel `plumed/label/masterclass-mpi` should receive a priority higher than `conda-forge`, so as to install the MPI version of PLUMED.

On Linux, the command above should install the following packages:

````
  gromacs            plumed/label/masterclass-mpi/linux-64::gromacs-2019.6-h3fd9d12_100
  plumed             plumed/label/masterclass-mpi/linux-64::plumed-2.7.0-h3fd9d12_100
  mpi                conda-forge/linux-64::mpi-1.0-openmpi
  openmpi            conda-forge/linux-64::openmpi-4.1.0-h9b22176_1
  [ etc ... ]
````

The exact versions might be different. Notice however that GROMACS and PLUMED come from the `plumed/label/masterclass-mpi` channel, whereas the required libraries come from the `conda-forge` channel.
To be sure the installed GROMACS is compiled with MPI patched with PLUMED, try the following shell command:

````
gmx_mpi mdrun -h 2> /dev/null | grep -q plumed && echo ok
````

It should print `ok`.
To be sure that PLUMED has been compiled with MPI, try the following shell command:

````
plumed --has-mpi && echo ok
````

It should print `ok`.

Please ensure that you have setup PLUMED and GROMACS on your machine before starting the exercises. Also notice that in order to obtain good performances it is better to compile GROMACS from source on the machine where you are running your simulations. You can find out in the PLUMED documention how to patch GROMACS with PLUMED so as to be able to install it from source. For this tutorial, the conda precompiled binaries will be sufficient.

## Resources

The data needed to execute the exercises of this Masterclass can be found on [GitHub](https://github.com/plumed/masterclass-21-5).  You can clone this repository locally on your machine using the following command:

````
git clone https://github.com/plumed/masterclass-21-5.git
````

_All the exercises were tested with PLUMED version 2.7.0 and GROMACS 2019.6_

## Exercises

Throughout this tutorial we will run simulations of alanine dipeptide in vacuum using GROMACS and PLUMED.  Whereas this system is too simple to be considered a proper benchmark for enhanced sampling methods, it is complex enough to be used in learning about them. Notice that, although PLUMED has a portable interface, the support for replica-exchange simulations is limited and depends on the specific molecular dynamics engine that you have patched with plumed.  You should check the documentation of the MD code you are using to know if replica exchange simulations will work correctly with PLUMED.

_At the time of this writing there is a bug in the rendering of the manual for PLUMED 2.7. In particular,
all pages containing an example that requires multiple replicas are truncated. Since there is no new features
in v2.7, you are recommended to switch to the v2.6 manual. To do so, just replace the string
`doc-v2.7/user-doc` with the string `doc-v2.6/user-doc` in the address bar._

### Introduction to replica simulations

Many methods are based on the simultaneous simulation of multiple replicas. In some cases, all the replicas will use the same input file, whereas in other cases a separate input file should be provided for each replica.  Notice that using a single input file does not imply that all the replicas will feel the same biasing potential.  Biasing potentials in PLUMED might be history dependent, and the history for one replica might differ from the history for the other replicas, the potentials might, therefore, end up being different.

PLUMED has been designed so that multiple-replica simulations can be run even if all the replicas are acting in the same directory. In order to avoid clashes in output files, PLUMED will append a suffix corresponding to the index of the replica to the name of each output file (for instance, the command `PRINT FILE=colvar.dat` will print on a file names `colvar.0.dat` in the first replica, etc.).  Suffixes will be added also to input files, so that if you run a simulation where the input file is `plumed.dat`, the first replica will open a file named `plumed.0.dat`, and so on. However, for input files, if the file including the suffix does not exist, PLUMED will look for the file without the suffix (in the example the file called `plumed.dat`). This provides maximum flexibility and allows to manage case where the input file is the same for all replicas and cases where specific and different input files should be used for each replica.

In addition to this, it is possible to use a [special-replica-syntax](https://www.plumed.org/doc-master/user-doc/html/special-replica-syntax.html) that allows one to use different inputs for the various replicas even thought they all share the same `plumed.dat` file. For instance, the command `RESTRAINT ARG=d AT=@replicas:1.0,1.1,1.2 KAPPA=1.0` will apply restraints at different positions for the three replicas.

Notice however that starting with GROMACS 2019 replica simulations are forced to run in separate directories. To exploit the possibility to use a single input file, one should put it in the parent directory and refer to is as `-plumed ../plumed.dat`.  Output files will be produced in separate directories by default, but their names will be suffixed. If you want the PLUMED output files to be in the parent directory, just prepend their name with `../` (as in `PRINT FILE=../colvar.dat`).

In order to use multiple-replica methods, you should run your simulation using MPI. This can be done prefixing your command with `mpiexec -np N --oversubscribe`, where `N` is the number of processes that you want to use and the `--oversubscribe` option is an OpenMPI option that is required to use more processes than the number of available processors. This is typically suboptimal, but we will need it in our lectures to run, e.g., simulations with 32 replicas on a computer with 4 cores.

In brief, to run a GROMACS simulation where the individual replicas are in directories names `dir0`, `dir1`, etc and the `plumed.dat` file is in the parent directory you will need a command such as

````
mpiexec -np 16 --oversubscribe gmx_mpi mdrun -multidir dir? dir?? -plumed ../plumed.dat
````

To run the PLUMED driver processing a trajectory with multiple processes you will need a command such as

````
mpiexec -np 16 --oversubscribe plumed driver -multi 16 -plumed plumed.dat --ixtc traj.xtc
````

If you have random crashes on MacOS, try to set this environemnt variable:

```bash
export OMPI_MCA_btl="self,tcp"
```

### Exercise 1: Multiple-windows umbrella sampling with replica exchange

In an earlier masterclass on Umbrella sampling you learned how to run a multiple-windows umbrella sampling simulation simulation with multiple independent simulations. Here we will run it using replica exchange. The only differences are that:

- Simulations should be run at the same time using `mpiexec`
- You will have to specify a stride for GROMACS to attempt coordinate exchanges, using the `-replex` option.

It will be sufficient to use a single `plumed.dat` file that looks like this:

```plumed
#SOLUTIONFILE=work/plumed_ex1.dat
# vim:ft=plumed
MOLINFO STRUCTURE=../reference.pdb
phi: TORSION ATOMS=__FILL__
psi: TORSION ATOMS=__FILL__
bb: RESTRAINT ARG=phi KAPPA=200.0 AT=@replicas:__FILL__
PRINT ARG=phi,psi,bb.bias FILE=../colvar_multi.dat STRIDE=100
```

In accordance with the instructions above, you should create 32 directories (one per replica), place the tpr file (for this exercise: `topolA.tpr`)
in each of them, and run the following command

````
mpiexec -np 32 --oversubscribe gmx_mpi mdrun -multidir dir? dir?? -plumed ../plumed.dat -s topolA.tpr -replex 200 -nsteps 200000
````

Notice that by omitting the `-replex` option you will be able to run a non-replica-exchange umbrella sampling simulation, that is identical to the one you performed in the earlier exercise.
You should now be able to exercises 4 and 6 from [this masterclass](https://plumed-school.github.io/lessons/21/003/data/) on umbrella sampling without replica exchange. 
Notice that this also means testing what happens when the initial conditions are different as is discussed in exercise 4 of the previous masterclass.  To understand the effect this has we 
would recomment running the following four simulations:
- Starting from state A, without `-replex` (this is identical to what was done in exercise 4 of the previous masterclass)
- Starting from state B, without `-replex` (this is identical to what was done in exercise 6 of the previous masterclass)
- Starting from state A, with `-replex 200`
- Starting from state B, with `-replex 200`

You should perform a WHAM analysis for each of the four simulations to compute the weights of each frame.  You will then be able to compute 
the relative stability of the two minima using what you learned in exercise 5 of [the masterclass on umbrella sampling](https://plumed-school.github.io/lessons/21/003/data/). 

To compute weights you need to take the following steps:

1. Concatenate the trajectories (`gmx_mpi trjcat -cat -f dir?/traj_comp.xtc dir??/traj_comp.xtc -o traj_multi.xtc`).
2. Run plumed driver on the concatenated trajectory (`mpiexec -np 32 --oversubscribe plumed driver --ixtc traj_multi.xtc --plumed plumed.dat --multi 32 --trajectory-stride 100`).
3. Read the resulting trajectories, perform WHAM, and compute relative population of the two states by adapting this script:

```python
import wham
kb=0.008314462618
T=300
col=[]
for i in range(32):
    col.append(plumed.read_as_pandas("colvar_multi." + str(i)+".dat"))
bias=np.zeros((len(col[0]["bb.bias"]),32))
for i in range(32):
        bias[:,i]=traj[i]["bb.bias"]
w=wham.wham(bias,T=kBT)
tr=col[0].phi
is_in_B=np.int_(np.logical_and(tr>0,tr<2))
is_in_A=np.int_(tr<0)
print(np.average(is_in_B,weights=np.exp(w["logW"]))/np.average(is_in_A,weights=np.exp(w["logW"])))
```

Now answer the following questions:
- Is the population different in the four runs?
- Is there the same dependence on initial condition that you saw in [the previous masterclass](https://plumed-school.github.io/lessons/21/003/data/) when replica exchange is used?

### Exercise 2: Demuxing your trajectories

Close to the end of one of the `md.log` files produced by your simulation you will find a short report that tells you about the accepted exchanges. This report might look like this:

````
Repl  average probabilities:
Repl     0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23   24   25   26   27   28   29   30   31
Repl      .30  .32  .29  .24  .21  .17  .16  .21  .31  .27  .28  .26  .23  .21  .16  .08  .11  .23  .28  .27  .31  .32  .31  .36  .34  .25  .18  .01  .21  .28  .26
````

A result like this one warns you that there is a bottleneck between replicas 27 and 28 (only 1 percent of the attempted exchanges have been accepted). Bottlenecks might be not visible in this
representation, however. When it comes to determining if there are bottlenecks the full path in the replica-index space of the continuous trajectories ("demuxed") is much more informative.

With this in mind we will now "demux" our trajectories. For these short trajectories we can use the `demux.pl` script provided by GROMACS.  Notice that for long trajectories and frequent exchanges this script sometimes has trouble processing the output file correctly.
Trouble arrises because the value of time is written by GROMACS with a limited number of digits. This can make the original script confused about when exchanges happened. At [this link](https://github.com/srnas/demux) you can find a modified script that solves this problem by asking you the value of the time step.  The new script can then compute the value of time from the step number.  As the step number is stored as an integer and does not suffer roundoff problems.

The demux script can be used to produce files named `replica_temp.xvg` and `replica_index.xvg` by issuing the following command:

````
demux.pl dir0/md.log
````

The `replica_temp.xvg` files provides the number of the replica on which each of the continuous simulations is located as a function of time. For instance, you can follow the migration in the replica ladder of the first replica by using the following python script:

```python
replica_temp=np.loadtxt("replica_temp.xvg")
plt.plot(replica_temp[:,0],replica_temp[:,1])
```

Column 1 contains the time, whereas the number on column i+1 tells you the the current replica index (that is: temperature, in a temperature replica exchange simulation; position of the restraint in a replica-exchange umbrella sampling
simulation) for the continuous simulation that started at position i.  This file is called "replica_temp" because it has been implemented with temperature replica exchange in mind, but here the index actually refers to the position of the restraint.

Now answer the following two questions:
- Does any replica explore the full range of indexes?
- Are all the continuous replicas able to explore the full range of indexes?

Notice that each row of the `replica_temp.xvg` file contains a permutation. The `replica_index.xvg` file just contains the inverse of this permutation.  The `replica_index.xvg` can be used to generate the "demuxed" (continuous) trajectories with the following command:

```python
import subprocess
subprocess.run("gmx_mpi trjcat -cat -f dir?/traj_comp.xtc dir??/traj_comp.xtc -demux replica_index.xvg -o " + ''.join([" "+str(i)+"_trajout.xtc" for i in range(32)]),shell=True)
```

The resulting trajectories can be visualized or analyzed as usual and, at variance with the original trajectories, will have no jumps or discontinuities.  They are instead continuous functions of time. You can thus use `plumed driver` to compute phi on the demuxed trajectories.

Now answer the following two questions:
- Does any replica jump from the metastable state at negative phi to the one at positive phi (or viceversa)?
- Are all the continuous replicas able to do so?
- Are these two questions related to the two questions above?

Notice that, if you run your simulation long enough, each "demuxed" trajectory is expected to uniformly cover the full range of replica indexes. Due to the location of the restraints, this will imply that each "demuxed" trajectory is expected to cover in an approximately uniform manner the range of the biased CV. Each of these trajectories should, to a certain extent, behave similarly to a metadynamics simulation.  However, the flatness of the distribution on the biased CV depends on the specific parameters of the restraints (stiffness and locations).

### Exercise 3: Block analysis from demuxed trajectories

Notice that the WHAM analysis does not need to know where each of the frames come from. This implies that when you run WHAM you can do it equivalently using the concatenation of the original trajectories or the concatenation of the "demuxed" trajectories. The advantage of the latter choice is that you can then perform a block analysis on the resulting trajectory where the number of blocks is exactly equal to the number of replicas.  These blocks will be independent simulation, with only two small exceptions:

- the paths in replica space are partly constrained, since when a replica goes up another replica goes down.
- replicas might be initialized from correlated conformations (e.g., all of them in A), inducing a correlation.

The second factor can be decreased by improving the way replicas are initialized. The first factor usually impacts the correlation much less than the actual exchanges. These blocks are thus optimally suited to perform a bootstrap analysis of the error without incurring any underestimation due to correlations between blocks.

There is a small tricky issue here. When we perform the bootstrap analysis, we are going to pick each block a different number of times.  Since each block (that is: each "demuxed" trajectory) has been spanning the replica indexes by spending a different time at each replica, the bootstrap trajectory will no longer spend the same time in each replica.  The included wham script allows to use this information passing an additional option `traj_weight`.  You can adjust the script below to perform the bootstrap analysis:

```python
bias=np.zeros((2001*32,32))
! demux.pl dir0/md.log
replica_temp=np.loadtxt("replica_temp.xvg")
replica_temp=np.int_(replica_temp[:,1:]) # ignore first column (time) and convert to int
for i in range(32):
    col=plumed.read_as_pandas("colvar.{}.dat".format(i))
    bias[:,i]=col["bb.bias"]

# here is the calculation done using the full trajectory
w0=wham.wham(bias,T=kb*T)
tr=col.phi
is_in_B=np.int_(np.logical_and(tr>0,tr<2))
is_in_A=np.int_(tr<0)
# here is the resulting ratio in the population of the two minima:
print(np.average(is_in_B,weights=np.exp(w0["logW"]))/np.average(is_in_A,weights=np.exp(w0["logW"])))

# now we run the bootstrap analysis
pop=[]
for i in range(200): # will require some time, you can first play with less than 200 iterations
    # here we pick the blocks
    c=np.random.choice(32,32)
    # here we count how much time was spent in each replica for the resulting trajectory
    tr_w=np.zeros(32)
    for k in range(32):
        tr_w+=np.bincount(replica_temp[:,c[k]],minlength=32)
    # we then use wham. The traj_weight option can be used to tell to the script
    # how much time was spent at each replica
    w=wham.wham(bias.reshape((32,-1,32))[c].reshape((-1,32)),T=kb*T,traj_weight=tr_w)
    tr=np.array(col.phi).reshape((32,-1))[c].flatten()
    is_in_B=np.int_(np.logical_and(tr>0,tr<2))
    is_in_A=np.int_(tr<0)
    pop.append(np.average(is_in_B,weights=np.exp(w["logW"]))/np.average(is_in_A,weights=np.exp(w["logW"])))

# and here we print average and standard deviation
print(np.average(pop),np.std(pop))
```

Notice that this approach is not really standard, so use it with care.  There are a few papers in the literature discussing similar ideas, but they usually require estimating the autocorrelation time in advance.

### Exercise 4: Bias-exchange metadynamics

We will now run a bias-exchange simulation of alanine dipeptide. In bias-exchange simulations, each replica biases a different collective variable. This is a very practical way to enhance sampling for a large number of variables (as many as the replicas that you can afford!). Notice that they will be biased one at a time. As a matter of fact only those that are useful in identifying a transition state will help, but the other ones will not hurt.

Prepare the input file for a simulation with 3 replicas where the following variables are biased:
- `phi`
- `psi`
- none of them

Initialize two of the replicas in structure A (using topolA.tpr) and one of the replicas is structure B (using topolB.tpr).  Notice that for plumed you can use a single input file that looks like this:

```plumed
#SOLUTIONFILE=work/plumed_ex2.dat
# vim:ft=plumed
MOLINFO STRUCTURE=../reference.pdb

# this is needed to allow arbitrary pairs to try exchanges
# in this case, 0<->1, 0<->2, and 1<->2
RANDOM_EXCHANGES

phi: TORSION ATOMS=__FILL__
psi: TORSION ATOMS=__FILL__
# You can use the same parameters that you used in masterclass 21.4
m: METAD ...
  ARG=@replicas:phi,psi,phi
  SIGMA=@replicas:__FILL__
  HEIGHT=__FILL__ # make sure that there is no bias on the third replica!
  BIASFACTOR=__FILL__
  PACE=__FILL__
  GRID_MIN=-pi
  GRID_MAX=+pi
...
```

Now run two separate simulations for 1000000 steps per replica. In one of them you will propose exchanges between replicas with a pace 200, in the other you will not propose any exchange (just omit `-replex 200` from the command line). The second run will thus be equivalent to running three simulations (free, metadynamics on phi, metadynamics on psi).

We will now use WHAM to combine the resulting trajectories.  We can proceed as we did above, but need to take account of the fact that when analyzing a metadynamics simulations the way the weight is computed is slightly different. As discussed in other masterclasses, one of the possible ways of obtaining the weight is to use the final potential computed along the trajectory. This requires a further processing step when doing a simple metadynamics simulation.  However, here we can compute the final potential while processing the concatenated trajectory.  In practice, you simply need to use a plumed input file, where `PACE` has been set to a large number and `HEIGHT` set to zero. You can then perform WHAM in the normal way and hence compute the populations of the two metastable states.

After you have calculated the relative populations in the two runs (with and without exchanges), answer the following questions:

- Is the relative population you obtain from the simulation with exchances consistent with what you obtained from metadynamics simulations with no exchanges?
- Are the two simulations (with exchanges and without exchanges) consistent with each other?

Notice that the third replica has been simulated without any metadynamics. This is a so-called neutral replica, that is sometimes used in bias-exchange simulations. You can compute the relative population of the two metastable states directly using the populations in that replica (no post-processing needed!).

- Is the result you obtain from this analysis the same as that you obtain when you use WHAM with all replicas?

Now imagine performing the bias-exchange simulation again usign only two replicas: one of them biasing psi and the other one with no bias. In other words, you would forget a variable on purpose that you know is very important:

- What do you expect will hapen to the resulting populations from this simulation?

### Exercise 5: Parallel-tempering metadynamics

In these final two exercises We will learn how to use parallel-tempering metadynamics. In parallel-tempering metadynamics, sampling is enhanced using parallel-tempering (which enhances all degrees of freedom) and metadynamics(which flattens the histogram for a few selected CVs).  If the biased CV contains a relevant bottleneck and is capable of approximately singling out the corresponding transition state, the corresponding transition will be enhanced as well.  Notice however that if the parallel-tempering side of the algorithm is sufficient to enhance sampling, it is not necessary to bias a CV that can identify the transition state.

The first step will be to prepare our input files. We will use 4 replicas, with temperatures taken from a geometric distribution ranging between 300 and 800K. You should be able to generate the corresponding tpr files using the following script

```python
import numpy as np
import re
T=np.geomspace(300,800,4)
for i in range(len(T)):
    with open("top/grompp.mdp") as f:
        l=f.read()
    with open("top/grompp{}.mdp".format(i),"w") as f:
        # if you use this script on your input files, make sure that 300 only appears
        # on the temperature line! or better replace it with a placeholder string such as __TEMP__
        print(re.sub("300",str(T[i]),l),file=f)
    subprocess.run("mkdir -p ptmetad_{}".format(i),shell=True)
    # we will initialize some replica in A and some replica in B
    if i%2==0:
        conf="A"
    else:
        conf="B"
    # we use -maxwarn 1 here since the grompp file has been adapted from an old gromacs version.
    # in general, only use this option after you have understood that the warning is harmless
    subprocess.run("cd top/; gmx_mpi grompp -f grompp{}.mdp -c conf{}.gro -maxwarn 1 -o ../ptmetad_{}/topol.tpr".format(i,conf,i), shell=True)
```

You will then be able to run a parallel tempering simulation with the following command

````
mpiexec -np 4 --oversubscribe gmx_mpi mdrun -multi ptmetad_? -replex 200
````

Notice that the acceptance will be computed by GROMACS taking into account the fact that simulations are running at different temperatures.  Also notice that in order to obtain a large enough acceptance given the temperature span, you will need a number of replicas that grows with the square root of the number of atoms in the system. For solvated molecules, you would typically need tens of replicas at least.

We will now add the metadynamics ingredient, by preparing a suitable PLUMED input file.  Since parallel-tempering metadynamics is designed to cope with cases where you do not have a good CV available, we will bias `psi` rather than `phi`!

```plumed
#SOLUTIONFILE=work/plumed_ex3.dat
# vim:ft=plumed
MOLINFO STRUCTURE=../reference.pdb
phi: TORSION ATOMS=__FILL__
psi: TORSION ATOMS=__FILL__
# You can use the same parameters that you used in masterclass 21.4
# However, it is recommended to scale HEIGHT with temperature.
# You can do it either using replicas: syntax in HEIGHT or specifying TAU
# instead of HEIGHT
m: METAD ...
  ARG=psi
  SIGMA=__FILL__
  HEIGHT=__FILL__
  BIASFACTOR=__FILL__
  PACE=__FILL__
  GRID_MIN=-pi
  GRID_MAX=+pi
...
```

The analysis will be simpler here. We will just analyze the first replica (in `ptmetad_0`) as if it was generated using a simple metadynamics simulation.  Now compute the usual ratio between the populations of the two metastable minima and answer the following questions:

- Is the result compatible with what you obtained using umbrella sampling?
- Was biasing psi useful in this case (you can also try to compute the populations from a parallel tempering simulation without metadynamics to answer this question)?

### Exercise 6: Parallel-tempering: pathological case

Repeat the previous exercise, but now place your replicas in the range between 300 and 310 and answer the following questions.

- Is the population of the two states compatible with what you obtained in the other exercises above.
- If not, which is the correct answer? Draw some conclusion on how to detect this type of problem in a realistic situation.
