# Install Lammps

These are some notes on how to obtain and install my Lammps clone. This is a slightly modified of the offical LAMMPS code that can be obtained from [http://www.lammps.org](http://www.lammps.org). Some of these modifications will eventual become a part of the official version.

Clone my lammps repository from GitHub

    git clone https://github.com/SGenheden/lammps

Then, install the colvars library (needed for some simulations)

    cd lammps/lib/colvars
    make -f Makefile.g++

Now we are ready to install Lammps. By default Lammps will only contain a limited set of functional. The rest of the functionality can be accessed by installing a set of packages. You can read more about packages [here](http://lammps.sandia.gov/doc/Section_start.html#start_3). Therefore, we will start to add the necessary packages for Elba simulations

    cd ../../src
    make yes-dipole yes-kspace yes-manybody yes-misc yes-molecule yes-rigid yes-user-colvars
    cp USER-MISC/angle_dipole.* USER-MISC/pair_lj_charmm_coul_long_14.* USER-MISC/pair_lj_sf_dipole_sf.* .

The last command add a few bits from the USER-MISC package. This is a large package with user-contributed code and it is unnecessary to install all of it.

Finally, we are ready to compile lammps using an appropriate make file.

Two that work with the OpenMPI and Intell compilers on Southampton machines/clusters can be found in this folder

    module add intel/2013.2 openmpi/1.6.4/intel
    make -j 8 soton

The installation should end with something like this

    size ../lmp_soton
       text        data     bss     dec     hex filename
    8478121      176400   18176 8672697  8455b9 ../lmp_soton
