

# Project Title : Gromacs benchmarking

## Group members

Trinh Lan Hoa and Samprita Nandi

## Gromacs software
Gromacs is one of the fastest molecular dynamics simulation packages. It is used widely by researchers in the fields of computational biology, physics...

More information about Gromacs can be found here: https://zenodo.org/record/3923644#.X7vessKIYaw

## Discovery cluster
https://carc.usc.edu/user-information/user-guides/high-performance-computing/discovery-resources?fbclid=IwAR3nNkJvUiTvLwBNyOsBtJD01cyuoDreeLG_GDIC-bAOSSrno6xCo_CvSMY

## Methodology
Study systems: TRP-Cage, a protein with only 20 amino acids (PDB ID = 1L2Y) and Aquaporin, a membrane protein.

Using Gromacs version 2020.3 installed on Discovery cluster.

TRP-Cage             |  Aquaporin
:-------------------------:|:-------------------------:
![](https://github.com/hoatrinhusc/Gromacs-benchmark/blob/main/trp_vmd.png)  |  ![](https://github.com/csci596-2020/Gromacs-benchmark/blob/main/Aquaporin-Sideview.png)


| MD System  | Trp-Cage | Aquaporin (MEM protein) |
| ------------- | ------------- | ------------- |
| # atoms | 3809  | 81,743 |
| System size (nm)  |  3.39x3.39x3.39 | 10.8×10.2×9.6 |
| Time step (fs) | 2 | 2 |
| Cut-off radii(nm) | 1 | 1 |
| PME grid spacing(nm) | 0.10 | 0.12 |
| Neighbor searching frequency | 10| 10 |
| Benchmark steps | 50000 | 16000 |



**1. Strong scaling of OpenMP**

Systems: TRP-Cage.

Intel(R) Xeon(R) Gold 6130 CPU @ 2.10GHz

Wall clock time             |  Speed-up
:-------------------------:|:-------------------------:
![](https://github.com/csci596-2020/Gromacs-benchmark/blob/main/1MPI-OpenMP.png)  |  ![](https://github.com/csci596-2020/Gromacs-benchmark/blob/main/OpenMP_speedup.png)

In the figures, speed up as well as wall clock time is saturated around 15 OpenMP threads. When more than 25 OpenMP threads are used, there is a decrease in the performance. Modern computers have a limited number of threads, but even if the number of threads are unlimited, we don't gain speed up due to strong scaling.

**2. MPI as a solution**

Systems: TRP-Cage.<br/>
To overcome strong scaling of OpenMP and gain speed up, MPI is used to connect different computing nodes. However, the price is the communication cost between MPI processes.

Gromacs uses the particle-mesh Ewald (PME) algorithms to treat the long-ranged component of the non-bonded interaction. Because the algorithm uses a 3D FFT that requires global communication, its parallel efficiency gets worse as more ranks participate. 

As a study case of MPI communication cost, 50 MPI processes are considered. All the MPI processes are Xeon-2640v4. Then, the number of MPI ranks dedicated for PME work are varied. 

Benchmarking MD performance:
<figure>
  <img src="https://github.com/csci596-2020/Gromacs-benchmark/blob/main/MPI_PME_xeonv4.png"/>
</figure>

The table below breaks down the computing cost into the cost of each major component. The cost is measured in the total Sum of Giga-Cycles. Note that 50MPI-5PME means among 50 MPI ranks, 5 ranks are dedicated for PME calculation and 45 ranks are used for particle-particle interaction (PP), 50MPI-0PME means no seperate ranks for PME calculations.


| Computing  | 50MPI-5PME | 50MPI-20E |  50MPI-0PME |
| ------------- | ------------- | ------------- | ------------- |
| Domain decomp. | 14.303  | 8.631 | 16.906 |
| Neighbor search  | 15.907  | 8.852 | 14.509 |
|  Comm. coord. | 78.006 | 42.957 | 84.993 |
| Force | 362.643 | 244.736 | 239.064 |
| Wait + Comm. F | 183.130 | 71.440 | 81.855 |
| PME mesh | 94.766 | 248.676 | 752.059 |
| PME 3D-FFT Comm | 22.665 | 66.138 | 232.457 |
| PME wait for PP | 10.883 | 37.915  | 0|
| Wait + Recv. PME F | 246.225 | 17.998 | 0|

According to the table, while other costs are quite the same for 3 cartergories, the PME cost are significantly different. So it makes sense that to study MPI communication cost for the TRP-Cage system, we can focus on PME cost. 

<figure>
  <img src="https://github.com/csci596-2020/Gromacs-benchmark/blob/main/FFT%20cost.png"/>
</figure>
 From the figure, we see that when using all 50 MPI ranks for PME calculation, there is a huge increase in the communication cost of 3D Fast Fourier transform. That makes the performance get worse. 
 
**3. Strong scaling using MPI method**

Pure MPI jobs:

System : Aquaporin (MEM protein). In this section, ntask-per-node = 16, cpu-per-task = 1, we are varying no of nodes like 1,4,10,...,50,60,70,calculations are done paricularly on Xeon silver 4116 processor @ 2.1GHz. The image below shoes the output in terms of performance(ns/day) as we increase the number of resources. 


<img src="https://user-images.githubusercontent.com/43625587/99996974-57a64180-2d71-11eb-9bc6-89bffa2a2069.png" width="500" height="350"/>

We see here expected performance(ns/day) does not scale up linearly beyond 250 corses(15 nodes here). As we change the number of nodes from 15 to 20, adding 33% more hardware only adds 17% more performance. Eventually, it reaces to a saturation limit. We have calculated speedup from the walltime and next graph shows behaviour of the speedup with no of resources. 

<img src="https://user-images.githubusercontent.com/43625587/100000554-968ac600-2d76-11eb-9618-eee054e1ad2c.png" width="500" height="350"/>

When resources used in the simulation  reach to **less than 300 atoms/core** speed up does **not** increase linearly with no of nodes anymore due to strong scaling. Red line denotes the perfect scale up.





Next, efficiency is calculated from the result and shown below. 

<img src="https://user-images.githubusercontent.com/43625587/99996977-58d76e80-2d71-11eb-81eb-b74041ed1fa1.png" width="500" height="350"/>

**4. Hybrid method(MPI+Openmp+GPU)**

Systems: TRP-Cage.<br/>
In this section, we are checking the performance output(ns/day) as we increase the number of resources both in terms of cpu and gpu. Here, ntasks-per node = 4,cpus-per-task=2 and gpus-per-node = 1 on on Xeon 2640v4 processor @ 2.4GHz. As we increase the number of nodes, the number of gpus requested also increase linearly so as the MPI ranks. 

<img src="https://user-images.githubusercontent.com/74804041/101670915-a5c17300-3a08-11eb-8755-58659c3b266e.png" width="500" height="350"/>

From the figure, we see that as we increase gpus and cpus, after a optimum level  there is a significant decrease in the output due to communication cost. So, for this system the optimum resource is 4 nodes + 4 gpus. 

**5. Conclusion**

In this study, we have shown the strong scaling limit in two different biomolecule systems consisting of different number of atoms. Thus, we have to choose computational resources wisely to achieve optimum performance without wasting much resources.


