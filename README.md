# A Quick Guide to Slurm
I wrote up a quick guide to how I use Slurm on the VisualAI nodes. This is by no means complete, and I'd appreciate any pull requests that can explain further or edit my content to be even better.

## Running a job on the VisualAI nodes
Running a job on the VisualAI nodes is now handled by SSH'ing into ionic via the command:
```sh
$ ssh <netid>@ionic.cs.princeton.edu
```

To run jobs on the VisualAI systems, the job command should contain the `-A visualai` flag as follows:
```sh
$ sbatch job_name.sh -A visualai
```

##### Common Issues:
1. When running a job, I get the error `sbatch: error: Batch job submission failed: Invalid account or account/partition combination specified`.**

  It might be possible that you do not have permissions to run jobs in the VisualAI group. In order to do this, make sure you are subscribed to both the [beowulf](https://lists.cs.princeton.edu/mailman/listinfo/beowulf) and [visualai-cluster](https://lists.cs.princeton.edu/mailman/listinfo/visualai-cluster) listservs. Afterwards, email csstaff to be permissioned into the VisualAI group.

## Setting up your Slurm environment
In order to run jobs on Slurm, you need to set up a script such that adequate are resources are allocated by the Slurm scheduler to make your job runnable. For example, when I submit my jobs, I use the command
```sh
$ sbatch run_test_job.sh
```
where my file `run_test_job.sh` contains the contents:
```sh
#!/bin/bash
#SBATCH --job-name=job-name-display    # the name of the job
#SBATCH --output=output.txt            # where stdout and stderr will write to
#
#SBATCH --gres=gpu:1                   # number of GPUs your job requests
#SBATCH --mem=32G                      # amount of memory needed
#SBATCH --time=2:00:00                 # limit on total runtime
#
# send mail during process execution
#SBATCH --mail-type=all
#SBATCH --mail-user=<netid>@princeton.edu
#

srun -A visualai ./test_val_network.sh
```

A great resource for additional Slurm options can be found [here](https://slurm.schedmd.com/sbatch.html).

This script then calls _another_ script `test_val_network.sh`, which looks as follows:

```sh
#!/bin/bash

module load caffe/1.00

echo "RGB model on val..."
python test_network.py --deploy_net prototxts/deploy_clip_retrieval_rgb_iccv_release_feature_process_context_recurrent_embedding_lfTrue_dv0.3_dl0.0_nlv2_nlllstm_no_embed_edl1000-100_edv500-100_pmFalse_losstriplet_lwInter0.2.prototxt \
                       --snapshot_tag rgb_iccv_release_feature_process_context_recurrent_embedding_lfTrue_dv0.3_dl0.0_nlv2_nlllstm_no_embed_edl1000-100_edv500-100_pmFalse_losstriplet_lwInter0.2 \
                       --visual_feature feature_process_context \
                       --language_feature recurrent_embedding \
                       --max_iter 30000 \
                       --snapshot_interval 30000 \
                       --loc \
                       --test_h5 data/average_fc7.h5 \
                       --split val \
```

The reason for two scripts is that the first script is called on the ionic head node (where your Terminal session is occurring), which tells the second script to run on a **visualai node**. This is important because some modules, like `caffe/1.00` are installed _only_ on the visualai nodes, and so the line `module load caffe/1.00` must run while executing on a visualai node. Note that the first script runs on the ionic head node, and the second script runs on a visualai node.

This is how I currently run my jobs on the visualai nodes, though it is quite probable that this can be done more efficiently (e.g. interactive jobs).


## Interactive Jobs
Interactive jobs on Slurm allow you perform tasks directly on the visualai nodes through your Terminal session. An example command for starting an interactive session might look like:
```sh
$ salloc --gres=gpu:1 -c 2 --mem=10G -A visualai srun --pty $SHELL -l
```
After the `salloc` command, we can specify the same parameters by including them as options rather than in a setup script. The parameters for `salloc` look familiar, but latter part of the command can be broken down:
 - `srun` is specified so that it can be omitted when writing commands during the interactive session. If `srun` were not added here, commands written in the interactive session would get executed on the access nodes rather than with the allocated resources.
 - the `--pty` flag is important for getting something that behaves like a terminal.
 - the `$SHELL` tells `srun` to request what's running behind the shell variable of that system (i.e. bash on the ionic nodes)
 - `-l` prepends the task number to the lines of stdout

Additional flags can be added by referencing the options for [`salloc`](https://slurm.schedmd.com/salloc.html) and [`srun`](https://slurm.schedmd.com/srun.html).

## Useful Slurm Commands
A good basic tutorial for using Slurm commands is through [this guide](https://www.rc.fas.harvard.edu/resources/running-jobs/), and a good list of Slurm commands can be found [here](https://hpc.llnl.gov/banks-jobs/running-jobs/slurm-commands). Some of the convenient ones that I use:

  - [`squeue`](https://slurm.schedmd.com/squeue.html): see a list of running jobs
  - [`sacct`](https://slurm.schedmd.com/sacct.html): see a list of your recently run jobs.
  - [`scancel <jobid>`](https://slurm.schedmd.com/scancel.html): cancel your running job with id `jobid`
  - [`sbatch`](https://slurm.schedmd.com/sbatch.html): used to submit a batch script to the Slurm scheduler. See example use above
  - [`srun`](https://slurm.schedmd.com/srun.html): run a parallel job on cluster managed by Slurm

## FAQ

1. How does the `CUDA_VISIBLE_DEVICES` environment variable now work with Slurm?

  `CUDA_VISIBLE_DEVICES` is now set by Slurm's GRES plugin, depending on the number of CPUs allocated for the running job. If a job is run with the flag `--gres=gpu:2`, then the job will be run with `CUDA_VISIBLE_DEVICES=0,1`
