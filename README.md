

# Project Title : Gromacs benchmarking

## Group

Trinh Lan Hoa and Samprita Nandi

## Gromacs introduction
https://zenodo.org/record/3923644#.X7vessKIYaw

## Discovery cluster introduction
https://carc.usc.edu/user-information/user-guides/high-performance-computing/discovery-resources?fbclid=IwAR3nNkJvUiTvLwBNyOsBtJD01cyuoDreeLG_GDIC-bAOSSrno6xCo_CvSMY

## Methodology
Study system: TRP-Cage, a protein with only 20 amino acids (PDB ID = 1L2Y).
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
| PME grid spacing(nm) | 0.16 | 0.12 |
| Neighbor searching frequency | 10| 10 |
| Benchmark steps | 500000 | 16000 |


**1. Strong scaling of OpenMP**

Intel(R) Xeon(R) Gold 6130 CPU @ 2.10GHz
<figure>
  <img src="https://github.com/hoatrinhusc/Gromacs-benchmark/blob/main/1MPI-OpenMP.png"/>
</figure>

Modern computers have a limited number of threads, but even if the number of threads are unlimited, we don't gain speed up due to strong scaling.

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


**3. Strong scaling of MPI**

Pure MPI jobs
nodes = 1,4,10,...,50,60,70; ntask-per-node = 16, cpu-per-task = 1; first 6 calculation done paricularly on Xeon silver 4116 processor @ 2.1GHz 

<img src="https://user-images.githubusercontent.com/43625587/100000554-968ac600-2d76-11eb-9618-eee054e1ad2c.png" width="600" height="400"/>

When the simulation reaches to **less than 300 atoms/core** speed up does **not** increase linearly with no of nodes anymore. Red line denotes the perfect scale up.


<img src="https://user-images.githubusercontent.com/43625587/99996974-57a64180-2d71-11eb-9bc6-89bffa2a2069.png" width="600" height="400"/>

Similarly, expected performance(ns/day) does not scale up linearly after 15 nodes. As we change the number of nodes from 15 to 20, adding 33% more hardware only adds 17% more performance. Eventually, it reaces to a saturation limit.



<img src="https://user-images.githubusercontent.com/43625587/99996977-58d76e80-2d71-11eb-81eb-b74041ed1fa1.png" width="600" height="400"/>

