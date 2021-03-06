.. _containers:

Generating container recipes & images
=====================================

EasyBuild has support for generating *container recipes* that will use EasyBuild to build and install a
specified software stack. In addition, EasyBuild can (optionally) leverage the build tool provided by the
container software of choice to create *container images*.

.. note:: The features documented here have been available since EasyBuild v3.6.0 but are still *experimental*,
          which implies they are subject to change in upcoming versions of EasyBuild.

          You will need to enable the ``--experimental`` configuration option in order to use them.
          See :ref:`experimental_features` for more information.

For now (since EasyBuild v3.6.0), only Singularity (http://singularity.lbl.gov/) is supported.

.. contents::
    :depth: 3
    :backlinks: none

.. _containers_req:

Requirements
------------

* Singularity v2.4 or more recent
* having ``sudo`` permissions (only required to actually build container images, see :ref:`containers_usage_build_image`)


.. _containers_usage:

Usage
-----

.. _containers_usage_containerize:

Generating container recipes (``--containerize`` / ``-C``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To generate container recipes, use ``eb --containerize``, or ``eb -C`` for short.

The resulting container recipe will, in turn, leverage EasyBuild to build and install the software
that corresponds to the easyconfig files that are specified as arguments to the ``eb`` command
(and all required dependencies, if needed).

.. note:: EasyBuild will refuse to overwrite existing container recipes.

          To re-generate an already existing recipe file, use the ``--force`` command line option.

.. _containers_usage_container_base:

Base container image (``--container-base``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to let EasyBuild generate a container recipe, it is *required* to specify which container image
should be used as a base, via the ``--container-base`` configuration option.

Currently, three types of container base images can be specified:

* ``localimage:<path>``: the location of an existing container image file
* ``docker:<name>``: the name of a Docker container image (to be downloaded from Docker Hub, https://hub.docker.com/)
* ``shub:<name>``: the name of a Singularity container image (to be downloaded from Singularity Hub, https://singularity-hub.org/)

For the ``docker`` and ``shub`` types, an additional *tag* can be specified: ``docker:<name>:<tag>`` or ``shub:<name>:<tag>``.


.. note:: You can also instruct EasyBuild which container base image should be used via the
          $EASYBUILD_CONTAINER_BASE environment variable, or by specifying ``container-base``
          in an EasyBuild configuration file;
          see :ref:`configuration_types`.

.. note::
          EasyBuild currently does not (yet) support generating a container recipe that results in a container image
          that is built from scratch, this will be implemented in a future version of EasyBuild.
          
          To get started quickly, we recommend using one of the container base images available from
          https://singularity-hub.org/collections/143.


.. _containers_usage_container_base_requirements:

Requirements for base container image
+++++++++++++++++++++++++++++++++++++

There are a couple of specific requirements for the base container image:

* all dependencies of EasyBuild must be installed, incl. Lmod (cfr. :ref:`requirements`)
* a user named ``easybuild`` must be available
* the ``/scratch`` and ``/app`` directories must exist,
  and the ``easybuild`` user must have write permissions to those directories

The ``easybuild`` user will be used when running EasyBuild to install the specified software stack.

.. note:: The generated container recipe currently hardcodes some of this.
          We intend to make this more configurable in a future version of EasyBuild.


.. _containers_usage_build_image:

Building container images (``--container-build-image``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To instruct EasyBuild to also build a container image from the generated container recipe, use ``--container-build-image``
(in combination with ``-C`` or ``--containerize``).

EasyBuild will leverage functionality provided by the container software of choice
(see :ref:`containers_cfg_image_type`) to build the container image.

For example, in the case of Singularity, EasyBuild will run ``sudo /path/to/singularity build`` on the generated container recipe.

.. note:: In order to leverage the image building functionality of the container software, admin privileges are
          typically required. Therefore, EasyBuild will run the command to build the container image with ``sudo``.
          You may need to enter your password to let the command execute.

          EasyBuild will only run the actual container image build command with ``sudo``.
          It will not use elevated privileges for anything else.

          In case of doubt, you can use ``--extended-dry-run`` or ``-x`` do perform a dry run, so you can evaluate
          which commands will be executed (see also :ref:`extended_dry_run`).

          If you're not comfortable with this, you can just let EasyBuild generate the container recipe,
          and then use that to build the actual container images yourself, either locally or through
          Singularity Hub (https://singularity-hub.org).

The container image will be placed in the location specified by the ``--containerpath`` configuration option
(see :ref:`containers_cfg_path`), next to the generated container recipe that was used to build the image.

.. note::
    When building container images, make sure to use a file system location with sufficient available storage space.
    Singularity may pull metadata during the build, and each image can range from several hundred MBs to GBs,
    depending on software stack you are including in the container image.

.. note:: EasyBuild will refuse to overwrite existing container images.

          To re-generate an already existing image file, use the ``--force`` command line option.


.. _containers_usage_example:

Example usage
~~~~~~~~~~~~~

In this example, we will use a pre-built base container image located at ``/tmp/example.simg``
(see also :ref:`containers_usage_container_base`).

To let EasyBuild generate a container recipe for GCC 6.4.0 + binutils 2.28::

    eb GCC-6.4.0-2.28.eb --containerize --container-base localimage:/tmp/example.simg --experimental

With other configuration options left to default (see output of ``eb --show-config``),
this will result in a Singularity container recipe using ``example.simg`` as base image,
which will be stored in ``$HOME/.local/easybuild/containers``::

    $ eb GCC-6.4.0-2.28.eb --containerize --container-base localimage:/tmp/example.simg --experimental
    == temporary log file in case of crash /tmp/eb-dLZTNF/easybuild-LPLeG0.log
    == Singularity definition file created at /home/example/.local/easybuild/containers/Singularity.GCC-6.4.0-2.28
    == Temporary log file(s) /tmp/eb-dLZTNF/easybuild-LPLeG0.log* have been removed.
    == Temporary directory /tmp/eb-dLZTNF has been removed.


.. _containers_example_recipe:

Example of a generated container recipe
+++++++++++++++++++++++++++++++++++++++

Below is an example of container recipe for that was generated by EasyBuild, using the following command::

    eb Python-3.6.4-foss-2018a.eb OpenMPI-2.1.2-GCC-6.4.0-2.28.eb -C --container-base shub:shahzebsiddiqui/eb-singularity:centos-7.4.1708 --experimental

It uses the ``shahzebsiddiqui/eb-singularity:centos-7.4.1708`` base container image that is available from Singularity hub
(see https://singularity-hub.org/collections/143).

.. code::

    Bootstrap: shub
    From: shahzebsiddiqui/eb-singularity:centos-7.4.1708

    %post
    yum --skip-broken -y install openssl-devel libssl-dev libopenssl-devel
    yum --skip-broken -y install libibverbs-dev libibverbs-devel rdma-core-devel


    # upgrade easybuild package automatically to latest version
    pip install -U easybuild

    # change to 'easybuild' user
    su - easybuild

    eb Python-3.6.4-foss-2018a.eb OpenMPI-2.1.2-GCC-6.4.0-2.28.eb --robot --installpath=/app/ --prefix=/scratch --tmpdir=/scratch/tmp

    # exit from 'easybuild' user
    exit

    # cleanup
    rm -rf /scratch/tmp/* /scratch/build /scratch/sources /scratch/ebfiles_repo

    %runscript
    eval "$@"

    %environment
    source /etc/profile
    module use /app/modules/all
    module load Python/3.6.4-foss-2018a OpenMPI/2.1.2-GCC-6.4.0-2.28

    %labels



.. note:: We also specify the easyconfig file for the OpenMPI component of ``foss/2018a`` here,
          because it requires specific OS dependencies to be installed (see the 2nd ``yum ... install`` line in
          the generated container recipe).

          We intend to let EasyBuild take into account the OS dependencies of the entire software stack automatically
          in a future update.

The generated container recipe includes ``pip install -U easybuild`` to ensure that the latest version of EasyBuild
is used to build the software in the container image, regardless of whether EasyBuild was already present in the
container and which version it was.

In addition, the generated module files will follow the default module naming scheme (``EasyBuildMNS``).
The modules that correspond to the easyconfig files that were specified on the command line will be loaded
automatically, see the statements in the ``%environment`` section of the generated container recipe.


.. _containers_example_build_image:

Example of building container image
+++++++++++++++++++++++++++++++++++

You can instruct EasyBuild to also build the container image by also using ``--container-build-image``.

Note that you will need to enter your ``sudo`` password (unless you recently executed a ``sudo`` command
in the same shell session)::

    $ eb GCC-6.4.0-2.28.eb --containerize --container-base localimage:/tmp/example.simg --container-build-image --experimental
    == temporary log file in case of crash /tmp/eb-aYXYC8/easybuild-8uXhvu.log
    == Singularity tool found at /usr/bin/singularity
    == Singularity version '2.4.6' is 2.4 or higher ... OK
    == Singularity definition file created at /home/example/.local/easybuild/containers/Singularity.GCC-6.4.0-2.28
    == Running 'sudo /usr/bin/singularity build  /home/example/.local/easybuild/containers/GCC-6.4.0-2.28.simg /home/example/.local/easybuild/containers/Singularity.GCC-6.4.0-2.28', you may need to enter your 'sudo' password...
    == (streaming) output for command 'sudo /usr/bin/singularity build  /home/example/.local/easybuild/containers/GCC-6.4.0-2.28.simg /home/example/.local/easybuild/containers/Singularity.GCC-6.4.0-2.28':
    Using container recipe deffile: /home/example/.local/easybuild/containers/Singularity.GCC-6.4.0-2.28
    Sanitizing environment
    Adding base Singularity environment to container
    ...
    == temporary log file in case of crash /scratch/tmp/eb-WnmCI_/easybuild-GcKyY9.log
    == resolving dependencies ...
    ...
    == building and installing GCCcore/6.4.0...
    ...
    == building and installing binutils/2.28-GCCcore-6.4.0...
    ...
    == building and installing GCC/6.4.0-2.28...
    ...
    == COMPLETED: Installation ended successfully
    == Results of the build can be found in the log file(s) /app/software/GCC/6.4.0-2.28/easybuild/easybuild-GCC-6.4.0-20180424.084946.log
    == Build succeeded for 15 out of 15
    ...
    Building Singularity image...
    Singularity container built: /home/example/.local/easybuild/containers/GCC-6.4.0-2.28.simg
    Cleaning up...
    == Singularity image created at /home/example/.local/easybuild/containers/GCC-6.4.0-2.28.simg
    == Temporary log file(s) /tmp/eb-aYXYC8/easybuild-8uXhvu.log* have been removed.
    == Temporary directory /tmp/eb-aYXYC8 has been removed.


The inspect the container image, you can use ``singularity shell`` to start a shell session *in* the container::

    $ singularity shell --shell "/bin/bash --norc" $HOME/.local/easybuild/containers/GCC-6.4.0-2.28.simg

    Singularity GCC-6.4.0-2.28.simg:~> source /etc/profile

    Singularity GCC-6.4.0-2.28.simg:~> module list

    Currently Loaded Modules:
      1) GCCcore/6.4.0   2) binutils/2.28-GCCcore-6.4.0   3) GCC/6.4.0-2.28

    Singularity GCC-6.4.0-2.28.simg:~> which gcc
    /app/software/GCCcore/6.4.0/bin/gcc

    Singularity GCC-6.4.0-2.28.simg:~> gcc --version
    gcc (GCC) 6.4.0
    ...


.. note:: We are passing ``--shell "/bin/bash --norc`` to ``singularity shell`` to avoid that the ``.bashrc`` login
          script thay may be present in your home directory is sourced, since that may include statements that are
          not relevant in the container environment.

.. note:: The ``source /etc/profile`` statement should not be required, we intend to fix this in future updates.


Or, you can use ``singularity exec`` to execute a command in the container.

Compare the output of running ``which gcc`` and ``gcc --version`` locally::

    $ which gcc
    /usr/bin/gcc
    $ gcc --version
    gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-16)
    ...

and the output when running the same commands in the container::

    $ singularity exec GCC-6.4.0-2.28.simg which gcc
    /app/software/GCCcore/6.4.0/bin/gcc

    $ singularity exec GCC-6.4.0-2.28.simg gcc --version
    gcc (GCC) 6.4.0
    ...


Configuration
-------------

.. note:: You can specify each of these configuration options either as options to the ``eb`` command,
          via the equivalent ``$EASYBUILD_CONTAINER*`` environment variable, or via an EasyBuild configuration file;
          see :ref:`configuration_types`.

.. _containers_cfg_path:

Location for generated container recipes & images (``--containerpath``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To control the location where EasyBuild will put generated container recipes & images, use the ``--containerpath``
configuration setting. Next to providing this as an option to the ``eb`` command, you can also define
the ``$EASYBUILD_CONTAINERPATH`` environment variable or specify ``containerpath`` in an EasyBuild configuration file.

The default value for this location is ``$HOME/.local/easybuild/containers``, unless the ``--prefix`` configuration
setting was provided, in which case it becomes ``<prefix>/containers`` (see :ref:`prefix`).

Use ``eb --show-full-config | grep containerpath`` to determine the currently active setting.


.. _containers_cfg_image_format:

Container image format (``--container-image-format``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The format for container images that EasyBuild is produces via the functionality provided by the container software
can be controlled via the ``--container-image-format`` configuration setting.

For Singularity containers (see :ref:`containers_cfg_type`), three image formats are supported:

* ``squashfs`` *(default)*: compressed images using ``squashfs`` read-only file system
* ``ext3``: writable image file using ``ext3`` file system
* ``sandbox``: container image in a regular directory

See also https://singularity.lbl.gov/user-guide#supported-container-formats and http://singularity.lbl.gov/docs-build-container.


.. _containers_cfg_image_name:

Name for container recipe & image (``--container-image-name``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, EasyBuild will use the name of the first easyconfig file (without the ``.eb`` suffix) as a name
for both the container recipe and image.

You can specify an altername name using the ``--container-image-name`` configuration setting.

The filename of generated container recipe will be ``Singularity.<name>``.

The filename of the container image will be ``<name><extension>``,
where the value for ``<extension>`` depends on the image format (see :ref:`containers_cfg_image_format`):

* '``.simg``' for ``squashfs`` container images
* '``.img``' for ``ext3`` container images
* *empty* for ``sandbox`` container images (in which case the container image is actually a directory rather than a file)


.. _containers_tmpdir:

Temporary directory for creating container images (``--container-tmpdir``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The container software that EasyBuild leverages to build container images may be using
a temporary directory in a location that doesn't have sufficient free space.

You can instruct EasyBuild to pass an alternate location via the ``--container-tmpdir`` configuration setting.

For Singularity, the default is to use ``/tmp``, see http://singularity.lbl.gov/build-environment#temporary-folders.
If ``--container-tmpdir`` is specified, the ``$SINGULARITY_TMPDIR`` environment variable will be defined accordingly
to let Singularity use that location instead.


.. _containers_cfg_type:

Type of container recipe/image to generate (``--container-type``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With the ``--container-type`` configuration option, you can specify what type of container recipe/image EasyBuild
should generated. Possible values are:

* ``singularity`` *(default)*: Singularity (https://singularity.lbl.gov) container recipes & images
* ``docker``: Docker (https://docs.docker.com/) container recipe & images

.. note:: Currently (since EasyBuild v3.6.0) only ``singularity`` is actually supported.



.. _containers_stacking:

'Stacking' container images
---------------------------

To avoid long build times and excessive large container images, you can construct your target container image
step-by-step, by first building a base container image for the compiler toolchain you want to use,
and then using it to build a container images for a particular (set of) software package(s).

For example, to build a container image for Python 3.6.4 built with the ``foss/2018a`` toolchain::

    $ cd /tmp

    # use current directory as location for generated container recipes & images
    $ export EASYBUILD_CONTAINERPATH=$PWD

    # build base container image for OpenMPI + GCC parts of foss/2018a toolchain, on top of CentOS 7.4 base image from Singularity Hub
    $ eb -C --container-build-image OpenMPI-2.1.2-GCC-6.4.0-2.28.eb --container-base shub:shahzebsiddiqui/eb-singularity:centos-7.4.1708 --experimental
    ...
    == Singularity image created at /tmp/OpenMPI-2.1.2-GCC-6.4.0-2.28.simg
    ...

    $ ls -lh OpenMPI-2.1.2-GCC-6.4.0-2.28.simg
    -rwxr-xr-x 1 root root 590M Apr 24 11:43 OpenMPI-2.1.2-GCC-6.4.0-2.28.simg

    # build another container image for the for the full foss/2018a toolchain, using the OpenMPI + GCC container as a base
    $ eb -C --container-build-image foss-2018a.eb --container-base localimage:$PWD/OpenMPI-2.1.2-GCC-6.4.0-2.28.simg --experimental
    ...
    == Singularity image created at /tmp/foss-2018a.simg
    ...

    $ ls -lh foss-2018a.simg
    -rwxr-xr-x 1 root root 614M Apr 24 13:11 foss-2018a.simg

    # build container image for Python 3.6.4 with foss/2018a toolchain by leveraging base container image foss-2018a.simg
    $ eb -C --container-build-image Python-3.6.4-foss-2018a.eb --container-base localimage:$PWD/foss-2018a.simg --experimental
    ...
    == Singularity image created at /tmp/Python-3.6.4-foss-2018a.simg
    ...

    $ ls -lh Python-3.6.4-foss-2018a.simg
    -rwxr-xr-x 1 root root 759M Apr 24 14:01 Python-3.6.4-foss-2018a.simg

    $ singularity exec Python-3.6.4-foss-2018a.simg which python
    /app/software/Python/3.6.4-foss-2018a/bin/python

    $ singularity exec Python-3.6.4-foss-2018a.simg python -V
    vsc40023 belongs to gsingularity
    Python 3.6.4
