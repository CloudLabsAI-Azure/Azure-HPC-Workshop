## Exercise 6: Run OpenFOAM DrivAer-Fastback simulation using EESSI stack

The Az-HOP deployment in this tutorial comes with the [EESSI software stack](https://eessi.github.io/docs) pre-configured on all compute nodes.
In this exercise you will run and analyze the DrivAer-Fastback CFD simulation with 3 million cells without having to build OpenFOAM by using `OpenFOAM/9-foss-2021a` available in EESSI.

### Task 1: Prepare the DrivAer-Fastback example

1. On the lab computer, in the browser window displaying the Code Server, in the **Terminal** pane, at the **[clusteradmin@hb120v2-1 clusteradmin]$**  prompt, run the following command to Clone the OpenFOAM-9 repository from GitHub:

   ```bash
   git clone https://github.com/OpenFOAM/OpenFOAM-9.git
   ```

2. Copy the DrivAer-Fastback tutorial to your home directory:

   ```bash
   cp -r OpenFOAM-9/tutorials/incompressible/simpleFoam/drivaerFastback ~
   ```

3. Apply the following changes to the default tutorial:

    - Add `FOAM_MPIRUN_FLAGS` to the `mpirun` command when using `runParallel` (needed for all version of OpenFOAM)
    - Decompress the geometry
    - Rename `constant/geometry` to 'constant/triSurface`
    - Reconstruct the single partition after the solve

      Here are the commands:

      ```bash
      cd ~/drivaerFastback
      sed -i '/RunFunctions/a source <(declare -f runParallel | sed "s/mpirun/mpirun \\\$FOAM_MPIRUN_FLAGS/g")' Allrun
      sed -i 's#/bin/sh#/bin/bash#g' Allrun
      gunzip constant/geometry/*
      mv constant/geometry constant/triSurface
      sed -i 's/# runApplication reconstructPar/runApplication reconstructPar/g' Allrun
      ```

### Task 2: Submit PBS job

1. Create a PBS submit script as `~/drivaerFastback/submit.sh`. When running on multiple nodes it is necessary to export all the OpenFOAM environment variables (unless you add loading the modules in `.bashrc`). This is done with the `FOAM_MPIRUN_FLAGS` that we added to the `runParallel` in the previous step. The script will run for the number of cores specified to PBS (`select` x `mpiprocs`):

   ```bash
   #!/bin/bash

   source /cvmfs/pilot.eessi-hpc.org/versions/2021.12/init/bash
   ml OpenFOAM/9-foss-2021a
   source $FOAM_BASH
   $PBS_O_WORKDIR/Allclean

   ranks_per_numa=30
   export FOAM_MPIRUN_FLAGS="-mca pml ucx -hostfile $PBS_NODEFILE $(env |grep 'WM_\|FOAM_' | cut -d'=' -f1 | sed 's/^/-x /g' | tr '\n' ' ') -x MPI_BUFFER_SIZE -x UCX_POSIX_USE_PROC_LINK=n -x PATH --map-by ppr:${ranks_per_numa}:numa"
   $PBS_O_WORKDIR/Allrun -m M -cores $(wc -l <$PBS_NODEFILE)
   ```

2. Submit the OpenFOAM batch job, requesting our job to be exclusively allocated to two HB120rs_v2 nodes from the node array `hb120v2`:

   ```bash
   cd ~/drivaerFastback
   qsub -l select=2:slot_type=hb120v2:ncpus=120:mpiprocs=120,place=scatter:excl submit.sh
   ```

3. Monitor the node allocation in CycleCloud and the job until its completion by using the `qstat` command. Log files will be generated in the `~/drivaerFastback` with the pattern `log.*`. Expect the job to run for about 12 minutes.

### Task 3: Create Linux Desktop session for visualization

1. On the lab computer, in the browser window, switch back to the **Azure HPC On-Demand Platform** portal, and then in the **Interactive Apps** section, select **Linux Desktop**.

   ![alt](image/EX5-Task4-Step1.png)

2. On the **Linux Desktop** launching page, from the **Session target** drop-down list, ensure that **With GPU - Small GPU node for single session** entry is selected. In the **Maximum duration of your remote session** field, enter **1**,  and then select **Launch**.

   ![alt](image/EX2-Task5-Step12.png)

   > **Note**: This will begin compute node provisioning of the type you specified. This also creates a new job with its **Queued** status displaying on the same page.

3. Switch back to the **Linux Desktop** launching page, and then verify that the corresponding job's status has changed to **Running**.

4. Adjust **Compression** and **Image quality** according to your preferences, and then select **Launch Linux Desktop**.

   ![alt](image/EX2-Task5-Step17.png)

   > **Note**: This will open another browser tab displaying the Linux Desktop session.

5. Open a **Terminal Emulator** in the Linux Desktop.

      ![alt](image/EX3-Task5-Step18a.png)

6. Install the **ParaView** viewer:

   ```bash
   wget "https://www.paraview.org/paraview-downloads/download.php?submit=Download&version=v5.10&type=binary&os=Linux&downloadFile=ParaView-5.10.1-MPI-Linux-Python3.9-x86_64.tar.gz" -O ParaView-5.10.1-MPI-Linux-Python3.9-x86_64.tar.gz
   tar xvf ParaView-5.10.1-MPI-Linux-Python3.9-x86_64.tar.gz
   ```

### Task 4: Visualize the DrivAer-Fastback simulation results

1. Create a case file and launch ParaView:

   ```bash
   touch ~/drivaerFastback/case.foam
   vglrun ./ParaView-5.10.1-MPI-Linux-Python3.9-x86_64/bin/paraview
   ```

2. Within **ParaView** open the case `/drivaerFastback/case.foam`. From the tool bar click on **open file**.

   ![alt](image/EX6-Task4-task2a.png)

3. Click on **drivaerFastback**.

   ![alt](image/EX6-Task4-task2b.png)

4. Select **case.foam** and click on **OK**.  

   ![alt](image/EX6-Task4-task2c.png)

5. When the model is loaded, you can load the car geometry as follows:
   - In the bottom left pane, in the "Mesh Regions" list, unselect "internalMesh" and select the    following fields:
      - `patch/body`
      - `patch/frontWheels`
      - `patch/ground`
      - `patch/inlet`
      - `patch/rearWheels`
   - Click "Apply" above the list

      ![alt](image/EX6-Task4-task3a.png)

   - You should now see the model geometry, and you can move/rotate/zoom using the mouse

      ![alt](image/EX6-Task4-task3b.png)

6. To visualize the simulation results, click the "Play" button on the toolbar at the top of the window to advance to the end of the simulation.

   ![alt](image/EX6-Task4-task3c.png)

7. On the Active Variables Control toolbar you will find a drop down box where you can select variables. For example, select "p" for pressure.

   ![alt](image/EX6-Task4-task3d.png)

