class: title

# Byte-sized RSE: Introduction to HPC with Python
## Corran Webster
### Southampton Research Software Group

---
class: title

# Overview

**Introduction:** Introduction to HPC with Python

**Practical activity:** Python Concurrency

## You should have done setup!

---
class: title reverse

# HPC with Python
# Introduction

---

# High Performance Computing Basics

- An HPC cluster is a group of powerful computers dedicated to computational tasks
- Each individual computer or 'node' is more powerful than a typical workstation
- A cluster will have shared high-performance disk storage
- In the UK HPC facilities are ranked in Tiers from local to continental
- Iridis is Southampton's local HPC facility
- It consists of 3 clusters: Iridis 6, Iridis X and Iridis 7

???

-

---

# Accessing Iridis

- You need an account on Iridis.
- Access to Iridis is through secure shell (SSH) to a login node.
- The login node is a Linux computer where you can perform short, small tasks.
- You can run HPC batch jobs using the SLURM scheduler and the `sbatch` command.
- Jobs are configured with a BASH **submission script**
- You can also request a short interactive session with the `sinteractive` command.

???

-

---

# Submission Scripts

.left-column[

```bash
#!/bin/bash

#SBATCH --partition=batch
#SBATCH --time=00:01:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=2

# This is the command that will run
pwd
```
]
.right-column[

- The script is a collection of Bash commands to run one after another
- The special `#SBATCH` comments configure the parameters of the job.
]

???

- `partition` selects which group of computers to use
- `nodes` selects how many computers to use; each node runs one or more tasks
- `ntasks` or `ntasks_per_node` select how many different tasks or concurrent programs to run
- `cpus-per-task` selects how many CPUs to allocate to each task
- `time` sets the maximum time to allow for the job

---

# Modules

- Different HPC programs use different versions of underlying technologies
- You need to select which technologies you are using, such as languages, GPU support, compilers, etc. through the **module** system on Iridis
- For example, if your code needs Python 3.14 you load it using the Bash command `module load python/3.14`
- You can see all available modules with the `module avail` command.

---

```bash
#!/bin/bash

#SBATCH --job-name=python-example
#SBATCH --partition=batch
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1

# Load required modules
module load python

# Activate Python virtual environment
source ~/myenv/bin/activate

# Set any environment variables or configuration options
export PYTHONUNBUFFERED=1

# Move to job directory
cd $SLURM_SUBMIT_DIR

# Run the Python script
python my_script.py
```

---

# Monitoring Jobs

- The `squeue -u $USER` command shows you the state of your jobs.
- You can get more detailed information about a job using `scontrol show jobid=` and the ID of your job
- You can cancel a job using the `scancel` command with the ID of the job.
- Output will appear in a text file starting with `slurm-` followed by the job number.
- When complete, this file will contain error messages and performance feedback.

---

# MPI (Message Passing Interface)

- When running jobs on multiple nodes you run into the issue of subdividing work.
- MPI provides a way for nodes to communicate with each other.
- For Python code, MPI4Py is the current best option for using MPI.
- Basic MPI is fairly low-level and intrusive to your code.
- We will use a different approach via Python's `concurrent.futures` module. This permits almost identical code to be run locally and with MPI.

---

# Effective Use of HPC

- Research code requires a lot of experimentation.
- The faster your experiments fail, the quicker you can work out what is wrong.
- HPC batch systems have long turn-around times.
- Ideally, get code working *locally* first, then run the *same* code under HPC and scale up.
- Make your code as fast as possible before moving to HPC.

---

class: title reverse

# Data Augmentation
# Practical Activity

---

# Activity Introduction

- We have code that prepares images for machine learning which we will prepare for running on Iridis 6.
- We'll demonstrate this via live coding.
  - We'll use a repository with example code.

---
class: title reverse

# Thank you!
# What questions do you have?
