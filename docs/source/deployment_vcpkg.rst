Deployment of vcpkg Worker
==========================


.. autosummary::
   :toctree: generated

About vcpkg
-----------

`vcpkg <https://vcpkg.io>`_ is a package manager, just like apt or yum, but for C++ libraries. 
It has integration with Visual Studio, and it is cross-platform, but we will primarily use it on Windows to collect Dynamic link library (DLL) files
Because building these dll files are extremely computing intensive, and the projects hosted are limiting (~2000 projects), there is no need for high concurrency for our system.
And vcpkg worker are implemented as standlone python scripts, so it is very easy to deploy.

Worker environment setup
------------------------

We are using Windows Server 2022 Base, but deployment to Windows 10 or Windows 11 should be similar.

    .. note::

        Building vcpkg packages take up a lot of CPU resource and disk space, 
        it is recommended to use a machine with at least 4 cores, 16GB of RAM, and 1TB of disk space.

#. Please install these software on the worker machine:

    * Python 3.9+
    * Git
    * Visual Studio (any version you want to use)
    * CMake
    * Universal Ctags
    * Dia2dump
    * 7zip

#. Register dll file for dia2dump

    .. code-block:: bash

        # Need administator pivilage
        regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\DIA SDK\bin\msdia140.dll"
        regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildCommunityTools\DIA SDK\bin\amd64\msdia140.dll"

#. Install vcpkg

    Please follow this link https://github.com/microsoft/vcpkg?tab=readme-ov-file#quick-start-windows to setup vcpkg properly

#. Config system path, and make sure these commands are available in the command line:

    * python
    * git
    * readtags
    * ctags
    * dia2dump
    * 7z
    * vcpkg

#. Clone Assemblage and switch to vcpkg branch

    .. code-block:: bash

        git clone git@github.com:Assemblage-Dataset/Assemblage.git
        cd Assemblage
        git checkout windows_vcpkg

#. Install python dependencies

    .. code-block:: bash

        pip install -r requirements.txt


#. Run the worker

    .. code-block:: bash

        python dryworker.py

