# Solvation free energy in water and octanol

This tutorial will show how to compute the solvation free energy of toluene in CG water and octanol.

As a starting point, we will use toluene solvated in atomistic water and octanol. We will also assume that toluene already have been parametrized. How to solvate in atomistic solvent and the parametrization of small solutes are explained in other tutorials, which you can find [here](https://github.com/SGenheden/Tutorials/).


The procedure is similar to the one used in the following publication:
    [Predicting Partition Coefficients with a Simple All-Atom/Coarse-Grained Hybrid Model](http://dx.doi.org/10.1021/acs.jctc.5b00963)

### Coarse graining the water and octanol

The first step is to coarse-grain the solvent molecules. This is straightforward and will use an in-house script that uses a dictionary to map the CG beads onto the atoms.

The command for the water is

    python $SCRIPTS/Lammps/aa2cg.py toluene_wat.gro -i forcefield.elba -a tol=data.toluene -b 40 40 40 -o toluene_wat

and for octanol

    python $SCRIPTS/Lammps/aa2cg.py toluene_oct.gro -i forcefield.elba -a tol=data.toluene -b 40 40 40 -o toluene_oct

The `toluene_wat.gro` and `toluene_oct.gro` are the solvated structures, `forcefield.elba` is the standard ELBA parameters and `data.toluene` is the parameter-topology file in Lammps file format. The box is 40 x 40 x 40 Å.

This script will create a combined data file for toluene and the solvent as well as a combined force field file, e.g. `data.toluene_wat` and `forcefield.toluene_wat`.

Next, we will introduce some ELBA-specific scaling factors that will improve the interactions between the AA and CG parts.

    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.toluene_wat -b 2,3 4,5 -s 0.5 1.75 -p e c
    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.toluene_oct -b 6 -s 0.9 -p e

The scaling used for water is the standard scaling for membrane simulations, and is actually not needed here. But we will use it anyway to make the input to Lammps consistent. The scaling used for octanol is a 0.9 for the Lennard-Jones potential between AA and the apolar tail bead (_type 6_).

### Solvation free energy

The solvation free energy simulation will be a standard procedure and it is all in the input file `in.ti`. We will equilibrate the system for 1.2 ns before we start the free energy-part: this simulation will apply a coupling parameter between the solute and the surroundings. This coupling will be changed from 0.0 to 0.96 in stages. Each stage will be 3 ns and the first 1.2 ns will be discarded as equilibration.

Let's have a look at some of the input, special to this type of simulation.

The line

    fix adaptSS all adapt 1 pair lj/sf/dipole/sf:1 epsilon 1*6 7*${nt} v_fL &
                        pair lj/sf/dipole/sf:1 scale   1*6 7*${nt} v_fL scale yes

turns on the coupling parameter. It will couple both the van der Waals (`epsilon` setting) and Coulomb (`scale` setting). The coupling will between the CG beads (type 1-6) and the AA (types 7-$nt). `${nt}` is the total number of atom types and it is a variable that will be set as input. The setting `lj/sf/dipole/sf:1 ` make sure that the coupling is between CG and AA beads.


    variable stepNow equal step
    variable L       equal (floor((v_stepNow-1)/v_Nwintot)*v_deltaL)
    variable fL      equal (1-v_L)*(1-v_L)*(1-v_L)*(1-v_L)

which defines a variable for the current step, a variable L (*lambda*) which is a function of the current step and the *fL* which is the fourth power of *1 - lambda*. *Nwintot* is the total number of steps for *lambda* value and *deltaL* is the increment in *lambda*, which is set to 0.04.

The computation of the derivate with respect to *lambda* is carried out with the following lines of input

    compute  PotEngSS  solute group/group solvent pair yes
    variable dPotEngSS equal c_PotEngSS*v_dfL/v_fL
    fix dPotEngSS all ave/time ${Ne} ${Nr} ${Nf} v_L v_dPotEngSS file out.dPotEngSS_${lig}-${solv}

the first line computes the interaction between the solute and the other water molecules the second line computes the derivative with respect to *lambda* and the third line prints it to an ouput file. The combination of *Ne*, *Nr* and *Nf* ensures that the average is only over the sampling portion and that the output is only once for each *lambda*. The explanation for these variables can be found in the Lammps manual.

There is a number of input variable in the `in.ti` that needs to be given when the calculation is run

    lig:  the name of the ligand (e.g. toluene)
    solv: the name of the solvent (e.g. wat or oct)
    nt:   the total number of atom types (10 for toluene)    

**NB!** The number of atom types can be found in e.g. `forcefield.toluene_wat`. The number of masses defined at the top of this file shows the number of atom types.

So to run the calculation, one can for instance use

    mpirun -np 4  $LMPPATH/src/lmp_mpi -i in.ti -var lig toluene -var solv wat -var nt 10

and

    mpirun -np 4 $LMPPATH/src/lmp_mpi -i in.ti -var lig toluene -var solv oct -var nt 10

but the calculations might take a while so they are better run on a cluster.

To estimate the decoupling free energy, i.e. the negative of the solvation free energy, we will use thermodynamic integration (TI), which allows us to extrapolate from 0.96 which is the last value of *lambda* that we simulated and 1.0.

First, put the name of the solute in a list

    cat << EOF > solutes.txt
    toluene
    EOF

and then type following command

    python $SCRIPTS/Lammps/collect_ti.py solutes.txt --postfix wat

for the water, and

    python $SCRIPTS/Lammps/collect_ti.py solutes.txt --postfix oct

for octanol.

The reason we have to put the name of the solute in a list is that the script was made to collect results for a series of ligands. If we have performed a second repeat and put the results in a sub-folder called `R2`, the script would have performed TI on that results as well and computed a standard error. Now the error will be reported as 0.0 kJ/mol because we did not do repeats.

The solvation free energy will be printed in kJ/mol and should be around –3.0 and –15.0 kJ/mol for water and octanol, respectively.
