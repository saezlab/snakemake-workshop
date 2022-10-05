.. _setup for tutorial:
Setup for the tutorial
======================

Of course, any decent snakemake tutorial will start by installing snakemake. Install the latest snakemake package as described in the `setup guide <https://snakemake.readthedocs.io/en/stable/getting_started/installation.html#installation-via-conda-mamba>`_

For M1 MacOS
------------
For many packages there are no versions available for the M1 chips. If you encounter this situation, you can install packages that were compiled for Intel chips instead. For this, you need to create the snakemake environment by setting appropriate flags. Snakemake will then automatically install packages that were designed for Intel. This occurs frequently, mainly with Bioconductor packages.

.. note:: 
    Be aware that the first time you load/execute packages from this conda environment it might seems like the execution is hanging. This is normal as the code is translated for your processor and subsequent executions will be faster.

.. code-block:: console
    
    CONDA_SUBDIR=osx-64 conda create -n snk64   # create a new environment called snk64
    conda activate snk64
    conda env config vars set CONDA_SUBDIR=osx-64  # subsequent commands use intel packages

Conda might prompt you to activate the base environment again to save changes. It will also display warning messages every time you (de-)activate this environment.

You can now follow the same guide as above to install the snakemake package.


Download tutorial repository
----------------------------
Now that you have a conda environment with basically only snakemake installed, let's clone the tutorial git repository. Navigate to a folder where you would like to clone it to.

.. code-block:: console

    git clone https://github.com/saezlab/snk-tutorial
    cd snk-tutorial

The project repository structure is shown below. Snakemake developers recommend a folder structure that separates the workflow from data, results and configurations files.
.. note:: 

    This structure is a more lightweight version of the cookiecutter template provided by the snakemake workflows project. You can find how to setup your own project directory using a template :ref:`here <cookiecutter>`.


.. code-block:: none

    ├── .gitignore
    ├── LICENSE
    ├── README.md
    ├── workflow
    │   ├── rules
    |   │   ├── download_data.smk
    │   ├── envs
    |   │   ├── scanpy.yaml
    │   ├── scripts
    |   │   ├── fake_samples.py
    |   │   ├── QC_samples.py
    |   └── Snakefile
    └── config
        └── config.yaml

The ``workflow`` folder contains a Snakefile, and several subfolders which contain pipeline modules, scripts and any package dependencies required to execute the workflow. There is already a module called ``download_data.smk`` which will take care of downloading some toy data (PBMCs) and create some toy samples. 

Editing the Snakefile
---------------------

You can use any text editor to edit snakemake code found in the Snakefile, which is just a mix of python code and yaml-like directives. I personally use `VS Code <https://code.visualstudio.com>`_ with the `Snakemake language extension <https://marketplace.visualstudio.com/items?itemName=Snakemake.snakemake-lang>`_, whereas snakemake creators recommend using the `Atom editor <https://atom.io>`_. You can also use any IDE with python code highlighting such as Jupyterlab.

Let's now take a look at the Snakefile:

.. code-block:: python
    
    from snakemake.utils import min_version
    min_version("7.0")

    configfile: "config/config.yaml"

    # define which output files you would like to generate
    rule all:
        input:
            'data/sample1.h5ad'


    module download_data:
        snakefile: "rules/download_data.smk"
        config: config

    use rule * from download_data as dwn_*

Overall you can see that it is python code with two blocks in YAML. Firstly, it requires a minimum version requirement of snakemake itself. Then it defines the path to the ``configfile``, where parameters used in the workflow are stored. These parameters are then available in the nested dict ``config``.

.. note::
    Newer versions of snakemake keep track of modifications to this config file and will prompt you to rerun your workflow if it has changed. It does however not track exactly which parameters changed, so it is left to the user whether it requires a rerun or not.

Then, there is a ``rule all`` statement: this is a special rule with only inputs, no outputs and no actual task. This is a special rule placed always at the top of the ``Snakefile`` and defines which files you want to create in the workflow, instead of writing them out by hand. Additionally, it allows you to add files programmatically using python. 

You can check exactly which processes will be run using the following command:

Dry-run example
---------------

.. code-block:: console

    snakemake --use-conda -n

The command specifies that it should be run using any defined environments with ``--use-conda``. The ``-n`` flag triggers a dry-run and tells you exactly what will be launched. It also let's you know how many processes will be launched and can help estimate how many cores you should use. The output should look something like the following:

.. code-block:: console

    Building DAG of jobs...
    Conda environment workflow/envs/scanpy.yaml will be created.
    Job stats:
    job                 count    min threads    max threads
    ----------------  -------  -------------  -------------
    all                     1              1              1
    dwn_download            1              1              1
    dwn_make_samples        1              1              1
    total                   3              1              1


    [Thu Sep 22 15:47:04 2022]
    checkpoint dwn_download:
        output: data/filtered_gene_bc_matrices/hg19
        jobid: 2
        reason: Missing output files: data/filtered_gene_bc_matrices/hg19
        resources: tmpdir=/var/folders/j2/xqm_3c792md7svmbnk61b97c0000gn/T
    Downstream jobs will be updated after completion.


    [Thu Sep 22 15:47:04 2022]
    rule dwn_make_samples:
        input: <TBD>
        output: data/sample1.h5ad, data/sample2.h5ad, data/sample3.h5ad
        jobid: 1
        reason: Missing output files: data/sample1.h5ad; Input files updated by another job: data/filtered_gene_bc_matrices/hg19
        resources: tmpdir=/var/folders/j2/xqm_3c792md7svmbnk61b97c0000gn/T

    [Thu Sep 22 15:47:04 2022]
    localrule all:
        input: data/sample1.h5ad
        jobid: 0
        reason: Input files updated by another job: data/sample1.h5ad
        resources: tmpdir=/var/folders/j2/xqm_3c792md7svmbnk61b97c0000gn/T

    Job stats:
    job                 count    min threads    max threads
    ----------------  -------  -------------  -------------
    all                     1              1              1
    dwn_download            1              1              1
    dwn_make_samples        1              1              1
    total                   3              1              1

    Reasons:
        (check individual jobs above for details)
        input files updated by another job:
            all, dwn_make_samples
        missing output files:
            dwn_download, dwn_make_samples

    This was a dry-run (flag -n). The order of jobs does not reflect the order of execution.

The output first tells you that a new conda environment needs to be created. You can take a look at the corresponding dependency file to see which packages will be downloaded.

Then it shows you that there are three separate jobs that would be run: 'all' is what you have seen previously in the ``Snakefile``, the other two are defined in the download module. Removing the dry-run flag would first install the conda environment and then execute the jobs.

Install dependencies
--------------------
It can be useful to do the installation separately, especially if you have complex dependencies, or if you want to set up the environments for later execution without access to the internet. When you actually run a job, you need to specify the number of cores you will use with ``-c N`` or ``-cN``, where N is the number of cores.

.. code-block:: console

    snakemake --conda-create-envs-only --use-conda -c1

.. code-block:: console

    Building DAG of jobs...
    Creating conda environment workflow/envs/scanpy.yaml...
    Downloading and installing remote packages.
    Environment for /Users/user/Documents/Projects/snk-tutorial/workflow/rules/../envs/scanpy.yaml created (location: .snakemake/conda/d6540f768478c6b08ce2736c834601d8_)

The installation should work flawlessly and the environment will be stored inside the ``.snakemake/`` folder located in the working directory, with a hash as name. Any changes in the dependency file will trigger a new installation.

Download data
-------------
With the necessary dependencies installed, you can now download the data with the following command:

.. code-block:: console
    
    snakemake --use-conda -c1

.. note:: 
    You can see that any output to the shell or stdout/stderr are printed to the console. For parallelised jobs this will print every job output simultaneously to the same console. 

    You can check older run logs in the ``.snakemake/log`` directory.
    
    Think about setting up `your own logging <https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#log-files>`_ for local execution. In slurm cluster exection, the output is automatically sent to the equivalent .out or .err files separately for each job.

If you now try to request one sample again, snakemake will tell you that there is nothing to be done:

.. code-block:: console

    snakemake data/sample1.h5ad --use-conda -c1

.. code-block:: console

    Building DAG of jobs...
    Updating job dwn_make_samples.
    Nothing to be done (all requested files are present and up to date).
    Complete log: .snakemake/log/2022-09-22T111259.106356.snakemake.log

This is exactly the functionality that makes snakemake so useful: only do what is necessary. 
