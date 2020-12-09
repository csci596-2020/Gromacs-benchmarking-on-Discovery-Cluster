

# Project Title : Gromacs benchmarking

## Group 

Trinh Lan Hoa and Samprita Nandi

## Gromacs introduction
https://zenodo.org/record/3923644#.X7vessKIYaw

## Discovery cluster introduction
https://carc.usc.edu/user-information/user-guides/high-performance-computing/discovery-resources?fbclid=IwAR3nNkJvUiTvLwBNyOsBtJD01cyuoDreeLG_GDIC-bAOSSrno6xCo_CvSMY

## Methodology
Study systems: TRP-Cage, a protein with only 20 amino acids (PDB ID = 1L2Y) and Aquaporin (MEM protein).
<figure>
  <img src="https://github.com/hoatrinhusc/Gromacs-benchmark/blob/main/trp_vmd.png"/>
</figure>


Using Gromacs version 2020.3 installed on Discovery cluster.

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
Systems: TRP-Cage.//
Intel(R) Xeon(R) Gold 6130 CPU @ 2.10GHz
<figure>
  <img src="https://github.com/csci596-2020/Gromacs-benchmark/blob/main/1MPI-OpenMP.png"/>
</figure>

<figure>
  <img src="https://github.com/csci596-2020/Gromacs-benchmark/blob/main/OpenMP_speedup.png"/>
</figure>

From the figure, we see that speed up as well as wall clock time is saturated around 15 OpenMP threads. When we use more than 25 OpenMP threads, there is a decrease in performance. Modern computers have a limited number of threads, but even if the number of threads are unlimited, we don't gain speed up due to strong scaling.

**2. MPI as a solution**

To overcome strong scaling of OpenMP and gain speed up, MPI is used to connect different computing nodes. However, the price is the communication between MPI processes.

Gromacs uses the particle-mesh Ewald (PME) algorithms to treat the long-ranged component of the non-bonded interaction. Because the algorithm uses a 3D FFT that requires global communication, its parallel efficiency gets worse as more ranks participate. 

As a study case of MPI communication cost, I consider 20 MPI processes, each MPI process is on a different computing node of xeon-2640v4. 1 OpenMP thread per rank. Then, I varies the number of MPI ranks dedicated for PME work and benchmark MD performance.
<figure>
  <img src="https://github.com/hoatrinhusc/Gromacs-benchmark/blob/main/MPI_PME_xeonv4.png"/>
</figure>

The table below breaks down the computing cost into the cost of each major component ( each accounts for more than 2% of total wall-clock time) . The cost is measured in the total Sum of Giga-Cycles. Note that 20MPI-0PME means no seperate ranks for PME calculations, 20MPI-4PME means among 20 MPI ranks, 4 ranks are dedicated for PME calculation and 16 ranks are used for particle-particle interaction (PP).


| Computing  | 20MPI-0PME | 20MPI-4PME |  20MPI-10PME |
| ------------- | ------------- | ------------- | ------------- |
| Domain decomp. | 72.341  | 48.588 | 40.588 |
| Neighbor search  | 84.652 | 69.198 | 72.632 |
|  Comm. coord. | 297.054 | 207.974 | 148.992 |
| Force | 1887.417 | 1824.295 | 1735.850 |
| Wait + Comm. F | 299.149 | 356.349 | 197.719 |
| PME mesh | 2105.949 | 645.770 | 1136.321 |
| PME wait for PP | 0 | 100.144 | 1219.859 |
| Wait + Recv. PME F | 0 | 288.256 | 7.370 |
| Total PME cost | 2105.949 | 1034.17 | 2363.55 |

According to the table, while other costs are quite the same for 3 cartergories, the PME cost are significantly different. So it makes sense that to study MPI communication cost for the TRP-Cage system, we can focus on PME cost. 

<figure>
  <img src="https://github.com/hoatrinhusc/Gromacs-benchmark/blob/main/PME_breakdown.png"/>
</figure>
 From the figure, we see that when we increase # of PME ranks to 4, there is a significant decrease in cost due to the speed up in PME mesh calculation. But when we increase # of PME ranks to 10, there is a significant increase in cost due to the slow down by the waiting time between PME ranks and PP ranks. 
 This analysis, however, is not complete, since there are other factors like load imbalancing, etc... which we might consider in the future.

**3. Strong scaling using MPI method**

Pure MPI jobs: 

System : Aquaporin (MEM protein). In this section, ntask-per-node = 16, cpu-per-task = 1, we are varying no of nodes like 1,4,10,...,50,60,70,calculations are done paricularly on Xeon silver 4116 processor @ 2.1GHz. The image below shoes the output in terms of performance(ns/day) as we increase the number of resources. 


<img src="https://user-images.githubusercontent.com/43625587/99996974-57a64180-2d71-11eb-9bc6-89bffa2a2069.png" width="500" height="350"/>

We see here expected performance(ns/day) does not scale up linearly beyond 250 corses(15 nodes here). As we change the number of nodes from 15 to 20, adding 33% more hardware only adds 17% more performance. Eventually, it reaces to a saturation limit. We have calculated speedup from the walltime and next graph shows behaviour of the speedup with no of resources. 

<img src="https://user-images.githubusercontent.com/43625587/100000554-968ac600-2d76-11eb-9618-eee054e1ad2c.png" width="500" height="350"/>

When resources used in the simulation  reach to **less than 300 atoms/core** speed up does **not** increase linearly with no of nodes anymore due to strong scaling. Red line denotes the perfect scale up.


<img src="https://user-images.githubusercontent.com/43625587/99996974-57a64180-2d71-11eb-9bc6-89bffa2a2069.png" width="500" height="350"/>


Next, efficiency is calculated from the result and shown below. 

<img src="https://user-images.githubusercontent.com/43625587/99996977-58d76e80-2d71-11eb-81eb-b74041ed1fa1.png" width="500" height="350"/>

**4. Hybrid method(MPI+Openmp+GPU)**

In this section, we are checking the performance output(ns/day) as we increase the number of resources both in terms of cpu and gpu. Here, ntasks-per node = 4,cpus-per-task=2 and gpus-per-node = 1 on on Xeon 2640v4 processor @ 2.4GHz. As we increase the number of nodes, the number of gpus requested also increase linearly so as the MPI ranks. 

<img src="https://user-images.githubusercontent.com/74804041/101670915-a5c17300-3a08-11eb-8755-58659c3b266e.png" width="500" height="350"/>

From the figure, we see that as we increase gpus and cpus, after a optimum level  there is a significant decrease in the output due to communication cost. So, for this system the optimum resource is 4 nodes + 4 gpus. 



