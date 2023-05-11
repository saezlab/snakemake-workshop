Environment management
======================

Snakemake has the big advantage of being agnostic to which programming language you code your scripts in. This means that you can use the same workflow to run R, Python, Perl, Bash, etc. scripts.
However, this also means that you need to manage the environment(s) in which these scripts are run. Snakemake supports both `conda <https://conda.io>`_ and containers for this purpose.


Using conda/mamba
-----------------
Thankfully, it is relatively easy to integrate conda environments into Snakemake. For this you need three things:
- A working conda installation (we recommend using mamba instead)
- Adding a ``conda:`` directive to your rule
- A description of the conda environment in a ``environment.yaml`` file

Installing conda
~~~~~~~~~~~~~~~~
If you don't have conda installed yet, you can install it by following the instructions on the `conda website <https://conda.io/projects/conda/en/latest/user-guide/install/index.html>`_.

Alternatively, it is recommended to install `mamba <https://mamba.readthedocs.io/en/latest/installation.html>`_ instead, as it is more efficient in identifying and downloading relevant dependencies.
You can install mamba by following the instructions on the `mamba website <https://mamba.readthedocs.io/en/latest/installation.html>`_, or you can also install the `mambaforge distribution <https://github.com/conda-forge/miniforge#mambaforge>`_.

By default, snakemake will use mamba in the background if it is available.

Adding a conda environment to a rule
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
To add an environment to a rule, you just need to specify the path to the ``environment.yaml`` file in the ``conda:`` directive of the rule. As in:

.. code-block:: python

    rule myrule:
        input:
            "input.txt"
        output:
            "output.txt"
        conda:
            "envs/environment.yaml"
        script:
            "scripts/myscript.py"

Importantly, snakemake interprets the path as being relative to the location of the rule definition file. So if you have a rule definition file in ``rules/myrules.smk``, and the environment file is in ``envs/environment.yaml``, then you need to specify ``conda: "../envs/environment.yaml"``.
It is good practice to keep all your environment definitions in a separate directory, such as ``envs/``.

Environment definition
~~~~~~~~~~~~~~~~~~~~~~
The environment definition is a YAML file that lists which packages you want to install as well as where conda/mamba should look for them (i.e. which channels to use). For example:

.. code-block:: yaml

    name: some_name
    channels:
      - conda-forge
      - bioconda
    dependencies:
        - python=3.8
        - pandas
        - numpy
        - matplotlib
        - seaborn
        - pip
        - pip:
            - mypackage
            - git+https://github.com/saezlab/decoupler-py.git
            - git+https://github.com/saezlab/pypath@3d28c5f7c8fab1bff9b353a6e4f86bd1cc33738d


The ``name`` field specifies the name of the environment. Usually, conda will use this as an environment name. Snakemake actually ignores this field and replaces it with a hash of the file's contents in order to track any changes in the environment that could cause changes to the output of the workflow (Snakemake verions above 7.0).

The ``channels`` field specifies which channels conda should use to look for packages. The order is important as long as conda is configured to use strict channel priorities. Snakemake will tell you if this is the case or not. Also see the section on `channel priorities tips and tricks <https://conda-forge.org/docs/user/tipsandtricks.html>`_.

Then you can list the dependencies that you need. Most dependencies will be available from the ``conda-forge`` and ``bioconda`` channels. You can always check on anaconda.org if and on which channel a package is available. If you need a package that is not available on conda, you can also install it with ``pip`` (see the example above). 
Python packages can be installed directly from github through pip. The example above shows you how to install the master branch or a specific commit of decoupler-py and pypath respectively.

By default the environments will be installed in ``.snakemake/conda/`` of your workdir.

It is good practice in general to pin the version of the packages you are using so that it is reproducible after the fact. This is especially important for packages that are installed from github as they might change over time. For example, if you install a package from github, you should always specify the commit hash that you are using.
See below how to pin your environment.

Post-deploy scripts
~~~~~~~~~~~~~~~~~~~
Sometimes, you need to run some additional commands after the environment has been deployed: download reference data or configure some packages. Snakemake allows you to define a specific bash script that will be run after the environment has been deployed.
This bash script needs to be located in the same folder as the ``<env>.yaml`` file that defines the environment, with the same name but ending in ``.post-deploy.sh``. For example, if your environment definition is in ``envs/<env>.yaml``, then the post-deploy script needs to be ``envs/<env>.post-deploy.sh``. The conda prefix is available as the environment variable ``$CONDA_PREFIX`` in this script.

Here is an example of a post-deploy script that activates the conda environment and the executes an R script:

.. code-block:: bash

    #!env bash
    conda activate -p $CONDA_PREFIX

    mkdir -p logs

    (test -f logs/myRscript.post-deploy.log || rm logs/myRscript.post-deploy.log)

    $CONDA_PREFIX/bin/Rscript envs/myRscript.R >> logs/myRscript.post-deploy.log 2>&1

Note that the output of the script is rerouted to a ``.log`` file, as currenlty snakemake does not print any output from the post-deploy script. In that case, it also expects R to be installed already (specified in the environment definition).

.. note:: 

    I use this kind of post-deploy script routinely to install R packages from github. This is mainly due to the fact that bioconda packages are usually outdated and conda does not have the utility to install from github for R.

    Please note, that you need to have the ``remotes`` package installed in your R environment in order to install packages using ``remotes::install_github('somerepo', upgrade=never)``. Specify ``upgrade=never`` to avoid upgrading packages that are already installed (e.g. pinned packages in ``<env>.yaml`` file). Also think of pinning the version to a specific tag or commit hash.

Execute with conda
~~~~~~~~~~~~~~~~~~
Once you have proceeded with the steps above, you can execute your workflow with conda by specifying the ``--use-conda`` flag on the CLI. If the environment is already deployed it will activate the appropriate environment. If not, it will create the environment first and then activate it.

You can trigger the creation of environments by specifying the ``--conda-create-envs-only`` flag on the CLI. This will create all environments and then exit. This is useful if you want to create all environments first and then execute the workflow offline.

If you use profiles, you can also specify the ``use-conda: True`` flag in the profile configuration file. This will make sure that the flag is always used when executing the workflow with that profile.

Pin files
~~~~~~~~~
You can provide an explicit list of packages and versions to be installed in the environment by providing a ``<env>.<platform>.pin.txt`` file in the same folder as the ``<env>.yaml`` file. This will accelerate the deployment time and ensure reproducibility over longer time periods. If the dependencies are not available it will fall back to the ``<env>.yaml`` file.

These pin files are created with the `snakedeploy <https://snakedeploy.readthedocs.io/>`_ package. Pin your packages with:

.. code-block:: bash

    snakedeploy pin-conda-envs envs/<env>.yaml

You can provide multiple environments at once.

While the pin files are not required during development, they should be committed to your repository so that they are available once you finish a project or need to rerun it later.

.. note:: 

    Any packages installed with a post-deploy script will not be pinned. You need to pin them manually in the script.

Containers
----------
Snakemake also allows you to use containers to deploy your workflow. I have not used this feature yet, but you can find more information in the `snakemake documentation <https://snakemake.readthedocs.io/en/stable/snakefiles/deployment.html#running-jobs-in-containers>`_.