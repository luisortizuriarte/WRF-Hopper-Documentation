

# WRF Compilation and Execution Guide for Hopper (AMD Architecture)

This guide outlines the process for compiling WRF v4.x using the GNU 12 compiler suite on Hopper. 

## Environment Preparation

To ensure the compiler links the correct AMD-specific instructions (e.g., `zen2`, `zen3`, or `x86_64_v3`), the compilation must be performed on an AMD node.

**1. Access the AMD Hardware**
Log into the dedicated AMD login node or request an interactive session on the AMD compute partition:

```bash
salloc --partition=interactive --constraint=amd --time=02:00:00 --nodes=1 --ntasks=4

```

**2. Load the Software Stack**

```bash
module purge
module load gnu12 openmpi4 netcdf-c netcdf-fortran autotools m4

```

## NetCDF Symlink Workaround

Modern Spack package managers strictly isolate C and Fortran libraries into separate installation directories. The WRF configuration script frequently fails to locate the Fortran headers (like `netcdf.inc`) when pointed to a split installation. To bypass this, create a unified directory structure in your project space.

**1. Capture the Module Paths**

```bash
NC_C=$(dirname $(dirname $(which nc-config)))
NC_F=$(dirname $(dirname $(which nf-config)))

```

**2. Create a Unified Directory**
Create a master directory to house the symlinks. *(Replace `$USER` with your NetID if running outside of your own environment).*

```bash
mkdir -p /projects/$USER/wrf_netcdf/include
mkdir -p /projects/$USER/wrf_netcdf/lib

```

**3. Symlink the C and Fortran Libraries**

```bash
ln -sf $NC_C/include/* /projects/$USER/wrf_netcdf/include/
ln -sf $NC_C/lib/* /projects/$USER/wrf_netcdf/lib/
ln -sf $NC_C/lib64/* /projects/$USER/wrf_netcdf/lib/ 2>/dev/null

ln -sf $NC_F/include/* /projects/$USER/wrf_netcdf/include/
ln -sf $NC_F/lib/* /projects/$USER/wrf_netcdf/lib/
ln -sf $NC_F/lib64/* /projects/$USER/wrf_netcdf/lib/ 2>/dev/null

```

## Configuration and Patching

**1. Set Environment Variables**
Direct WRF to the newly created unified directory and enable large file support.

```bash
export NETCDF=/projects/$USER/wrf_netcdf
unset NETCDFF
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export NETCDF_classic=1

```

**2. Configure WRF**
Navigate to the WRF source directory, clean any previous builds, and generate the configuration file.

```bash
cd ~/WRF
./clean -a
./configure

```

* **Compiler Selection:** Enter `34` for GNU (gfortran/gcc) with distributed memory (`dmpar`).
* **Nesting Selection:** Enter `1` for basic nesting.

**3. Patch the 'time' Wrapper Bug**
Hopper's compute nodes often fail to resolve the `time` wrapper injected by the `gnu12` configuration script, which completely halts the compilation of Fortran files. Strip this command out of the generated file:

```bash
sed -i 's/time $(DM_FC)/$(DM_FC)/g' configure.wrf

```

## Compilation and Verification

**1. Build the Executables**
Compile the real-data case using the cores allocated in your interactive session to accelerate the process.

```bash
./compile -j 4 em_real >& compile_em_real.log

```

**2. Verify the Build**
Confirm that `wrf.exe`, `real.exe`, `ndown.exe`, and `tc.exe` were successfully generated in the `main/` directory. Check the dynamic linking to guarantee the executable is pointing to the `x86_64_v3` architecture rather than `cascadelake`.

```bash
ldd main/wrf.exe | grep netcdf

```

---

## Slurm Job Submission

To execute the newly compiled binary, the Slurm submission script must explicitly mandate the AMD partition. If submitted without the hardware constraint, the scheduler may route the job to an Intel node, resulting in immediate failure.

To maximize parallel efficiency and avoid over-decomposition on high-resolution urban canopy grids, align the core request with the physical architecture of Hopper's AMD nodes (64 cores, 256 GB RAM per node).

### Sample Submission Script (`submit_wrf.sh`)

```bash
#!/bin/bash
#SBATCH --job-name=wrf_run
#SBATCH --partition=normal
#SBATCH --constraint=amd         # CRITICAL: Forces execution on AMD nodes
#SBATCH --nodes=1                # Scales appropriately for most 500m domains
#SBATCH --ntasks=64              # Utilizes exactly one full AMD node
#SBATCH --mem=0                  # Claims all 256GB of available node memory
#SBATCH --time=24:00:00          # Adjust based on simulation length
#SBATCH --output=wrf_run_%j.log
#SBATCH --error=wrf_run_%j.err

# 1. Load the exact environment used during compilation
module purge
module load gnu12 openmpi4 netcdf-c netcdf-fortran

# 2. Navigate to the execution directory
cd /projects/$USER/your_wrf_run_directory

# 3. Execute WRF utilizing the InfiniBand network
mpirun ./wrf.exe

```