# *Installing RoseTTAFold on an HPC Without Sudo Permissions* 
This package contains deep learning models and related scripts to run RoseTTAFold.  
This repository is the official implementation of RoseTTAFold: Accurate prediction of protein structures and interactions using a 3-track network. This guide will help you install RoseTTAFold on an HPC system without using sudo permissions or Docker, utilizing CUDA 11.8 and miniconda3. Follow each step carefully.

This installation guide builds upon the resources provided by the [RoseTTAFold Commons](https://github.com/RosettaCommons/RoseTTAFold).

## Prerequisites

1. Ensure you have an account on your HPC.

2. Request access to a GPU partition compatible with CUDA 11.8.

## Step 1: Log into your HPC System

Open terminal and do this command specific to your own HPC and username:
```
ssh <your_username>@<your_hpc_address>
```
Please remember to replace `<your_username>` with your actual HPC username and `<your_hpc_address>` with the address of your HPC.

## Step 2: Load the Necessary Modules

Before continuing we need to load some modules:
```
module load nvidia/cuda/11.8
module load miniconda3
module load gnu11
module load aria2
```

## Installation

1. Clone the package
```
git clone https://github.com/RosettaCommons/RoseTTAFold.git
cd RoseTTAFold
```

2. Create conda environment using `RoseTTAFold-linux.yml` file.
```
# create conda environment for RoseTTAFold, cuda11 is required for this step
conda env create -f RoseTTAFold-linux.yml
```

Then we need to activate the environment:

```
conda activate RoseTTAFold
```

3. Download Neural Network weights

Once we've activated our conda environment we can move on to downloading the network weights:
```
wget https://files.ipd.uw.edu/pub/RoseTTAFold/weights.tar.gz
tar xfz weights.tar.gz
```

4. Download and install third-party software.
```
./install_dependencies.sh
```

5. Download sequence and structure databases
```
# uniref30 [46G]
wget http://wwwuser.gwdg.de/~compbiol/uniclust/2020_06/UniRef30_2020_06_hhsuite.tar.gz
mkdir -p UniRef30_2020_06
tar xfz UniRef30_2020_06_hhsuite.tar.gz -C ./UniRef30_2020_06

# BFD [272G]
wget https://bfd.mmseqs.com/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt.tar.gz
mkdir -p bfd
tar xfz bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt.tar.gz -C ./bfd

# structure templates (including *_a3m.ffdata, *_a3m.ffindex) [over 100G]
wget https://files.ipd.uw.edu/pub/RoseTTAFold/pdb100_2021Mar03.tar.gz
tar xfz pdb100_2021Mar03.tar.gz
# for CASP14 benchmarks, we used this one: https://files.ipd.uw.edu/pub/RoseTTAFold/pdb100_2020Mar11.tar.gz
```

## Running RoseTTAFold
Before running RoseTTAFold, you need to make sure the output directory is correctly configured in the `run_e2e_ver.sh` script:

Update the `run_e2e_ver.sh` script to your own specific output folder. Replace `/path/to/your/output_directory/output_$BASENAME` with the path to your desired output directory. If you haven't created an output directory yet, do so now. It should follow the same structure:

Edit the working directory in the script, this should be on line 35: 
```
WDIR=$(realpath -s "/path/to/your/output_directory/output_$BASENAME")
```

The `output_$BASENAME` part ensures that your output folder will be named based on the input FASTA file. For instance, if your input file is called example2.fasta, the output folder will be named `output_example2`.

Make sure that the modules required at the start are loaded, now we want to make a SLURM script to run the RoseTTAFold. The directory this script should be in, should be in your RoseTTAFold folder. You can create your own with different specific components, but it should look something like this with your own specific fasta file and input directories:

1. Partition (`--partition`): Choose a partition optimized for GPU use (gpu).

2. Memory (`--mem`): Allocate sufficient memory (e.g., 100G) based on your inputs.

3. GPU Allocation (`--gpus`): Typically, 1 GPU is sufficient (--gpus=1).

4. CPU Threads (`--cpus-per-task`): Set to 1 if running a single-threaded job.

5. Output and Error Files: Customize file names to avoid overwriting (%j adds the job ID).

```
#!/bin/bash
#SBATCH --job-name=SUBMIT-ROSETTA         # Job name
#SBATCH --partition=gpu                 # Partition (all, cpu, gpu)
#SBATCH --cpus-per-task=1             # Number of cores per MPI task
#SBATCH --nodes=1                       # Maximum number of nodes to be allocated
#SBATCH --ntasks=1                      # Number of MPI tasks (i.e. processes)
#SBATCH --ntasks-per-node=1            # Maximum number of tasks on each node
#SBATCH --gpus=1                        # count of GPUs required for the job
#SBATCH --output=example2.%j.out   # Path to the standard output and error files relative to the working directory
#SBATCH --error=example2.%j.err    # Path to the error files relative to the working directory
#SBATCH --mem=100G

module purge
module load nvidia/cuda/11.8
module load gnu11
module load miniconda3
eval "$(conda shell.bash hook)"
conda activate rosettafold

export OMP_NUM_THREADS=1

#python3 pytorch_example.py
./run_e2e_ver.sh ../inputs_RoseTTAFold/example2.fasta # Path to inputs directory and then your fasta file
```

Once we create this file name it `slurm-example.sh` and to run it we just have to run this command:
```
sbatch slurm-example.sh
```

You can monitor the Job:
```
squeue -u <your_username>
```

Or you can monitor the logs by going to your output folder and clicking the logs for your specific fasta file.

## Expected outputs
For the end-to-end version, there will be a single PDB output having estimated residue-wise CA-lddt at the B-factor column (t000_.e2e.pdb).

## Example: Predicting Protein Structure with RoseTTAFold

In this section, we’ll demonstrate how to predict protein structures using RoseTTAFold on an HPC setup. We’ll use two FASTA sequences representing lysozyme proteins from *Homo sapiens*—one representing the wild-type structure and another with a point mutation. We’ll then compare the predicted 3D structures generated by RoseTTAFold.

1. Preparing the Input FASTA Files

You’ll need two FASTA files as input:

Wild type:

```
>Wild type LYSOZYME|Homo sapiens
KVFERCELARTLKRLGMDGYRGISLANWMCLAKWESGYNTRATNYNAGDRSTDYGIFQINSRYWCNDGKTPGAVNACHLSCSALLQDNIADAVACAKRVVRDPQGIRAWVAWRNRCQNRDVRQYVQGCGV
```

1JKB type:

```
>1JKB LYSOZYME|Homo sapiens
KVFERCELARTLKRLGMDGYRGISLANWMCLAKWASGYNTRATNYNAGDRSTDYGIFQINSRYWCNDGKTPGAVNACHLSCSALLQDNIADAVACAKRVVRDPQGIRAWVAWRNRCQNRDVRQYVQGCGV
```

2. Running RoseTTAFold

After preparing the FASTA files, you can run the structure prediction using the RoseTTAFold pipeline. The process involves submitting a job with the following SLURM script (adjust paths to your setup):

```
#!/bin/bash
#SBATCH --job-name=ROSETTA_PREDICTION
#SBATCH --partition=gpu
#SBATCH --cpus-per-task=1
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --gpus=1
#SBATCH --output=rosetta_output.%j.out
#SBATCH --error=rosetta_error.%j.err
#SBATCH --mem=100G

module load nvidia/cuda/11.8
module load gnu11
module load miniconda3
eval "$(conda shell.bash hook)"
conda activate rosettafold

# Running RoseTTAFold for both the wild-type and mutant sequences
./run_e2e_ver.sh ../inputs_RoseTTAFold/wild_type.fasta
./run_e2e_ver.sh ../inputs_RoseTTAFold/1JKB.fasta
```

3. Visualizing the prediction

For this step if you do not have Chimera on your HPC, you can transport the output files locally using this command with your local terminal:
```
scp -r your_username@your_hpc_address:/path/to/your/output/files /path/to/your/local/directory
```

Installing Chimera locally: [Download UCSF Chimera](https://www.cgl.ucsf.edu/chimera/download.html)

Open up your Chimera locally once downloaded and input your PDB files, and this is what you should see:

**1JKB PDB Files:**

![Lysozyme Structure](https://github.com/vasemili/RoseTTAFold-HPC/blob/main/examples/RoseTTAFoldimage.png)

**Wild Type PDB Files:**

![Lysozyme Structure](https://github.com/vasemili/RoseTTAFold-HPC/blob/main/examples/RoseTTAFoldWildimage.png)

4. Comparing the Structural Differences:

To visually compare the wild-type and mutant protein structures, follow these steps using UCSF Chimera:

**Step 1: Open both PDB files with Chimera**

This is what you should see once both files are open on Chimera:

![Lysozyme Structure](https://github.com/vasemili/RoseTTAFold-HPC/blob/main/examples/ComparingPDBs.png)

**Step 2: Superimpose the Structures**

Use the command line (under Favorites > Command Line) and run:

```
matchmaker #0 #1
```

This aligns the structures based on their residues. You should get *RMSD between 130 pruned atom pairs is 0.073 angstroms; (across all 130 pairs: 0.073)* for the residuals. Meaning the structurals are very identical. This is the visualization you should get:

![Lysozyme Structure](https://github.com/vasemili/RoseTTAFold-HPC/blob/main/examples/MatchedPDBs.png)

**Visualizing Differences:**

Rotate and zoom into regions of interest to see how the mutation affects the structure. You can color different parts of the structures for better visualization, run this on the Chimera command line:

```
color hot pink #0
color sky blue #1
```

The sky blue command is there just in case you inputted PDB files in different order.

You can use the command line in Chimera to highlight specific residues around the mutation site (in our case it should be position 34):

```
select :34
color red :34
```

This is the final visualization you should see:

![Lysozyme Structure](https://github.com/vasemili/RoseTTAFold-HPC/blob/main/examples/FocusedMatchPDB.png)


## Additional Information
This additional information is here in case some libraries did not get installed:

If you do not have pytoch install, or it did not install properly then you can install it using this command with a linux system:
```
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
```
If that did not work then you have different settings, and you need to go this page: `https://pytorch.org/get-started/locally/` to see the specific command for your system.

If you do not have psipred installed:
```
conda install biocore::psipred
```

If you do not have hhsuite installed:
```
conda install -c conda-forge -c bioconda hhsuite
```



## FAQ
1. Segmentation fault while running hhblits/hhsearch  
For easy install, we used a statically compiled version of hhsuite (installed through conda). Currently, we're not sure what exactly causes segmentation fault error in some cases, but we found that it might be resolved if you compile hhsuite from source and use this compiled version instead of conda version. For installation of hhsuite, please see [here](https://github.com/soedinglab/hh-suite).

2. Submitting jobs to computing nodes  
The modeling pipeline provided here (run_pyrosetta_ver.sh/run_e2e_ver.sh) is a kind of guidelines to show how RoseTTAFold works. For more efficient use of computing resources, you might want to modify the provided bash script to submit separate jobs with proper dependencies for each of steps (more cpus/memory for hhblits/hhsearch, using gpus only for running the networks, etc). 

## Links:


* [Robetta server](https://robetta.bakerlab.org/) (RoseTTAFold option)
* [RoseTTAFold models for CASP14 targets](https://files.ipd.uw.edu/pub/RoseTTAFold/casp14_models.tar.gz) [input MSA and hhsearch files are included]

## Credit to performer-pytorch and SE(3)-Transformer codes
The code in the network/performer_pytorch.py is strongly based on [this repo](https://github.com/lucidrains/performer-pytorch) which is pytorch implementation of [Performer architecture](https://arxiv.org/abs/2009.14794).
The codes in network/equivariant_attention is from the original SE(3)-Transformer [repo](https://github.com/FabianFuchsML/se3-transformer-public) which accompanies [the paper](https://arxiv.org/abs/2006.10503) 'SE(3)-Transformers: 3D Roto-Translation Equivariant Attention Networks' by Fabian et al.


## References

M. Baek, et al., Accurate prediction of protein structures and interactions using a three-track neural network, Science (2021). [link](https://www.science.org/doi/10.1126/science.abj8754)

I.R. Humphreys, J. Pei, M. Baek, A. Krishnakumar, et al, Computed structures of core eukaryotic protein complexes, Science (2021). [link](https://www.science.org/doi/10.1126/science.abm4805)
