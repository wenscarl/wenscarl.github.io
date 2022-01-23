---
layout: post
title: Large Scale Inversion of Subsurface Flow
date: 2020-07-26
published: true
tag: 
- inversion
- parallel computing
- adjoint method
---
## When was this work done?

This is a intern project([**PCRSI**](https://www.lanl.gov/projects/national-security-education-center/information-science-technology/summer-schools/parallelcomputing/2018/2018-projects.php)) at Los Alamos National Lab(LANL) summer 2018. My mentor was Dr. Satish Karra and co-author is Dr. Daniel O'Malley. My sincere thanks to them. 

This project dealt with a typical inversion problem arised in computational geophysics. The goal is to infer the distribution of permeability from observation data of pressure. This project is coded in Fortran and C++(lots of binding), under the framework of [PETSc](https://www.mcs.anl.gov/petsc/). This code is highlighted by hybrid of MPI and openMP in its FEM solver. Actually PETSc doesn't support hybridization by then. The scaling was tested on over 2400 cores. PDF verion is listed [here](/docs/Inversion/1906.01132.pdf) and also accessible by this [link](https://arxiv.org/abs/1906.01132). Also come with a [poster](https://www.lanl.gov/projects/national-security-education-center/information-science-technology/summer-schools/parallelcomputing/_assets/images/2018projects/Shu.pdf).

