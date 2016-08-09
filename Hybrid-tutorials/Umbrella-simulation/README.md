# Umbrella simulation of toluene

### Making force field files

    cd Prep

First, we will make the force field for the toluene molecule using *antechamber* from the AmberTools package. Type

    antechamber -i toluene.pdb -fi pdb -o toluene.prepi -fo prepi -c bcc -pf yes
    rm -rf sqm*

Next, we need to convert the tolune force field to lammps format. There is a tool to this with the Lammps distribution, but first we
need a prmtop and prmcrd file. We will create them with *tleap*.

    cat << EOF > leapcom
    source leaprc.gaff
    loadamberprep toluene.prepi
    #loadamberparams toluene.frcmod
    x=loadpdb toluene.pdb
    saveamberparm x toluene.top toluene.crd
    savepdb x toluene_leap.pdb
    quit  
    EOF

    tleap -f leapcom

If the molecule contains some parameters not in the General Amber force field, you can create them with the *parmchk* tool and load the frcmod-file in *tleap*.

Now we can convert the prmtop and prmcrd files to Lammps format

    python $LMPPATH/tools/amber2lmp/amber2lammps.py

This will create a *data.toluene* file that contains force field and topology information. We will insert toluene in a DOPC membrane at specific positions along the membrane. This is done with in-house scripts. The files *data.128dopc_4232wat* and *forcefield.elba* contains the pre-equilibrated membrane coordinates and the ELBA force field parameters, respectively. Type

    python $SCRIPTS/Lammps/insert_in_elba.py data.toluene -b data.128dopc_4232wat -f forcefield.elba  -z {0..30} -o elba_toluene

which will creare *data.elba_toluene_z?* and *forcefield.elba_toluene*. The former contains that coordinates, charges, dipole moments and topology and the latter contains force field parameters. We need to modify the latter to incorporate proper empirical scaling parameters.

    python $SCRIPTS/Lammps/scale_elba_pairs.py forcefield.elba_toluene -b 2,3 4,5 -s 0.5 1.75 -p e c

### Inserting in a membrane

    cd ../Grow

The next step is to "grow" the toluene from a non-interacting molecule to a fully interacting one, such that it is properly inserted in the membrane. There is a *in.grow* file supplied to perform just this. It is commented rather well and should be rather easily to understand. The *fix adapt* fix will be used to scale the van der Waals and electrostatic interaction between the AA and CG beads and we will scale them with the third power of *lambda*, where *lambda* will be increased gradually from 0 to 1.0. We will change the value of *lambda* with 0.005 every 500 steps, and thus will simulate for 100 ps in total.

To run the simulation type for instance

    mpirun -np 8 $LMPPATH/src/lmp_soton -i in.grow -var z 0

the last 0 is the depth and you need to run 31 simulation with this value set to 0 through 30. Each simulation take about 20 minutes with 8 cores.

Each simulation will create a datafile with the final snapshot from the simulation.

If the simulation crashes due to instabilities, you could try to increase the simulation time, but if it is just a few, you can copy the datafile from a neighbouring depth. This will be alright as a starting point for the umbrella simulation.

### Umbrella simulation

    cd ../Umbrella

We will start with preparing input for the colvars module. This is done with the *make_colvars.py* script. Type

    python $SCRIPTS/Lammps/make_colvars.py ../Prep/data.elba_toluene_z0 -m 1 128 -s 129 -z {0..30}

it does not matter which data file your are using. This script find the atom range of the lipids (molecule 1 to 128) and the atom range of the solute (molecule 129). It will create 31 different files for the different z-depth. They will be named *colvars.z?*.

To run the umbrella simulation type for instance

    mpirun -np 8 $LMPPATH/src/lmp_soton -i in.umbrella -var z 0    

As with the growing-simulation, you have to run 31 simulation at different depth. The output will be placed in separate subfolders for each depth. When the simulations are finished, you may copy results file to the *Umbrella* folder and remove the temporary folders. The above command is to run one simulation at a time. However, Lammps can run ensemble jobs very easily. First, you need to uncomment line 11 in *in.umbrella* so that the z-variable is defined by its universe index, i.e. its partition. Then execute the following command (preferably on a cluster).

    mpirun -n 496 $LMPPATH/src/lmp_soton -partition 31x16 -log log.lammps -screen stdout.lammps -i in.umbrella

The final step is the analyse the umbrella simulation with WHAM. This will be performed with the *calc_pmf.py* script. Type

    python $SCRIPTS/Membrane/calc_pmf.py -f z{0..30}.colvars.traj -c {0..30} -w 2.5 -m lammps

the *wham.pmf* files will contain the potential of mean force (PMF) and it is plotted in *pmf_pmf.png*. You should also look at the *pmf_hist.png* to make sure that you have sufficient overlap of the different histograms.
