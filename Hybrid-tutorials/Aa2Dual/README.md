# Converting an all-atom toluene/DOPC system to hybrid

The *tol_128dopc.gro* file contains the all-atom system of toluene in a DOPC membrane. We will now convert it to a hyrid system where
the DOPC membrane is described with a CG potential but the all-atom description of toluene is retained.

First we will make sure that the system is whole over the periodic boxes

    python $SCRIPTS/Pdb/pdb_makewhole.py tol_128dopc.gro

which will create *tol_128dopc_whole.pdb*

Next, we will convert to a hybrid system with the *aa2cg.py* script. The box information is taken from the end of the gro-file.
The data-file of toluene was taken from the umbrella simulation tutorial

    python $SCRIPTS/Lammps/aa2cg.py tol_128dopc_whole.pdb -i forcefield.elba -a TOL=data.toluene -b 64.1170 68.7128 71.6643 -o elba_toluene

which will create the file Lammps files *data.elba_toluene* and *forcefield.elba_toluene*, as well as pdb-file for visualization.

We need to modify the force field file to incorporate proper empirical scaling parameters.

    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.elba_toluene -b 2,3 4,5 -s 0.5 1.75 -p e c

The Lammps files can now be used to initiate a hybrid simulation. This folder contains an example input file for LAMMPS that could be used to run a simple simulation.
