---
layout: post
title: Vector fitting for matching microstrip line with cap capacitor
tag: cem
date: 2020-07-25
published: true
categories: cem
---

This document describes how to generate equivalent circuit model for microstrip line  example (Fig 4) in paper *Integration of Arbitrary Lumped Multiport Circuit Networks Into the Discontinuous Galerkin Time-Domain Analysis*. (A **typo** in paper: \[L<sub>s</sub> =0.102nH\]. 

<!--more-->

## Step 1. Generate a half microstrip line model, only port 1 to port 2, and run DGTD to obtain the S-parameters.

+ Go to folder *forDeliver/halfmatchTL* and execute *runMPI.sh*.  The input pulse is set to be 4GHz -12Ghz. This will generate a folder *TimeDomainVoltages*.
+ Run MATLAB file  *computeSpara.m*, to generate s-parameters. Run for both S11 and S12, then a MATLAB data file *halfmatchTL_S11.mat* and *halfmatchTL_S12.mat* will be on disk and then copy them to folder *matchMicroStrip*.
  + S11: set line 16 S11flag = 1 
  + S12: set line 16 S11flag = 0

## Step 2. Run MATLAB file *MicroStripEquiv.m*. 

+ To choose range of data. The vector fitting may **NOT** work well for very complex y-parameters, so sometimes a range of frequency should be selected. As in ``` datarange = 1:length(freq);``` , all data is used. 
+ To choose order of fitting as you need in line ``` opts.N=4```. 
+ routine **N_PORT_circuit** in this script is the working horse. See its own comments.
+ Line 67 is for port 1 to port 2 and line 72 is for port 3 to port 4. Input arguments mean that the node in this circuit starts with *lastNd+1*; index for RLC(VCVS) starts from *lastdev+1*; nodes that interface with external circuit starts from *port_id_start*. 
+ Run this file, you will see MATLAB command line prompts the following: ```Lid1 0 5 -1.056360e-06
  Cid1 5 6 -1.583723e-15```This is the SPICE netlist of circuit. Copy and paste it to file *halfmicrostrip.cir*. 
+ Always this file starts with ```.title XXX```.

## Step 3(a)

Run python script  _MicroStripEquiv.py_. The library_path should be set correctly according to your installation of PySpice. The installation of PySpice and NgSpice shared library can be found [here](https://pyspice.fabrice-salvaire.fr/) and [here](http://ngspice.sourceforge.net/shared.html).  Note, NgSpice Version 30 is recommended over others. Line 55-59 is the gap capacitor. 

The input and output stage is connect in serial with a 50Ohm resistor.

<img src="/images/cem/how.png" alt="how" style="zoom: 33%;" />

So the s-parameters from equivalent should be computed as 
$$
S_{11} = \frac{2V_1-V_s}{V_s}\\
S_{21} = \frac{2V_2}{V_s}
$$
Then run MATLAB script *MicroStripSparaCircuit.m*, you will see the results as:

<img src="/images/cem/s12.png" alt="how" style="zoom: 93%;" />

***

## Step 3(b)

 Since the lump port is not exactly the same as wave port where the equivalent circuit model is obtained from. So change the   $Y_{22}$   to 0.02G at port 2 and port 3 by turning on __Modflag__ in line ``` Modflag=1``` in file _MicroStripEquiv.m_. Then go the process stated in step 3(a) you will see.



<img src="/images/cem/s12_50.png" alt="how" style="zoom: 93%;" />

 



***

Files included:

Folder **forDelivery** contains the DGTD solver in **dgtd_MPI**, geometry files in **geo** and all necessary file to obtain the s-parameter in folder **halfmatchTL**. To start,go to **halfmatchTL** and execute *runMPI.sh*. 

In folder **circuitfittingProject**, there includes necessary MATLAB and python files. Each file should be commented well. 

For any question, please contact shuwang12@unm.edu. 

