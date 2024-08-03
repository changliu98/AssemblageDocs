Deployment on Windows
=====================


.. autosummary::
   :toctree: generated

Intro
--------

While deployment on Linux system is more convenient, the deployment on Windows is more complicated due to the nature of Windows system, that can't be packed as images for distribution.
To deoloy Assemblage and harvest , you need to deploy the coordinator to a server, then set up workers on Windows instances.


Coordinator Setup
-----------------

Please make sure these are installed, or you have access to:

#. Docker
#. Docker Compose
#. Git
#. Port 5672, 50052
#. A GitHub account, and a personal access token

.. warning::
    You should put you server under firewall and limit the access to these ports, also make sure these ports are accessile by your worker instances

.. note::
    By default, only repositories that have licenses will be used to build binaries

#.  Clone the repository
    
        .. code-block:: bash
    
            git clone git@github.com:Assemblage-Dataset/Assemblage.git
            cd Assemblage


#.  Install local dependencies, and schange the crawler, coordinator configurations, which is locating under `/assemblage/configure/` folder. 
    You can also change the location of these config files, just remember to also update the `docker-compose.yml` file

        .. code-block:: bash

            mkdir /assemblage/configure
            nano assemblage/configure/coordinator_config.json
            nano assemblage/configure/crawler_config.json
            pip install -r requirements.txt

#.  Build the docker image

        .. code-block:: bash

            sh build.sh
            docker compose up -d



Worker Setup   
------------

Windows worker requires many software to be installed, and the installation is not as simple as Linux worker. 
Here is the step-by-step guide to set up a Windows worker

#. Install/build the software:

    #. Python 3.9+
    #. Git
    #. MSVC Build Tools
    #. Microsoft Visual Studio
    #. CMake
    #. 7zip
    #. Dia2dump
    #. Universal Ctags

#. Make sure these executables are added to the PATH

    #. ctags
    #. dia2dump
    #. 7z
    #. readtags
    #. msbuild
    #. python

#. Register dll file for dia2dump

    .. code-block:: bash

        # Need administator pivilage
        regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\DIA SDK\bin\msdia140.dll"
        regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildCommunityTools\DIA SDK\bin\amd64\msdia140.dll"

#. Clone the repository

    .. code-block:: bash

        git clone git@github.com:Assemblage-Dataset/Assemblage.git
        cd Assemblage
        git checkout windows_github

#. Install the dependencies
    
        .. code-block:: bash
    
            pip install -r requirements.txt

#. Change the worker configuration, which is locating under `/assemblage/configure/` folder, examples are provided in the repository

    .. code-block:: bash

        mkdir /assemblage/configure
        nano assemblage/configure/worker_config.json

#. Run the worker

    .. code-block:: bash

        python start_worker.py --config assemblage/configure

.. note::
    
    You can create boot up tasks using Task scheduler, to start the worker automatically when the system starts, 
    which is very useful for scaling up the workers on cloud instances. Some scripts are provided under `script` folder


Optional: Recover dataset
-------------------------

    Assemblage can recover the state from previous running state and remake the binary dataset from the last state, which can be useful
    if the binary itself can not be distributed. To reload the previous state, grab some of the following recipe(in JSON format), and 
    boot up the CLI, navigate to `loadrepo` option, and provide the JSON file, system will build the dataset from the provided file.

    :download:`sept25.json.zip <assets/sept25.json.zip>`
    :download:`winpe_recipe.zip <assets/winpe_recipe.zip>`


    .. warning::
        The previous state is not guaranteed to be the same as the current state, as the repository may have been hidden/deleted, some binaries might not be recovered.
        Meanwhile, to accurately recover dataset from source code, a **full git clone** with all history will be performed, which will be extremely slow and resource consuming

