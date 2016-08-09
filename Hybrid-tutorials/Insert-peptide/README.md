# Insertion of Kalp23 peptide

### Making force field files

As a starting point we will use the Kalp23 structure created with UCSF Chimera. It contains only heavy atoms and I have aligned it with the z-axis. First, we will add the hydrogens with *tleap* from AmberTools and we will write out an Amber prmtop and prmcrd file that will be converted to Lammps format. We will use a modified *learc.ff99SB* file so that we do not charge the terminal amino acids.


    cat << EOF > leapcom
    source leaprc.ff99SB
    x=loadpdb kalp23_aligned.pdb
    saveamberparm x kalp23.top kalp23.crd
    savepdb x kalp23_leap.pdb
    quit
    EOF

    tleap -f leapcom

Now we can convert the files to Lammps format

    python $LMPPATH/tools/amber2lmp/amber2lammps.py

This will create a *data.kalp23* file that contains force field and topology information. We can use a longer timestep by increasing the masses of the hydrogen atoms through the mass repartition procedure. We will increase the mass of the hydrogens 3 fold. Type


    python $SCRIPTS/Lammps/lmp_massrepart.py data.kalp23 -f 3

which will create *data.kalp23_heavyh*


### Preparing the membrane

We have to modify the datafile of the pre-equilibrated DOPC membrane slightly. First, we have to make the configuration whole, i.e. each molecule should not be broken in the central box. This is important as we will place a peptide next to it and thus introducing new periodicity. Second, we have to insert a new atom type for the final tail bead so that it can be restrained.

To change the atom type of the final tail bead, type

    python $SCRIPTS/Lammps/lmp_mod_atype.py data.128dopc_4232wat -a 1-128:9 1-128:15 -t 7 --increase

which will create *data.128dopc_4232wat_mod*. The final tail beads are atoms 9 and 15 in molecules 1 to 128. *This is of course unique for DOPC*.

To make the configuration whole and to write out a PDB file of this configuration, type

    python $SCRIPTS/Lammps/lmp_makewhole.py data.128dopc_4232wat_mod
    python $SCRIPTS/Lammps/lmp_data2pdb.py data.128dopc_4232wat_mod_whole -d 0:wat 1-128:dop


### Placing the peptide and insert in the membrane

Now we are ready to insert the peptide in the membrane. Initially, we will place it next to membrane and then we will push the membrane with high pressure. To place the protein, type

    python $SCRIPTS/Pdb/next_to.py -m kalp23_leap.pdb -r 128dopc_4232wat_mod_whole.pdb -o kalp23_initial_placed.pdb -s 10 0 0 --fromedge

the resulting PDB-file can be visualized together with the membrane with for instance, VMD

    vmd -m 128dopc_4232wat_mod_whole.pdb kalp23_initial_placed.pdb

Now, we will combine the datafile of the peptide with the pre-equilibrated DOPC membrane. Type

    python $SCRIPTS/Lammps/insert_in_elba.py data.kalp23_heavyh -b data.128dopc_4232wat_mod_whole -f forcefield.elba_mod -p kalp23_initial_placed.pdb --resize -o 128dopc_kalp23
    
    python $SCRIPTS/Lammps/lmp_data2pdb.py data.128dopc_kalp23 -o 128dopc_kalp23_initial.pdb -d 0:wat 1-128:dop 129:kalp23_leap.pdb

This will create *data.128dopc_kalp23* and *forcefield.128dopc_kalp23*. We need to modify the latter to incorporate proper empirical scaling parameters.

    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.128dopc_kalp23 -b 2,3 4,5 -s 0.5 1.75 -p e c

Finally, to run the insertion simulation you will run Lammps with the *in.push* script.

    mpirun -np 8 $LMPPATH/src/lmp_soton < in.push > out.push

this will produce a datafile *data.128dopc_kalp23_pushed* that can be used to initialise further simulations. You can easily convert it to a PDB file with

    python $SCRIPTS/Lammps/lmp_data2pdb.py data.128dopc_kalp23_pushed -w -x -r -d 0:wat 1-128:dop 129:kalp23_leap.pdb
