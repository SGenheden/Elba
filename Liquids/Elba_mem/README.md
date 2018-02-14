# ELBA coarse-grained membranes

This folder contains pre-equilibrated boxes of coarse-grained (CG) membranes. It contains boxes for DMPC, DOPC and POPC. There are 128 lipids and 5120 water molecules. The membranes has been equilibrated at 310 K.

For each solvent, there is a Lammps datafile and a PDB-file for easy visualization. The datafile is supposed to be used with the force field parameters in `forcefield.elba `.

Some of these data-files might be incompatible with the latest Lammps version. However, if you use them with any in-house scripts the output data-file should be alright, e.g.

    python $SCRIPTS/Lammps/lmp_makewhole.py data.dopc
