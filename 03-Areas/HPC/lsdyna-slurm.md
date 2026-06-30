


### QARGOS AWS SLURM 

```
#!/bin/bash
#SBATCH --job-name=lsdyna_60c
#SBATCH --output=lsdyna-%j.out
#SBATCH --error=lsdyna-%j.err

#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --time=04:00:00
#SBATCH --partition=queue1
#SBATCH --nodelist=queue1-dy-hpc8a96xlarge-2


export LSTC_LICENSE=ansys
export LSTC_FILE=/fsx/Hrishikesh/Ls-Dyna/01_Fixture_Design2_ITR14/ansyscl
export ANSYSLMD_LICENSE_FILE=1055@10.0.0.123
# ==============================
# Intel MPI Environment
# ==============================
source /fsx/data/ansys_inc/v252/tp/MPI/Intel/2021.10.0/linx64/env/vars.sh

# ==============================
# CRITICAL FIX (this line matters)
# ==============================
#export I_MPI_PMI_LIBRARY=$(find /usr -name "libpmi*.so" 2>/dev/null | head -n 1)

# ==============================
# Performance tuning
# ==============================
export OMP_NUM_THREADS=1
export I_MPI_PIN=1
export I_MPI_PIN_DOMAIN=core

# ==============================
cd /fsx/Hrishikesh/Ls-Dyna/01_Fixture_Design2_ITR14/

# ==============================
# RUN USING mpirun (NOT srun)
# ==============================
mpirun -np 4 /fsx/data/src/mpp_d_R1420_avx2_impi \
       i=Fixture_Design_2_ITR_14_Shock_Test.k \
       memory=900m memory2=900m \
       > output.log

```


