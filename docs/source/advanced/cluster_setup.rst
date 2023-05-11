SLURM cluster setup
===================
Snakemake has the great functionality of being able to run just as well on your local computer as on a cluster. For the latter, it can also take care of submitting your jobs to the Slurm scheduler.

Profile
-------
A profile, is a folder which contains information on the default options you would like to execute snakemake with, as well as how to submit jobs. Luckily, there already exists a premade cookiecutter template for SLURM.

Firstly, you will need to connect to your cluster of choice and go to your working directory, install cookiecutter to your conda environment and then load the template. 

.. code-block:: console

    ssh user@login1.bioquant.uni-heidelberg.de #connect to bioquant server
    cd working-dir

    conda activate base
    conda install cookiecutter # in order to install profiles
    cookiecutter https://github.com/Snakemake-Profiles/slurm.git

The cookiecutter command will interactively let you choose what kind of profile you would like to create. They will be set in ``config.yaml`` inside the profile folder. If you do not know what to use, you can click enter and use the default ones (in brackets).

Here are some of the most important parameters:

profile_name
    folder where profile will be stored

use_singularity
    boolean, whether to use singularities by default if they are in the rules (e.g. docker image)

use_conda
    boolean,  whether to use conda environments by default if they are in the rules. I recommend setting this to true

jobs
    the number of jobs that can be active at the same time, set it to 10, or at least lower than the default 500. Equivalent of specifying cores on your local machine

restart_times
    How many times to relaunch when a job fails. Set to 0. Can be used to relaunch failed jobs `with more memory or time. <https://bluegenes.github.io/hpc-snakemake-tips/>`_

max_status_checks_per_second
    How many times it can check for job completion. Should be scaled with the number of jobs.

latency_wait
    Set to 60. How long snakemake waits for files after job is completed, before calling them failed.


I recommend storing this profile (folder) inside the ``config/`` directory. You can just copy-paste it to your various projects and change them individually as necessary.

The profile contains ``config.yaml`` that defines default options for snakemake when you run it with the profile. Here is what I am currently using on the BioQuant cluster.

.. code-block:: yaml

    cluster-sidecar: "slurm-sidecar.py"
    cluster-cancel: "scancel"
    restart-times: "0"
    jobscript: "slurm-jobscript.sh"
    cluster: "slurm-submit.py"
    cluster-status: "slurm-status.py"
    max-jobs-per-second: "2"
    max-status-checks-per-second: "10"
    local-cores: 1
    latency-wait: "60"
    use-conda: "True"
    use-singularity: "False"
    jobs: "5"
    printshellcmds: "True"

    # Example resource configuration
    # default-resources:
    # default-resources:  
    #   - runtime=14400
    #   - mem_mb=16000
    #   - disk_mb=1000000
    # # set-threads: map rule names to threads
    # set-threads:
    #   - single_core_rule=1
    #   - multi_core_rule=10
    # # set-resources: map rule names to resources in general
    # set-resources:
    #   - high_memory_rule:mem_mb=12000
    #   - long_running_rule:runtime=1200

Renaming logs
-------------
When you run jobs on the cluster, all console output from snakemake will be redirected automatically to the slurm .out and .err files. You can change their naming by modifying ``settings.json`` to the following:

.. code-block:: json

    {
        "SBATCH_DEFAULTS": "job-name=smk-{rule}-{wildcards} output=logs/{rule}/{rule}-{wildcards}-%j.out",
        "CLUSTER_NAME": "",
        "CLUSTER_CONFIG": "",
        "ADVANCED_ARGUMENT_CONVERSION": "no"
    }

Change the naming scheme as is most convenient to you, or to a dedicated slurm logs folder

Change sbatch defaults
----------------------
You can change other sbatch behaviours by adding the approproate keywords to ``settings.json``. For example, you can exclude specific nodes from running your jobs by using the ``exclude`` slurm keyword:

.. code-block:: json

    {
        "SBATCH_DEFAULTS": "exclude=cln[079,143],denbi4 job-name=smk-{rule}-{wildcards} output=logs/{rule}/{rule}-{wildcards}-%j.out",
        "CLUSTER_NAME": "",
        "CLUSTER_CONFIG": "",
        "ADVANCED_ARGUMENT_CONVERSION": "no"
    }

This can be useful on the BioQuant cluster, since some nodes are sometimes faulty and cancel jobs, which unfortunately is not caught by snakemake.
You can find a cheatsheet of sbatch commands `here <https://slurm.schedmd.com/pdfs/summary.pdf>`_.

In general, it is still recommended to modify the sbatch defaults in the profile (``config.yaml``), since it has the advantage of being interpreted by the submission script.

Rules with resources
--------------------
Rules can have ``resources`` and ``threads`` directives. For example:

.. code-block:: python

    rule misty_views:
        input:
            expr = "results/{sandwich}/Misty/view_p{radius}_expr.rds",
            metab = "results/{sandwich}/Misty/view_p{radius}_metab.rds"
        output: 
            directory("results/{sandwich}/Misty/model_p{radius}_int-{dtype}")
        threads: 4
        resources:
            mem_mb=25000,
            disk_mb=1000,
            time='12:00:00'
        script: "../scripts/Misty/misty_exp2_run.R"

These overwrite the defaults from the profile and ask for the appropriate resource allocation.

Using tmux
----------
When you execute snakemake on the cluster, it runs for the whole time your jobs are running as well and submits new jobs whenever necessary. However, if you just log in normally, any task running when you exit the ssh connection will be stopped.

tmux allows you to have a virtual session running, that you can connect to and disconnet from without cancelling any tasks. It is already preinstalled on the BioQuant cluster. The following command opens a new session called snakes:

.. code-block:: console

    tmux new -s snakes

You detach from it with ``Ctrl-B + D``. To re-attach to the previous session:

.. code-block:: console

    tmux a

You can find a cheatsheet with tmux commands `here <https://tmuxcheatsheet.com>`_

Running snakemake inside interactive job
----------------------------------------

`Installing snakemake <https://snakemake.readthedocs.io/en/stable/getting_started/installation.html#installation-via-conda-mamba>`_ and your conda environments is no different than installing it on your local computer.

Snakemake monitors your jobs while they are running. It should therefore **not be run on the login nodes** but in a separate interactive job.

.. code-block:: console

    tmux a #attach to your previous tmux session

    srun -t 5:00:00 --mem=5G --pty bash #slurm interactive job

    conda activate snk #activate your snakemake environment

    #check dry-run your rule all
    snakemake -n --profile ./path_profile_dir 

    #-j N specifies the max number of simultaneous jobs submitted at the same time
    #launch snakemake with max 10 parallel jobs, overwrites whatever is in the defaults
    snakemake -j 10 --profile ./path_profile_dir 

.. note::
    Be aware that when your interactive job ends, snakemake will not be able to track correct job completions etc. The interactive job should therefore have a longer max runtime than any of your jobs
