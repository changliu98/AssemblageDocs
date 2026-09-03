Windows builders (legacy)
=========================

.. note::
   The Windows/MSVC build path is frozen. On ``main`` it lives in
   ``backend/assemblage/legacy/windows`` and ``docker/legacy``, is excluded
   from the test gates, and is used only when a builder runs with
   ``COMPILER=MSVC``. The datasets built with it (Windows GitHub, vcpkg and
   the Windows half of DeepHistory) are published; new development targets
   Linux.

There are two ways to run it.

Option A: Windows container against a Linux coordinator
-------------------------------------------------------

``compose/windows.yml`` on ``main`` builds ``docker/legacy/windows/Dockerfile``
and runs ``backend/scripts/start_windows_worker.ps1``, which loads the
Visual Studio developer environment, registers the DIA SDK DLLs and starts
``start_worker.py`` with ``TYPE=builder``, ``COMPILER=MSVC`` and
``LANGUAGE=c++``.

The image is Windows Server Core LTSC 2022 with Visual Studio 2022 Build
Tools (VC tools, MSBuild, Windows 10 SDK, CMake), Python 3.12, Git,
Universal Ctags and the ``Dia2Dump`` tool shipped in the repository.

Requirements:

* A Windows Docker host running Windows containers. Windows images cannot
  be redistributed, so the image is built from Microsoft's installers and
  the first build takes a long time.
* The Linux coordinator, with RabbitMQ (5672) and MinIO reachable from the
  Windows host. Set ``MQ_HOST`` and ``S3_HOST`` in
  ``secrets.env`` to that host.

.. code-block:: powershell

   docker compose -f compose/windows.yml up --build -d

The container mounts ``backend`` at ``C:\app`` and ``binaries`` at
``C:\binaries``. Registration, task flow and reporting are the same as for
Linux builders; MSVC builders register with ``compiler=MSVC``. This path is
not covered by any automated test, and the frozen strategy has a known
signature mismatch with the current builder pipeline, so expect to fix code
before production use.

Option B: the ``windows_github`` branch
---------------------------------------

The branch that produced the Windows GitHub dataset (last commit May 2024)
is an older code base with a gRPC coordinator on port 50052, MySQL and JSON
configuration files. Its worker runs directly on a Windows machine:

#. Install Python 3.9 or newer, Git, Visual Studio with the MSVC build
   tools, CMake, 7zip, Dia2dump and Universal Ctags. Put ``python``,
   ``msbuild``, ``ctags``, ``readtags``, ``dia2dump`` and ``7z`` on
   ``PATH``.
#. Register the DIA SDK DLL as administrator:

   .. code-block:: bat

      regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\DIA SDK\bin\msdia140.dll"
      regsvr32 "C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildCommunityTools\DIA SDK\bin\amd64\msdia140.dll"

#. Check out the branch and install the packages its modules import (the
   branch has no requirements file):

   .. code-block:: bat

      git clone https://github.com/Assemblage-Dataset/Assemblage.git
      cd Assemblage
      git checkout windows_github

#. Write ``assemblage/configure/worker_config.json`` with the coordinator
   address and credentials, then run
   ``python start_worker.py --config assemblage/configure``.

The matching coordinator is the same branch's ``docker compose up`` with
``coordinator_config.json`` and ``crawler_config.json``. Task Scheduler can
start the worker at boot on cloud instances; helper scripts are under
``script``.

That branch can also rebuild a dataset from a recipe through the
``loadrepo`` command of its ``cli.py``:
:download:`sept25.json.zip <assets/sept25.json.zip>` and
:download:`winpe_recipe.zip <assets/winpe_recipe.zip>`. Repositories may
since have been deleted or changed, and rebuilding the same source does not
produce byte-identical binaries.

DeepHistory on Windows (Conan)
------------------------------

``backend/scripts/build_deephistory.py`` builds several released versions
of each library with Conan and MSVC on a Windows host, driven by
``backend/assemblage/legacy/deephistory_manifest.json``:

.. code-block:: powershell

   python backend/scripts/build_deephistory.py --manifest backend/assemblage/legacy/deephistory_manifest.json
   python backend/scripts/build_deephistory.py --packages sqlite3 fmt zlib
   python backend/scripts/build_deephistory.py --manifest backend/assemblage/legacy/deephistory_manifest.json --resume

Output is one folder per package build containing ``assemblage_meta.json``
and the DLL, EXE and PDB files, ready for the dataset CLI
(``assemblage-dataset -g --data <out> --dbfile deephistory.sqlite --functions --lines --rvas --pdbs``).
``docker/legacy/conan/Dockerfile`` packages the same toolchain as a Windows
container, and ``TYPE=legacy_conan python backend/scripts/start_worker.py``
runs the builder without a coordinator.
