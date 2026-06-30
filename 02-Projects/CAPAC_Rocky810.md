# CAPAC Rocky 8.10 Bootstrap SOP

## 1. Purpose

This SOP explains how to invoke and operate the CAPAC Rocky 8.10 CAE/HPC bootstrap framework.

The framework is designed for Rocky Linux 8.10 nodes used for CAE/HPC workloads, ANSYS-ready GUI access, Slurm scheduling, Munge authentication, MariaDB accounting, NTP synchronization, security/limits tuning, and operational monitoring.

---

## 2. Repository Structure

```text
CAPAC-Rocky810/
├── bootstrap/
│   ├── loader.sh
│   ├── bootstrap.sh
│   └── validate.sh
│
├── modules/
│   ├── common.sh
│   ├── 010-packages.sh
│   ├── 020-vnc.sh
│   ├── 030-NTP.sh
│   ├── 040-security.sh
│   └── 050-slurm.sh
│
├── slurm-tools/
│   ├── slurm-preload.sh
│   ├── slurm-postload.sh
│   └── capac-prepost-template.sbatch
│
└── docs/
    └── slurm.md
```

Runtime Slurm tools are installed under:

```bash
/opt/slurm-tools
```

---

## 3. Bootstrap Invocation

### One-line loader command

```bash
curl -fsSL https://raw.githubusercontent.com/darkmclown/CAPAC-Rocky810/main/bootstrap/loader.sh | \
sudo REPO_URL="https://github.com/darkmclown/CAPAC-Rocky810.git" BRANCH="main" WORKDIR="/opt/CAPAC-Rocky810" bash
```

The loader will:

```text
1. Install prerequisites
2. Clone/update the repository under /opt/CAPAC-Rocky810
3. Validate bootstrap scripts
4. Show menu options
```

### Run from existing clone

```bash
cd /opt/CAPAC-Rocky810
sudo git pull origin main
sudo bash bootstrap/bootstrap.sh
```

### Validation only

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --validate
```

### Manual module selection

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

---

## 4. Bootstrap Execution Modes

```text
1. Automated module execution
2. Manual module selection
3. Validation only
4. Exit
```

Recommended:

```text
Use manual mode during initial deployment.
Use automated mode only after all modules are tested.
```

---

## 5. Module Function Contract

Each module must expose a function based on the module filename.

| Module file | Required function |
|---|---|
| `010-packages.sh` | `packages_main()` |
| `020-vnc.sh` | `vnc_main()` |
| `030-NTP.sh` | `NTP_main()` |
| `040-security.sh` | `security_main()` |
| `050-slurm.sh` | `slurm_main()` |

Important:

```text
Function name is case-sensitive.
Example: 030-NTP.sh expects NTP_main(), not ntp_main().
```

---

## 6. Recommended Module Execution Order

```text
010-packages.sh
020-vnc.sh
030-NTP.sh
040-security.sh
050-slurm.sh
```

Recommended deployment flow:

```text
1. Install common packages
2. Configure VNC/XFCE if GUI access is required
3. Configure NTP/time/network baseline
4. Apply security and solver limits
5. Configure Slurm/Munge/accounting
6. Reboot
7. Validate cluster
```

---

# 7. Module Reference

## 7.1 `010-packages.sh` — Common Packages

Purpose:

```text
Installs common CAE/HPC/ANSYS-ready packages.
Enables EPEL.
Adds admin, monitoring, network, file transfer, development, runtime, and performance tools.
```

Run:

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

Select:

```text
010-packages.sh
```

Validation:

```bash
which htop
which iotop
which gcc
which gfortran
which python3
which rsync
```

---

## 7.2 `020-vnc.sh` — VNC / GUI / X11

Purpose:

```text
Configures remote GUI access for CAE/ANSYS applications.
Installs/configures XFCE, TigerVNC, X11 tools, Mesa/OpenGL libraries, and optimized VNC service.
```

Recommended user:

```text
cadfem
```

Run:

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

Select:

```text
020-vnc.sh
```

Private network mode:

```text
Bind VNC to localhost only? no
Open firewall port for VNC? yes
```

Connect:

```text
<server-private-ip>:5901
```

Secure SSH tunnel mode:

```text
Bind VNC to localhost only? yes
```

Tunnel:

```bash
ssh -L 5901:localhost:5901 cadfem@<server-ip>
```

Open VNC viewer:

```text
localhost:5901
```

Health check:

```bash
sudo systemctl status vncserver@1.service --no-pager -l
sudo ss -ltnp | grep 5901
pgrep -a Xvnc
pgrep -a xfce4-session
```

Expected:

```text
Active: active (running)
Xvnc :1
xfce4-session
```

---

## 7.3 `030-NTP.sh` — NTP / Chrony / IPv4 / Network

Purpose:

```text
Configures Asia/Kolkata timezone, Chrony, Cloudflare time source, IPv4 preference, and safe network tuning.
```

Run:

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

Select:

```text
030-NTP.sh
```

Validation:

```bash
timedatectl
chronyc tracking
chronyc sources -v
getent ahosts time.cloudflare.com
grep -n "precedence ::ffff:0:0/96" /etc/gai.conf
```

Expected:

```text
Time zone: Asia/Kolkata
System clock synchronized: yes
chronyd active
time.cloudflare.com visible
IPv4 precedence configured
```

---

## 7.4 `040-security.sh` — Security / Limits / HPC Performance

Purpose:

```text
Applies internal HPC open-performance baseline.
Disables SELinux/firewall.
Unlocks soft/hard limits.
Enables core dumps.
Raises systemd and kernel limits.
Tunes semaphores/shared memory.
Relaxes SSH/PAM settings for HPC usage.
```

Warning:

```text
Use only on trusted private HPC networks.
Do not use unchanged on public internet-facing servers.
```

Run:

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

Select:

```text
040-security.sh
```

Validation:

```bash
getenforce
grep '^SELINUX=' /etc/selinux/config
systemctl is-enabled firewalld nftables iptables ip6tables 2>/dev/null
ulimit -a
sysctl kernel.sem vm.max_map_count fs.file-max
sshd -t && echo "sshd config OK"
```

Expected:

```text
SELINUX=disabled
firewall disabled/masked
core file size unlimited
open files 1048576
sshd config OK
```

Reboot recommended:

```bash
sudo reboot
```

---

## 7.5 `050-slurm.sh` — Slurm / Munge / MariaDB / Modules

Purpose:

```text
Configures two-node Slurm cluster with Munge authentication, MariaDB accounting,
Environment Modules, OpenMPI modulefile, Intel MPI modulefile placeholder,
HPC group permissions, and reboot persistence.
```

Cluster:

| Role | Hostname | IP |
|---|---|---|
| Master + Compute | `cae-01` | `192.168.2.131` |
| Compute | `cae-03` | `192.168.2.133` |

Defaults:

```text
Cluster name: capac-hpc
Partition: cfd
CPUs per node: 40
Memory per node: 380000 MB
Job user: cadfem
Slurm service user: slurm
Shared group: hpc
Scratch: /home/data/scratch
Modulefiles: /opt/modulefiles
```

Master services:

```text
munge
mariadb
slurmdbd
slurmctld
slurmd
```

Compute services:

```text
munge
slurmd
```

Recommended run order:

```text
1. Run 050-slurm.sh on cae-03 first
2. Run 050-slurm.sh on cae-01 second
3. Let cae-01 sync Munge and Slurm config to cae-03
4. Validate from cae-01
```

Run:

```bash
cd /opt/CAPAC-Rocky810
sudo bash bootstrap/bootstrap.sh --manual
```

Select:

```text
050-slurm.sh
```

Master validation:

```bash
munge -n | unmunge && \
systemctl is-active mariadb slurmdbd slurmctld slurmd && \
scontrol ping && \
sinfo -Nel
```

Compute validation:

```bash
munge -n | unmunge && systemctl is-active slurmd
```

Expected master result:

```text
STATUS: Success
active
active
active
active
Slurmctld(primary) at cae-01 is UP
cae-01 and cae-03 visible
```

---

# 8. Slurm Runtime Tools

Runtime path:

```bash
/opt/slurm-tools
```

Tools:

```text
/opt/slurm-tools/slurm-preload.sh
/opt/slurm-tools/slurm-postload.sh
```

Sample job:

```text
/opt/slurm-tests/capac-prepost-template.sbatch
```

## 8.1 Preload Script

Use at the start of a Slurm job:

```bash
source /opt/slurm-tools/slurm-preload.sh
```

Captures:

```text
Run directory
Start timestamp
Job ID/name/partition
Node list
Task/CPU allocation
Environment snapshot
Module list
Node inventory
Munge health
Slurm health
Queue snapshot
```

## 8.2 Postload Script

Use at the end of a Slurm job:

```bash
source /opt/slurm-tools/slurm-postload.sh
```

Captures:

```text
End timestamp
Duration
sacct accounting summary
Final cluster health
Critical Slurm errors from last 30 minutes
Result file list
```

## 8.3 Standard Pre/Post Job Pattern

```bash
#!/usr/bin/env bash
#SBATCH --job-name=my-solver-job
#SBATCH --partition=cfd
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=40
#SBATCH --time=01:00:00
#SBATCH --output=/home/data/scratch/%x-%j.out
#SBATCH --error=/home/data/scratch/%x-%j.err

set -Eeuo pipefail

source /opt/slurm-tools/slurm-preload.sh

section "APPLICATION RUN"

log "Starting solver command"

# Replace with solver command
srun --nodes=2 --ntasks-per-node=1 --cpus-per-task=40 bash -lc '
hostname
echo "Allocated CPUs: ${SLURM_CPUS_PER_TASK}"
'

source /opt/slurm-tools/slurm-postload.sh
```

---

# 9. Health Monitoring Alias

Alias:

```bash
hpcmon
```

Create/update alias:

```bash
sudo tee /etc/profile.d/capac-hpc-aliases.sh >/dev/null <<'EOF'
alias hpcmon='echo "=== SLURM SERVICES ==="; systemctl is-active slurmctld slurmd slurmdbd 2>/dev/null || true; echo; echo "=== SLURM STATUS ==="; scontrol ping 2>/dev/null || true; sacctmgr show cluster 2>/dev/null || true; echo; echo "=== CRITICAL ERRORS LAST 30 MIN ==="; journalctl --since "30 minutes ago" -p err..alert -u slurmctld -u slurmd -u slurmdbd --no-pager'
EOF
source /etc/profile.d/capac-hpc-aliases.sh
```

Run:

```bash
hpcmon
```

Master expected:

```text
slurmctld active
slurmd active
slurmdbd active
controller UP
no critical errors
```

Compute expected:

```text
slurmctld inactive
slurmd active
slurmdbd inactive
controller UP
no critical errors
```

---

# 10. Slurm Operations

Show nodes:

```bash
sinfo -Nel
```

Show partition:

```bash
sinfo
```

Show queue:

```bash
squeue
```

Submit job:

```bash
sbatch <jobfile.sbatch>
```

Cancel job:

```bash
scancel <jobid>
```

Show accounting:

```bash
sacct --format=JobID,JobName,Partition,AllocCPUS,State,Elapsed,NodeList
```

Show specific job:

```bash
sacct -j <jobid> --format=JobID,JobName,Partition,AllocCPUS,State,Elapsed,NodeList
```

---

# 11. Standard Test Jobs

Hostname test:

```bash
sbatch /opt/slurm-tests/hostname-test.sbatch
```

CPU test:

```bash
sbatch /opt/slurm-tests/cpu-test.sbatch
```

80-core 10-minute stress test:

```bash
sbatch /opt/slurm-tests/80core-node-stress-10min.sbatch
```

Monitor:

```bash
watch -n 5 'squeue; echo; sinfo -Nel'
```

Accounting check:

```bash
sacct -j <jobid> --format=JobID,JobName,Partition,AllocCPUS,State,Elapsed,NodeList
```

Expected:

```text
AllocCPUS 80
State COMPLETED
NodeList cae-[01,03]
```

---

# 12. User Management

Add new HPC user:

```bash
sudo capac-add-hpc-user <username>
```

Example:

```bash
sudo capac-add-hpc-user analyst1
```

User must logout/login once to inherit group membership.

Validate:

```bash
id analyst1
sudo -u analyst1 touch /home/data/scratch/analyst1-test.txt
sudo -u analyst1 sbatch /opt/slurm-tests/cpu-test.sbatch
```

Expected:

```text
User is member of hpc
User can write to scratch
User can submit Slurm jobs
```

---

# 13. Reboot SOP

Recommended order:

```text
1. Check/cancel jobs
2. Reboot compute node cae-03 first
3. Validate cae-03
4. Reboot master node cae-01
5. Validate full cluster from cae-01
```

Check jobs:

```bash
squeue
```

Reboot compute:

```bash
sudo reboot
```

Validate compute:

```bash
munge -n | unmunge && systemctl is-active munge slurmd
```

Reboot master:

```bash
sudo reboot
```

Validate master:

```bash
munge -n | unmunge && \
systemctl is-active mariadb slurmdbd slurmctld slurmd && \
scontrol ping && \
sinfo -Nel
```

---

# 14. Git Operations

Commit all bootstrap changes:

```bash
git add bootstrap modules slurm-tools docs && \
git commit -m "Update CAPAC Rocky bootstrap modules and SOP" && \
git push origin main
```

Pull latest on node:

```bash
cd /opt/CAPAC-Rocky810
sudo git pull origin main
```

---

# 15. Troubleshooting

## Munge socket missing

Error:

```text
Munge encode failed: Failed to access "/var/run/munge/munge.socket.2"
```

Fix:

```bash
sudo systemctl stop munge
sudo rm -f /run/munge/munge.socket.2 /run/munge/munged.pid
sudo rm -f /var/run/munge/munge.socket.2 /var/run/munge/munged.pid
sudo chown -R munge:munge /etc/munge /var/lib/munge /var/log/munge /run/munge
sudo chmod 0700 /etc/munge
sudo chmod 0711 /var/lib/munge
sudo chmod 0700 /var/log/munge
sudo chmod 0755 /run/munge
sudo chmod 0400 /etc/munge/munge.key
sudo systemctl restart munge
munge -n | unmunge
```

## Slurm controller down

```bash
sudo systemctl restart munge mariadb slurmdbd slurmctld slurmd
scontrol ping
```

## Compute node not visible

On compute:

```bash
sudo systemctl restart munge slurmd
munge -n | unmunge
```

On master:

```bash
sinfo -Nel
scontrol update NodeName=cae-03 State=RESUME
```

## Logs

```bash
journalctl -u munge -n 100 --no-pager
journalctl -u slurmctld -n 100 --no-pager
journalctl -u slurmdbd -n 100 --no-pager
journalctl -u slurmd -n 100 --no-pager
```

---

# 16. Current Stable Baseline

Verified state:

```text
Master cae-01:
  slurmctld active
  slurmd active
  slurmdbd active
  accounting active
  controller UP

Compute cae-03:
  slurmd active
  controller reachable
  no critical Slurm errors

Cluster:
  cae-01 idle
  cae-03 idle
  cfd partition active
  80-core job verified under Slurm
```
