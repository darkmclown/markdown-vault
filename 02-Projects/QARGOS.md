# MAPDL Distributed SLURM Run Guide

## 1. Purpose

This guide is for running **ANSYS MAPDL/APDL in distributed mode** on SLURM.

Assumptions:

- ANSYS version: `v252`
- ANSYS path: `/fsx/data/ansys_inc/v252`
- Input file: `ds.inp`
- Job is submitted from the same directory where `ds.inp` exists
- License server: `1055@ip-10-0-0-123`
- MAPDL output file: `solve_linux.out`

---

## 2. Queue Reference

| Queue Name | HPC EC2 Instance Type | No. of HPC EC2 Instances | No. of Physical Cores | Total Memory | Memory per Core |
|---|---:|---:|---:|---:|---:|
| `queue1` | `hpc8a.96xlarge` | 2 | 192 | 768 GB | 4 GB/core |
| `hpc7a-16gbpc` | `hpc7a.24xlarge` | 2 | 48 | 768 GB | 16 GB/core |
| `hpc7a-8gbpc` | `hpc7a.48xlarge` | 2 | 96 | 768 GB | 8 GB/core |
| `hpc7a-4gbpc` | `hpc7a.96xlarge` | 2 | 192 | 768 GB | 4 GB/core |

Use the queue name in this SLURM line:

```bash
#SBATCH -p queue1
```

Example:

```bash
#SBATCH -p hpc7a-16gbpc
```

---

## 3. Final SLURM Script

Save as:

```bash
run_mapdl.sh
```

```bash
#!/bin/bash
#SBATCH -J mapdl_dis
#SBATCH -p queue1
#SBATCH -N 1
#SBATCH -n 12                 # Change total MAPDL distributed cores here
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

---

## 4. Change Cores or Queue

### 4.1 Change cores

Change only this line:

```bash
#SBATCH -n 12
```

Example for 24 cores:

```bash
#SBATCH -n 24
```

### 4.2 Change queue

Change only this line:

```bash
#SBATCH -p queue1
```

Example:

```bash
#SBATCH -p hpc7a-16gbpc
```

---

## 5. Submit Job

Go to the folder where `ds.inp` and `run_mapdl.sh` exist.

```bash
sbatch run_mapdl.sh
```

Do not run with:

```bash
bash run_mapdl.sh
```

If you run with `bash`, SLURM variables like `$SLURM_NTASKS` may be blank.

---

## 6. Monitor Job

### 6.1 Best watch command

```bash
watch -n 2 'squeue -u $USER; echo; ls -lh *.out *.err solve_linux.out 2>/dev/null'
```

### 6.2 Detailed queue and node watch

```bash
watch -n 2 'squeue -u $USER -o "%.10i %.12P %.20j %.8u %.2t %.10M %.6D %R"; echo; sinfo -N -l'
```

### 6.3 Tail MAPDL output

```bash
tail -f solve_linux.out
```

### 6.4 Tail SLURM output and error

Replace `74` with your job ID.

```bash
tail -f mapdl_74.out mapdl_74.err
```

---

## 7. Useful SLURM Commands

Check your jobs:

```bash
squeue -u $USER
```

Check one job:

```bash
scontrol show job <JOBID>
```

Check all nodes:

```bash
sinfo -N -l
```

Find `hpc7a` nodes:

```bash
sinfo -N -l | grep hpc7a
```

Check a specific node:

```bash
scontrol show node <NODE_NAME>
```

---

## 8. SLURM State Notes

| State | Meaning |
|---|---|
| `CF` | Configuring. Node is being prepared or booted. |
| `R` | Running. Job has started. |
| `PD` | Pending. Waiting for resources or queue conditions. |
| `CG` | Completing. Job is finishing cleanup. |

Example:

```text
JOBID PARTITION NAME      USER     ST TIME NODES NODELIST(REASON)
74    queue1    mapdl_di  ec2-user CF 1:39 1     queue1-dy-hpc8a96xlarge-1
```

---

## 9. Clean Working Directory

Preview files that will be deleted:

```bash
find . -maxdepth 1 -mindepth 1 ! -name "*.inp" ! -name "*.sh" -print
```

Delete everything except `.inp` and `.sh` files:

```bash
find . -maxdepth 1 -mindepth 1 ! -name "*.inp" ! -name "*.sh" -exec rm -rf -- {} +
```

---

## 10. Fix UCX Warning

Warning:

```text
UCX WARN transport 'rc' is not available
```

Fix already included in the script:

```bash
export UCX_TLS=tcp,self,sm
```

This forces MPI to use TCP and shared memory instead of RDMA/InfiniBand.

---

## 11. Fix `libGLU.so.1` Error

Error:

```text
error while loading shared libraries: libGLU.so.1: cannot open shared object file
```

Install on the compute node:

```bash
sudo dnf install -y mesa-libGLU
```

If needed:

```bash
sudo yum install -y mesa-libGLU
```

Verify:

```bash
ldconfig -p | grep libGLU
```

Expected output:

```text
libGLU.so.1 => /lib64/libGLU.so.1
```

---

## 12. Install `libGLU` on Dynamic Node

Get the actual node name from:

```bash
squeue -u $USER -o "%.10i %.12P %.20j %.8u %.2t %.10M %.6D %R"
```

or:

```bash
sinfo -N -l | grep hpc7a
```

Then run:

```bash
ssh <NODE_NAME> 'sudo dnf install -y mesa-libGLU && ldconfig -p | grep libGLU'
```

Example for `queue1`:

```
ssh queue1-dy-hpc8a96xlarge-1 'sudo dnf install -y mesa-libGLU && ldconfig -p | grep libGLU'

```

Example for `hpc7a-16gbpc`:

```
ssh hpc7a-16gbpc-dy-hpc7a24xlarge-1 'sudo dnf install -y mesa-libGLU && ldconfig -p | grep libGLU'
```


Use the node name shown by `squeue` or `sinfo`. Do not assume the node name blindly.

---

## 13. Quick Checklist

1. Keep `ds.inp` and `run_mapdl.sh` in the same folder.
2. Set queue:

```bash
#SBATCH -p queue1
```

3. Set cores:

```bash
#SBATCH -n 12
```

4. Submit:

```bash
sbatch run_mapdl.sh
```

5. Monitor:

```bash
watch -n 2 'squeue -u $USER; echo; ls -lh *.out *.err solve_linux.out 2>/dev/null'
```

6. Check solver output:

```bash
tail -f solve_linux.out
```

