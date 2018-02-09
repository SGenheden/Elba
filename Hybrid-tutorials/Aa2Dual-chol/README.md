# Converting an all-atom DOPC/cholesterol membrane to hybrid

**NB: For this tutorial, you need my version of LAMMPS to get the correct pair styles**

The file *step5_assembly.pdb* contains an all-atom system of DOPC/cholesterol membrane created with the Charmm membrane builder. It contains 108 DOPC lipids and 20 cholesterol molecules. We will now convert it to a hybrid model with the DOPC lipids described with ELBA CG model, but cholesterol modelled all-atom with Slipid parameters.

First we need to sort the residues in the pdb-file. It is convinient to have the all-atom molecules last in the structure. Also, Charmm membrane builder tends to mix different lipid types.

    python $SCRIPTS/Pdb/sort_residues.py step5_assembly.pdb -r DOP TIP CHL

This will create a *step5_assembly_sorted.pdb* file.

Next, we will convert it to a hybrid model. The box information is taken from the *step5_assembly.str* file.

    python $SCRIPTS/Lammps/aa2cg.py step5_assembly_sorted.pdb -i forcefield.elba -a CHL=data.chol -b 64.5275 64.5275 76.2 -p lj/charmm/coul/long/14 -o 108dopc_20chol

which will create the file Lammps files *data.108dopc_20chol* and *forcefield.108dopc_20chol*, as well as pdb-file for visualization.

We need to modify the force field file to incorporate proper empirical scaling parameters.

    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.108dopc_20chol -b 2,3 4,5 -s 0.5 1.75 -p e c

The Lammps files can now be used to initiate a hybrid simulation.

The head of the Lammps input file need to look like this

    units real
    atom_style hybrid sphere dipole molecular
    pair_style hybrid lj/sf/dipole/sf 12.0 lj/sf/dipole/sf 12.0 lj/charmm/coul/long/14 11.0 12.0
    pair_modify mix arithmetic
    kspace_style pppm/cg 1.0e-5
    special_bonds lj/coul 0.0  0.999999 0.999999
    pair_modify pair lj/charmm/coul/long/14 special lj 0.0 0.0 0.5
    pair_modify pair lj/charmm/coul/long/14 special coul 0.0 0.0 0.8333
    bond_style harmonic
    angle_style hybrid cosine/squared dipole charmm
    dihedral_style charmm

#### Heavy cholesterol

To gain efficiency, it may be advantageous to increase the mass of the hydrogen atoms in cholesterol through the mass repartitioning scheme. If this is done, one can usually propagate the motion of cholesterol with a 4 fs time step.

To increase the mass of the hydrogens (and reduce the mass of the heavy atoms), type

    python $SCRIPTS/Lammps/lmp_massrepart.py data.chol -f 3

which will increase the mass three fold and create the file *data.chol_heavyh*

We can now re-convert the AA system as above and modify the resulting force field file

    python $SCRIPTS/Lammps/aa2cg.py step5_assembly_sorted.pdb -i forcefield.elba -a CHL=data.chol_heavyh -b 64.5275 64.5275 76.2 -p lj/charmm/coul/long/14 -o 108dopc_20chol_heavyh
    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.108dopc_20chol_heavyh -b 2,3 4,5 -s 0.5 1.75 -p e c

This folder contains an example input file for LAMMPS that could be used to run a simple simulation.
