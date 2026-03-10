Here is a complete, clean Markdown guide that you can copy and paste directly into a `README.md` file for your GitHub repository. It is specifically tailored for George Mason University's Hopper cluster, incorporating all the exact module dependencies and configuration patches required to get the GNU toolchain working smoothly.

---

# Compiling WRF and WPS on GMU's Hopper HPC

This guide details the steps required to compile the Weather Research and Forecasting (WRF) model and the WRF Preprocessing System (WPS) on George Mason University's Hopper High-Performance Computing cluster. It utilizes the default GNU compiler and OpenMPI toolchain.

## 0. Request an Interactive Session

Compiling these models is resource-intensive and should never be done on the login nodes. Start by requesting an interactive compute node:

```bash
salloc --partition=interactive --time=02:00:00 --nodes=1 --ntasks=4

```

## Part 1: Compiling WRF (em_real)

### 1. Load Required Modules

Hopper uses a hierarchical module system (Lmod). You must load the compiler and MPI libraries before the NetCDF libraries become available. We also load `autotools` to provide the `m4` macro processor required by WRF.

```bash
module purge
module load gnu9
module load openmpi4
module load netcdf-c
module load netcdf-fortran
module load autotools

```

### 2. Set Environment Variables

WRF requires strict paths for NetCDF. We also enable large file support, which is critical for high-resolution simulations to bypass the 2GB file size limit.

```bash
export NETCDF=$(dirname $(dirname $(which nc-config)))
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export NETCDF_classic=1

```

### 3. Clone and Configure

Clone the repository and run the configuration script:

```bash
git clone https://github.com/wrf-model/WRF.git
cd WRF
./configure

```

When prompted, select:

* **Compiler/MPI:** Option for **GNU (gfortran/gcc)** and **dmpar** (Distributed Memory Parallelism). *Usually Option 34.*
* **Nesting:** **Option 1** (basic).

### 4. Patch the Configuration (Crucial Hopper Fix)

By default, WRF prepends the `time` command to the Fortran compiler. Because `/usr/bin/time` is not installed on Hopper's stripped-down interactive nodes, the build will immediately fail. Run this `sed` command to strip `time` out of the configuration file:

```bash
sed -i 's/time mpif90/mpif90/g' configure.wrf

```

### 5. Compile WRF

Compile the `em_real` case using the 4 cores allocated in your interactive session:

```bash
./compile -j 4 em_real >& compile_em_real.log

```

*Verification:* Check the `main/` directory for `wrf.exe` and `real.exe`, or ensure the last line of your log reads `Executables successfully built`.

---

## Part 2: Compiling WPS

WPS must be compiled in the same environment and directory level as WRF, as it links directly to the compiled WRF libraries.

### 1. Load Additional Modules

WPS requires image compression libraries to decode GRIB2 meteorological files. Load these on top of your existing WRF modules:

```bash
module load jasper
module load libpng
module load zlib

```

### 2. Set WPS Environment Variables

WPS requires specific Jasper variables. We map Hopper's `$JASPER_DIR` to the variables the configure script expects:

```bash
export JASPERLIB=${JASPER_DIR}/lib
export JASPERINC=${JASPER_DIR}/include

```

### 3. Clone and Configure

Navigate back to your parent directory, clone WPS, and configure it:

```bash
cd ..
git clone https://github.com/wrf-model/WPS.git
cd WPS
./configure

```

When prompted, select:

* **Option 3:** `Linux x86_64, gfortran (dmpar)`

### 4. Compile WPS

Compile the preprocessing executables:

```bash
./compile >& compile_wps.log

```

*Verification:* Run `ls -l *.exe`. You should see `geogrid.exe`, `ungrib.exe`, and `metgrid.exe` successfully linked in the directory.

---

### Note on Interactive Node Testing

If you test run `wrf.exe` on a single Hopper interactive node via `mpirun`, you may encounter an immediate OpenMPI segmentation fault related to UCX and the `mlx5` InfiniBand driver. To bypass the network driver and force the node to use shared memory for local testing, run:

```bash
mpirun --mca pml ob1 --mca btl vader,self -np 4 ./wrf.exe

```

*(Note: These flags are only for single-node interactive testing. Standard Slurm batch submissions on Hopper's compute partitions do not require them).*