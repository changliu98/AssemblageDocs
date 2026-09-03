vcpkg worker (legacy)
=====================

.. note::
   The vcpkg worker lives on the ``windows_vcpkg`` branch (last commit July
   2024) and is not part of the current code base. It produced the Windows
   vcpkg DLL dataset.

`vcpkg <https://vcpkg.io>`_ is a package manager for C++ libraries. The
worker builds its ports on Windows with MSVC and collects the DLL and PDB
files. The catalogue is small (about 2,000 ports) and each build is heavy,
so the worker is a standalone script with no coordinator or queue.

Machine setup
-------------

Tested on Windows Server 2022; Windows 10 and 11 should behave the same.
Use at least 4 cores, 16 GB of RAM and 1 TB of disk.

#. Install Python 3.9 or newer, Git, Visual Studio, CMake, Universal Ctags,
   Dia2dump, 7zip and vcpkg
   (https://github.com/microsoft/vcpkg#quick-start-windows).
#. Register the DIA SDK DLL as administrator:

   .. code-block:: bat

      regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\DIA SDK\bin\msdia140.dll"
      regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildCommunityTools\DIA SDK\bin\amd64\msdia140.dll"

#. Make sure ``python``, ``git``, ``ctags``, ``readtags``, ``dia2dump``,
   ``7z`` and ``vcpkg`` are on ``PATH``.

Run
---

.. code-block:: bat

   git clone https://github.com/Assemblage-Dataset/Assemblage.git
   cd Assemblage
   git checkout windows_vcpkg
   python worker_lite.py

The branch has no requirements file; install the packages ``worker_lite.py``
and the ``assemblage`` package import. ``worker_lite.py`` reads
``projects.json`` (package name to list of versions) from the working
directory and builds each entry in release mode with toolset ``v141`` for
x64. Change the constants at the top of the script for other
configurations.
