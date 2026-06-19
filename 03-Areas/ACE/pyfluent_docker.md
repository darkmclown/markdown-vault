# PyFluent Docker Setup Runbook

Simple notes and essential commands for setting up a custom Fluent Docker image and running a PyFluent example from bash.

Environment used in this setup:

| Item | Value |
|---|---|
| Host | `cae-03` |
| User | `cadfem` |
| Parent folder | `/home/data/pyfluent-docker` |
| Ansys install | `/home/data/ansys_inc/v261` |
| Ansys parent path | `/home/data/ansys_inc` |
| Fluent version | `261` |
| Docker image | `ansys_inc:fluent_261` |
| License server | `1055@ansyslicenseserver` |
| Ansys Licensing Interconnect | `2325@ansyslicenseserver` |
| Python path | `/home/data/pyfluent-docker/python` |

---

## Step 1 — Create base folder

```bash
export PARENT="/home/data/pyfluent-docker"

mkdir -p "$PARENT"
mkdir -p "$PARENT/src"
mkdir -p "$PARENT/docker-data"
mkdir -p "$PARENT/containerd"
mkdir -p "$PARENT/docker-tmp"
mkdir -p "$PARENT/fluent-test"
mkdir -p "$PARENT/vortex-bash-job"

sudo chown -R cadfem:cadfem "$PARENT"
```

Note:

- Keep all project files under `/home/data/pyfluent-docker`.
- Docker internal data can also be placed here if required.
- Ansys installation remains under `/home/data/ansys_inc/v261`.

---

## Step 2 — Configure Docker data path

Create or update Docker daemon config.

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "data-root": "/home/data/pyfluent-docker/docker-data"
}
EOF

sudo systemctl restart docker
docker info | grep "Docker Root Dir"
```

Expected:

```text
Docker Root Dir: /home/data/pyfluent-docker/docker-data
```

---

## Step 3 — Install required OS packages

For Rocky Linux 8.10:

```bash
sudo dnf install -y \
  git wget curl tar make gcc gcc-c++ \
  zlib-devel bzip2-devel openssl-devel \
  libffi-devel readline-devel sqlite-devel \
  xz-devel ncurses-devel tk-devel \
  findutils rsync patch \
  nmap-ncat
```

Note:

- `nmap-ncat` gives the `nc` command used for port testing.
- Python 3.10+ is required because the PyFluent Docker helper script uses modern Python syntax.

---

## Step 4 — Clone PyFluent source

```bash
cd "$PARENT/src"

git clone https://github.com/ansys/pyfluent.git pyfluent
```

Set common variables:

```bash
export ANSYS_INC="/home/data/ansys_inc"
export FLUENT_VER="261"
export FLUENT_DOCKER_DIR="$PARENT/src/pyfluent/docker/fluent_${FLUENT_VER}"
```

Check folder:

```bash
ls -lh "$FLUENT_DOCKER_DIR"
```

---

## Step 5 — Compile Python under project path

The system Python was not enough for the PyFluent Docker helper. Build Python inside the project folder.

```bash
export PY_VER="3.14.6"
export PY_PREFIX="$PARENT/python/$PY_VER"

mkdir -p "$PARENT/build"
cd "$PARENT/build"

wget https://www.python.org/ftp/python/${PY_VER}/Python-${PY_VER}.tgz
tar -xzf Python-${PY_VER}.tgz
cd Python-${PY_VER}

./configure \
  --prefix="$PY_PREFIX" \
  --enable-optimizations \
  --with-ensurepip=install

make -j"$(nproc)"
make install
```

Create stable symlinks:

```bash
mkdir -p "$PARENT/python/bin"

ln -sfn "$PY_PREFIX/bin/python3" "$PARENT/python/bin/python3"
ln -sfn "$PY_PREFIX/bin/pip3" "$PARENT/python/bin/pip3"

"$PARENT/python/bin/python3" --version
```

Expected:

```text
Python 3.14.6
```

Add Python to PATH:

```bash
cat >> ~/.bashrc <<'EOF'

# PyFluent Docker Python
export PATH="/home/data/pyfluent-docker/python/bin:/home/data/pyfluent-docker/python/3.14.6/bin:$PATH"
EOF

source ~/.bashrc
```

---

## Step 6 — Install PyFluent dependencies

```bash
export PYTHON="$PARENT/python/bin/python3"

"$PYTHON" -m ensurepip --upgrade
"$PYTHON" -m pip install --upgrade pip setuptools wheel

"$PYTHON" -m pip install \
  ansys-fluent-core \
  imageio
```

Verify:

```bash
"$PYTHON" - <<'PY'
import sys
print(sys.version)

import ansys.fluent.core as pyfluent
print("PyFluent import OK")
print(pyfluent.__version__)

import imageio
print("imageio import OK")
PY
```

Note:

- If a warning appears about `imageio_download_bin` not being on PATH, keep the Python paths from Step 5 in `.bashrc`.
- The important check is that `ansys.fluent.core` imports correctly.

---

## Step 7 — Copy Ansys files into Docker build context

The PyFluent repo provides a helper to copy only required Ansys files into the Fluent Docker build folder.

```bash
export PARENT="/home/data/pyfluent-docker"
export ANSYS_INC="/home/data/ansys_inc"
export FLUENT_VER="261"
export FLUENT_DOCKER_DIR="$PARENT/src/pyfluent/docker/fluent_${FLUENT_VER}"
export PYTHON="$PARENT/python/bin/python3"

cd "$PARENT/src/pyfluent/docker"

sudo rm -rf "$FLUENT_DOCKER_DIR/ansys_inc"
sudo chown -R cadfem:cadfem "$PARENT/src"
chmod -R u+rwX "$PARENT/src"
sudo chmod -R a+rX "$ANSYS_INC"

"$PYTHON" copy_ansys_files.py "$ANSYS_INC" "$FLUENT_DOCKER_DIR"
```

Check copied files:

```bash
ls -ld "$FLUENT_DOCKER_DIR/ansys_inc"
du -sh "$FLUENT_DOCKER_DIR/ansys_inc"
```

If permission errors appear during copy:

```bash
sudo "$PYTHON" copy_ansys_files.py "$ANSYS_INC" "$FLUENT_DOCKER_DIR"
sudo chown -R cadfem:cadfem "$FLUENT_DOCKER_DIR/ansys_inc"
```

---

## Step 8 — Patch Dockerfile package names

Some package names in the default Dockerfile may not resolve correctly on Rocky Linux 8.10.

```bash
cd "$FLUENT_DOCKER_DIR"

cp Dockerfile Dockerfile.bak.$(date +%Y%m%d-%H%M%S)

sed -i 's/libXcomposite1/libXcomposite.x86_64/g' Dockerfile
sed -i '/libnss3/d' Dockerfile
sed -i 's/nss-3.90.0-7.el8_10.x86_64/nss.x86_64/g' Dockerfile
sed -i 's/mesa-libGLU-9.0.0-15.el8.x86_64/mesa-libGLU.x86_64/g' Dockerfile
sed -i 's/libglvnd-glx-1.3.4-2.el8.x86_64/libglvnd-glx.x86_64/g' Dockerfile
sed -i 's/glx-utils-8.4.0-5.20181118git1830dcb.el8.x86_64/glx-utils.x86_64/g' Dockerfile
sed -i 's/libGLEW-2.0.0-6.el8.x86_64/libGLEW.x86_64/g' Dockerfile
```

Note:

- Avoid hard-pinned RPM versions unless the repo has exactly those builds.
- Keep package names flexible for Rocky 8.10 updates.

---

## Step 9 — Build Fluent Docker image

```bash
export TMPDIR="$PARENT/docker-tmp"

cd "$FLUENT_DOCKER_DIR"

DOCKER_BUILDKIT=0 docker build --no-cache -t ansys_inc:fluent_261 .
```

Check image:

```bash
docker images | grep ansys_inc
```

---

## Step 10 — Test Fluent image directly

Run Fluent interactively first.

```bash
export WORK_DIR="/home/data/pyfluent-docker/fluent-test"

docker run --rm -it \
  --name fluent-shm-test \
  --shm-size=4g \
  -e ANSYSLMD_LICENSE_FILE="1055@ansyslicenseserver" \
  -e ANSYSLI_SERVERS="2325@ansyslicenseserver" \
  -v "$WORK_DIR":/home/container/workdir \
  -w /home/container/workdir \
  ansys_inc:fluent_261 \
  3ddp -gu -t2
```

Expected:

- Fluent starts.
- License checkout works.
- No MPI bus error.

Important note:

- Without `--shm-size=4g`, Fluent MPI may fail with `BAD TERMINATION` or bus error.
- For serial runs, `--shm-size=4g` is still safe.

---

## Step 11 — Basic batch smoke test

Create simple journal:

```bash
cd /home/data/pyfluent-docker/fluent-test

cat > fluent_batch_smoke.jou <<'EOF'
(display "Fluent Docker batch smoke test started\n")

(with-output-to-file "docker_batch_smoke.txt"
  (lambda ()
    (display "Fluent Docker batch smoke test PASSED\n")
  )
)

(display "Fluent Docker batch smoke test completed\n")
(exit)
EOF
```

Run:

```bash
docker run --rm -it \
  --name fluent-batch-smoke \
  --shm-size=4g \
  -e ANSYSLMD_LICENSE_FILE="1055@ansyslicenseserver" \
  -e ANSYSLI_SERVERS="2325@ansyslicenseserver" \
  -v /home/data/pyfluent-docker/fluent-test:/home/container/workdir \
  -w /home/container/workdir \
  ansys_inc:fluent_261 \
  3ddp -gu -t2 -i /home/container/workdir/fluent_batch_smoke.jou
```

Verify:

```bash
cat /home/data/pyfluent-docker/fluent-test/docker_batch_smoke.txt
```

Expected:

```text
Fluent Docker batch smoke test PASSED
```

Note:

- Fluent TUI commands and Scheme commands are different.
- `(file/write-case "file.cas.h5")` is not valid Scheme and gives `unbound variable`.

---

## Step 12 — Python inside Docker: important path note

Do not copy the compiled Python to a different path such as `/opt/pyfluent-python`.

This failed because Python was compiled with this prefix:

```text
/home/data/pyfluent-docker/python/3.14.6
```

When copied to `/opt`, Python could not find the standard library and failed with:

```text
Fatal Python error: Failed to import encodings module
ModuleNotFoundError: No module named 'encodings'
```

Use one of these two safe options.

---

## Step 13A — Safe option: mount Python at the same path

This is the simplest and safest approach.

```bash
docker run --rm \
  -v /home/data/pyfluent-docker/python:/home/data/pyfluent-docker/python:ro \
  --entrypoint /bin/bash \
  ansys_inc:fluent_261 \
  -lc '/home/data/pyfluent-docker/python/bin/python3 - <<PY
import sys
print(sys.version)

import encodings
print("encodings OK")

import ansys.fluent.core as pyfluent
print("PyFluent import OK")
print(pyfluent.__version__)
PY'
```

Expected:

```text
encodings OK
PyFluent import OK
```

---

## Step 13B — Self-contained option: copy Python into image at the same absolute path

Use this if the image must carry Python inside it.

```bash
export PARENT="/home/data/pyfluent-docker"
export RUNTIME_IMG_DIR="$PARENT/pyfluent-runtime-image"

rm -rf "$RUNTIME_IMG_DIR"
mkdir -p "$RUNTIME_IMG_DIR"

rsync -a "$PARENT/python/" "$RUNTIME_IMG_DIR/python/"

cat > "$RUNTIME_IMG_DIR/Dockerfile" <<'EOF'
FROM ansys_inc:fluent_261

USER root

COPY python /home/data/pyfluent-docker/python

ENV PATH="/home/data/pyfluent-docker/python/bin:/home/data/pyfluent-docker/python/3.14.6/bin:${PATH}"
ENV PYTHONUNBUFFERED=1
ENV PYFLUENT_FLUENT_ROOT="/ansys_inc/v261/fluent"

EXPOSE 50055

WORKDIR /home/container/workdir
EOF

docker build --no-cache -t ansys_inc:fluent_261_pyfluent "$RUNTIME_IMG_DIR"
```

Verify Python inside the image:

```bash
docker run --rm \
  --entrypoint /bin/bash \
  ansys_inc:fluent_261_pyfluent \
  -lc '/home/data/pyfluent-docker/python/bin/python3 - <<PY
import sys
print(sys.version)

import encodings
print("encodings OK")

import ansys.fluent.core as pyfluent
print("PyFluent import OK")
print(pyfluent.__version__)
PY'
```

Expected:

```text
encodings OK
PyFluent import OK
```

Note:

- If you copy Python into the image, copy it to `/home/data/pyfluent-docker/python`.
- Do not relocate it to `/opt`.

---

## Step 14 — Create PyFluent vortex job folder

```bash
export JOB_DIR="/home/data/pyfluent-docker/vortex-bash-job"

mkdir -p "$JOB_DIR"
sudo chown -R cadfem:cadfem "$JOB_DIR"
cd "$JOB_DIR"
```

---

## Step 15 — Create headless PyFluent vortex Python file

This is a headless version of the steady vortex example. It skips graphics and GIF generation for stable Docker execution.

```bash
cat > /home/data/pyfluent-docker/vortex-bash-job/vortex_pyfluent_headless.py <<'EOF'
import os
from pathlib import Path

import ansys.fluent.core as pyfluent
from ansys.fluent.core import Dimension, FluentMode, Precision
from ansys.fluent.core.examples import download_file
from ansys.fluent.core.solver import (
    CellRegister,
    CellZoneCondition,
    General,
    Initialization,
    Materials,
    Methods,
    Models,
    NamedExpression,
    RunCalculation,
    WallBoundary,
)

WORK_DIR = Path("/home/container/workdir")
WORK_DIR.mkdir(parents=True, exist_ok=True)
os.chdir(WORK_DIR)

FLUENT_CORES = int(os.environ.get("FLUENT_CORES", "1"))

os.environ["ANSYSLMD_LICENSE_FILE"] = "1055@ansyslicenseserver"
os.environ["ANSYSLI_SERVERS"] = "2325@ansyslicenseserver"

print(f"Launching Fluent from PyFluent inside Docker with {FLUENT_CORES} core(s)...")

solver_session = pyfluent.launch_fluent(
    mode=FluentMode.SOLVER,
    dimension=Dimension.THREE,
    precision=Precision.DOUBLE,
    processor_count=FLUENT_CORES,
    ui_mode="no_gui",
    cleanup_on_exit=True,
    cwd=str(WORK_DIR),
    start_timeout=600,
)

try:
    print("Fluent version:", solver_session.get_fluent_version())

    print("Downloading vortex mesh...")
    vortex_mesh = download_file(
        "vortex-mixingtank.msh.h5",
        "pyfluent/examples/Steady-Vortex-VOF",
        save_path=str(WORK_DIR),
    )

    mesh_file_name = Path(vortex_mesh).name

    print("Reading mesh:", mesh_file_name)
    solver_session.settings.file.read_case(file_name=mesh_file_name)

    g = 9.81

    print("Setting gravity...")
    general_settings = General(solver_session)
    general_settings.operating_conditions.gravity.enable = True
    general_settings.operating_conditions.gravity.components = [0.0, 0.0, -g]

    print("Copying water material...")
    materials = Materials(solver_session)
    materials.database.copy_by_name(type="fluid", name="water-liquid")

    print("Creating stirring speed expression...")
    stirring_speed = NamedExpression(solver_session, new_instance_name="stirring_speed")
    stirring_speed.definition = "240 [rev min^-1]"
    stirring_speed.input_parameter = True

    print("Configuring MRF zone...")
    fluid_cell_zone = CellZoneCondition(solver_session, name="mrf")
    fluid_cell_zone.reference_frame.frame_motion = True
    fluid_cell_zone.reference_frame.reference_frame_axis_origin = [0, 0, 0]
    fluid_cell_zone.reference_frame.reference_frame_axis_direction = [0, 0, 1]
    fluid_cell_zone.reference_frame.mrf_omega.value = "stirring_speed"

    print("Configuring rotating wall...")
    wall_boundary = WallBoundary(solver_session, name="shaft_tank")
    wall_boundary.momentum.wall_motion = "Moving Wall"
    wall_boundary.momentum.relative = False
    wall_boundary.momentum.rotating = True
    wall_boundary.momentum.rotation_axis_direction = [0, 0, 1]
    wall_boundary.momentum.rotation_speed = "stirring_speed"

    print("Enabling VOF model...")
    model_setup = Models(solver_session)
    model_setup.multiphase.model = "vof"
    model_setup.multiphase.vof_parameters.vof_formulation = "implicit"
    model_setup.multiphase.vof_parameters.vof_cutoff = 1e-06
    model_setup.multiphase.advanced_formulation.implicit_body_force = True
    model_setup.viscous.options.curvature_correction = True

    print("Applying multiphase numerics...")
    solution_methods = Methods(solver_session)
    solution_methods.multiphase_numerics.solution_stabilization.execute_settings_optimization = True
    solution_methods.multiphase_numerics.solution_stabilization.execute_advanced_stabilization = True

    print("Changing phase names...")
    solver_session.tui.define.phases.set_domain_properties.change_phases_names(
        "water",
        "air",
    )

    general_settings.solver.time = "steady"

    print("Initializing solution...")
    solution_initialization = Initialization(solver_session)
    solution_initialization.reference_frame = "absolute"
    solution_initialization.defaults["k"] = 0.001
    solution_initialization.localized_turb_init.enabled = False

    print("Creating liquid patch cell register...")
    solver_session.settings.solution.cell_registers.create(name="liquid_patch")

    cell_register = CellRegister(solver_session, name="liquid_patch")
    cell_register.type = {
        "option": "hexahedron",
        "hexahedron": {
            "inside": True,
            "max_point": [100.0, 100.0, 0.19],
            "min_point": [-100.0, -100.0, -100.0],
        },
    }

    solution_initialization.initialize()

    print("Patching water volume fraction...")
    solution_initialization.patch.calculate_patch(
        domain="water",
        cell_zones=[],
        registers=["liquid_patch"],
        variable="mp",
        reference_frame="Relative to Cell Zone",
        use_custom_field_function=False,
        value=1,
    )

    print("Writing initial case/data...")
    solver_session.settings.file.write_case_data(file_name="vortex_init.cas.h5")

    print("Running 50 iterations...")
    run_calculation = RunCalculation(solver_session)
    run_calculation.iter_count = 50
    run_calculation.calculate()

    print("Writing final case/data...")
    solver_session.settings.file.write_case_data(file_name="vortex_final.cas.h5")

    Path("vortex_run_status.txt").write_text(
        "PyFluent vortex bash run PASSED\n"
        f"Cores: {FLUENT_CORES}\n"
        "Iterations: 50\n"
        "Files: vortex_init.cas.h5, vortex_final.cas.h5\n"
    )

    print("PyFluent vortex bash run PASSED.")

finally:
    solver_session.exit()
EOF
```

---

## Step 16 — Create serial bash launcher

This runs the PyFluent example from bash, with Python mounted into the Fluent container.

```bash
cat > /home/data/pyfluent-docker/vortex-bash-job/run_vortex_pyfluent_bash.sh <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

JOB_DIR="/home/data/pyfluent-docker/vortex-bash-job"
PYTHON_DIR="/home/data/pyfluent-docker/python"
IMAGE="ansys_inc:fluent_261"
CONTAINER_NAME="pyfluent-vortex-bash"

CORES="1"
SHM_SIZE="4g"

echo "Running Fluent in SERIAL mode"
echo "Fluent cores: ${CORES}"
echo "Docker shared memory: ${SHM_SIZE}"

docker rm -f "$CONTAINER_NAME" 2>/dev/null || true

docker run --rm -it \
  --name "$CONTAINER_NAME" \
  --shm-size="$SHM_SIZE" \
  -e ANSYSLMD_LICENSE_FILE="1055@ansyslicenseserver" \
  -e ANSYSLI_SERVERS="2325@ansyslicenseserver" \
  -e FLUENT_CORES="$CORES" \
  -v "$PYTHON_DIR":"$PYTHON_DIR":ro \
  -v "$JOB_DIR":/home/container/workdir \
  -w /home/container/workdir \
  --entrypoint /bin/bash \
  "$IMAGE" \
  -lc '
    set -Eeuo pipefail

    FLUENT_BIN="$(find / -path "*/fluent/bin/fluent" -type f 2>/dev/null | head -1)"

    if [ -z "$FLUENT_BIN" ]; then
      echo "ERROR: Fluent executable not found inside container"
      exit 1
    fi

    FLUENT_ROOT="$(dirname "$(dirname "$FLUENT_BIN")")"
    AWP_ROOT261="$(dirname "$FLUENT_ROOT")"

    export PATH="$(dirname "$FLUENT_BIN"):/home/data/pyfluent-docker/python/bin:/home/data/pyfluent-docker/python/3.14.6/bin:$PATH"
    export PYFLUENT_FLUENT_ROOT="$FLUENT_ROOT"
    export AWP_ROOT261="$AWP_ROOT261"
    export PYTHONUNBUFFERED=1

    echo "Using Fluent: $FLUENT_BIN"
    echo "Using PYFLUENT_FLUENT_ROOT: $PYFLUENT_FLUENT_ROOT"
    echo "Using AWP_ROOT261: $AWP_ROOT261"
    echo "Running with FLUENT_CORES=$FLUENT_CORES"

    /home/data/pyfluent-docker/python/bin/python3 /home/container/workdir/vortex_pyfluent_headless.py
  '
EOF

chmod +x /home/data/pyfluent-docker/vortex-bash-job/run_vortex_pyfluent_bash.sh
```

---

## Step 17 — Run the PyFluent job from bash

```bash
cd /home/data/pyfluent-docker/vortex-bash-job

./run_vortex_pyfluent_bash.sh
```

---

## Step 18 — Verify output files

```bash
cd /home/data/pyfluent-docker/vortex-bash-job

cat vortex_run_status.txt

ls -lh \
  vortex-mixingtank.msh.h5 \
  vortex_init.cas.h5 \
  vortex_init.dat.h5 \
  vortex_final.cas.h5 \
  vortex_final.dat.h5
```

Expected:

```text
PyFluent vortex bash run PASSED
Cores: 1
Iterations: 50
Files: vortex_init.cas.h5, vortex_final.cas.h5
```

---

## Notes from setup issues

### Python version issue

The PyFluent Docker helper required modern Python syntax. Building Python under the project path fixed this.

### Permission issue during Ansys copy

If `copy_ansys_files.py` fails with permission denied, remove the partial copied folder and rerun with corrected ownership.

### Docker package issue

Some RPM names in the Dockerfile were invalid for Rocky Linux 8.10. Patch them before building.

### Shared memory issue

Fluent MPI can fail without enough shared memory. Use:

```bash
--shm-size=4g
```

For larger parallel jobs, increase it.

### PyFluent host connection issue

Connecting from host Python to Fluent running inside Docker through `127.0.0.1:50055` can hit PyFluent transport handling errors. Avoid that path.

The stable approach is:

```text
Python runs inside the Docker container
PyFluent launches Fluent inside the same container
Results are written to the mounted work directory
```

### Python relocation issue

Do not move compiled Python to `/opt` unless Python was compiled for that prefix.

Bad path:

```text
/opt/pyfluent-python
```

Good path:

```text
/home/data/pyfluent-docker/python
```

---

## Final validation status

Once Step 18 passes:

```text
Fluent Docker image: OK
License checkout: OK
Python/PyFluent: OK
Docker mounted workdir: OK
PyFluent launches Fluent inside container: OK
Serial vortex run: OK
```
