# 🛫&nbsp; PyDGSRD 
This is a Python code that solves 1D hyperbolic conservation laws on nonuniform grids using the state redistribution method.  State redistribution is an algorithm that solves the small cell problem on cut cell grids.  That is, arbitrarily small cells on embedded boundary grids result in overly restrictive maximum stable time steps when using explicit time stepping algorithms. Similar in spirit to flux redistribution by Collela [1], state redistribution relaxes this time step restriction using a simple postprocessing operation.  Of course, this algorithm is most interesting in two and three dimensions, but this one-dimensional code illustrates the important aspects of the algorithm.

<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/srd.png" alt="SRD" width="300" >
</p>
<p align="center"> <i>High order approximation (p = 5) of an advecting pulse on a highly nonunform grid.  The DG solution on each element is plotted with a different colour.</i> <p align="center">
  
## Goal
The goal of this work is to use explicit Runge Kutta time steppers on highly nonuniform grids using the time step restriction 

<p align="center">
<a href="https://www.codecogs.com/eqnedit.php?latex=\Delta&space;t&space;\leq&space;\frac{h}{a}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\Delta&space;t&space;\leq&space;\frac{h}{a}," title="\Delta t \leq \frac{h}{a}" /></a>
</p>

where <a href="https://www.codecogs.com/eqnedit.php?latex=a" target="_blank"><img src="https://latex.codecogs.com/gif.latex?h" title="h" /></a> is the cell size, and <a href="https://www.codecogs.com/eqnedit.php?latex=a" target="_blank"><img src="https://latex.codecogs.com/gif.latex?a" title="a" /></a> is the maximum wavespeed in the numerical solution.   In the following grid, the majority of cells on the grid have size <a href="https://www.codecogs.com/eqnedit.php?latex=a" target="_blank"><img src="https://latex.codecogs.com/gif.latex?h" title="h" /></a> and only three have size <a href="https://www.codecogs.com/eqnedit.php?latex=\alpha&space;h" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\alpha&space;h" title="\alpha h" /></a>
<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/example.png" alt="example"  width="700">
</p>

State redistribution will allow the use of a time step that is proportional <a href="https://www.codecogs.com/eqnedit.php?latex=a" target="_blank"><img src="https://latex.codecogs.com/gif.latex?h" title="h" /></a> _not_ <a href="https://www.codecogs.com/eqnedit.php?latex=\inline&space;\alpha&space;h" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\inline&space;\alpha&space;h" title="\alpha h" /></a>.  The main idea of state redistribution is to temporarily merge, or coarsen, the numerical solution into neighborhoods located on the small cells.
Then, the solution on these neighborhoods is refined back onto the base, nonuniform, grid.

## 🏗&nbsp; Grid generation and preprocessing 
Nonuniform grids on which state redistribution can be applied are generated using `gengrid.py`.  For example, the call

```
python gengrid.py -L -1.0 -R 1.0 -N 100 -MESHTYPE uniform -MERGETYPE LRP
```
randomly generates a nonuniform grid on the interval [-1.0,1.0] with 100 elements.  The cell sizes are generated using a uniform distribution, and merging neighborhoods are generated by merging to the left and right periodically.  The different arguments that `gengrid.py` accepts are explained below:


-N 
number of cells on the grid

-L
left endpoint

-R
right endpoint

-MESHTYPE
* `uniform`: uniform grid.
* `rand`: cell sizes follow a uniform distribution.
* `perturb`: cells are generated by perturbing the endpoints of a uniform grid.
* `power`: cell sizes follow a power law distribution.
* `paper`: this is the grid used to demonstrate one-dimensional SRD in our paper.
* `bdry1`: one small cell on the left boundary, otherwise uniform.
* `brdy2`: one small cell on the left and right boundary, otherwise uniform.
* `brdy3`: one small cell on the left, and two small cells at the center, otherwise uniform.

-MERGETYPE

* `LRNP`: merge to the left, right, non-periodically.
<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/LRPNP.png" alt="mergetype"  width="700">
</p>
<p align="center"> <i>The LRP option merges a small cell until the neighborhood has size TOL on the left and the right of the small cell, with size alpha h, only if it can.</i> <p align="center">


* `LRP`:  merge to the left, right, periodically.
<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/LRP.png" alt="mergetype"  width="700" >
</p>
<p align="center"> <i>The LRP option merges a small cell until the neighborhood has size TOL on the left and the right of the small cell, with size alpha h.</i> <p align="center">


* `LP`: merge only to the left, periodically.
<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/LP.png" alt="mergetype"  width="700" >
</p>
<p align="center"> <i>The LP option merges a small cell until the neighborhood has size TOL only to the left of the small cell, with size alpha h.</i> <p align="center">


* `RP`: merge only to the right, periodically.
<p align="center">
  <img src="https://github.com/andrewgiuliani/PyDGSRD/blob/main/images/RP.png" alt="mergetype"  width="700" >
</p>
<p align="center"> <i>The LRP option merges a small cell until the neighborhood has size TOL only to the right of the small cell, with size alpha h.</i> <p align="center">

All of the above options create neighbourhoods using a specified tolerance (TOL) in the code, whereby cells are merged to the left or to the right until the neighborhood satisfies a size constraint:



After the grid generator finishes, it output three files.  
- The file with extension `.dat`, contains the `N+1` grid endpoints.  
- The file with extension `.pdat` contains the preprocessing information that specifies the merging neighborhoods.  The file contains three columns corresponding to `m`, `M`, and `overlaps` in the code.  `m` and `M` specify the indices of the first and last cell in the merging neighbourhoods.  For example the merging neighborhood associated to cell 5 is made up of cells with indices `m[5], m[5]+1, ..., M[5]`.  `overlaps` contains the number of neighborhoods that overlap each cell in the grid.
- The file with extension `.mdat` contains metadata about the preprocessing stage: cell size to be used in the CFL condition, which merging algorithm was chosen, the merging tolerance, and type of random grid that was generated.

## 🏃🏻‍♀️&nbsp; Running the code 
`PyDGSRD` can be called after grid generation with, for example,
```
python PyDGSRD.py -P 5 -T 1 -G grid_100
```
which computes a sixth order (`p = 5`) approximation to the solution at the final time (`T = 1`) on `grid_100`.
The different arguments that `PyDGSRD.py` accepts are explained below:

-P
polynomial degree of approximation

-T
final time

-G
grid filename (without any file extensions!)

-PLOT
plot the numerical solution at the final time using matplotlib

## 🧪 &nbsp; Examples

1. Generates the grid in the figures of this readme.  The small cell volume fraction, `alpha`, is set to 1e-5 in the code, but this can be modified.
```
python gengrid.py -L -1.0 -R 1.0 -N 100 -MESHTYPE bdry3 -MERGETYPE LRP
python PyDGSRD.py -P 5 -T 1.0 -G grid_100
```
2. Reproduces the one-dimensional convergence test in [2], here, I've chosen a sixth order accurate numerical solution, but this can be changed.  Merging neighbourhoods are created by merging to the left and right of each small cell.
```
python gengrid.py -L -1.0 -R 1.0 -N 100 -MESHTYPE paper -MERGETYPE LRNP 
python PyDGSRD.py -P 5 -T 1.0 -G grid_100
```
3. Generates a grid where the cell sizes follow a power law distribution.  This means that the grid is composed of vastly different cell sizes.
```
python gengrid.py -L -1.0 -R 1.0 -N 100 -MESHTYPE power -MERGETYPE LRP
python PyDGSRD.py -P 5 -T 1.0 -G grid_100
```

## ☎️ &nbsp; Contact
For help running the code, or any other questions, send me an email at
`giuliani AT cims DOT nyu DOT edu`

## 📓&nbsp; License
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)



## References

[1] Colella, Phillip, et al. "A Cartesian grid embedded boundary method for hyperbolic conservation laws." Journal of Computational Physics 211.1 (2006): 347-366.

[2] Giuliani, Andrew and Berger, Marsha. "A state redistribution method for discontinuous Galerkin methods on curvilinear embedded boundary grids".

