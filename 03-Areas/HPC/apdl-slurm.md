
###  AWS working  slurm script QARGOS

```
#!/bin/bash
#SBATCH -J mapdl_dis
#SBATCH -p queue1
#SBATCH -N 1
#SBATCH -n 64                 # <-- Change total MAPDL distributed cores here
#SBATCH -t 24:00:00
#SBATCH -o mapdl_%j.out
#SBATCH -e mapdl_%j.err

export AWP_ROOT252=/fsx/data/ansys_inc/v252
export PATH=$AWP_ROOT252/ansys/bin:$PATH
export ANSYSLMD_LICENSE_FILE=1055@ip-10-0-0-123

# Use TCP/shared-memory MPI transport when RDMA/InfiniBand is not available
export UCX_TLS=tcp,self,sm

NP=$SLURM_NTASKS

echo "Running MAPDL distributed"
echo "Host  : $(hostname)"
echo "PWD   : $PWD"
echo "Cores : $NP"
date

ansys252 -b -dis -np "$NP" \
  -dir "$PWD" \
  -i ds.inp \
  -o solve_linux.out \
  -j demo_RSM

echo "Completed at $(date)"

```


