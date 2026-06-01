Dataset Access
===============

.. autosummary::
   :toctree: generated

Overview
--------

Assemblage provides datasets on various build configurations, such as CPU arch, compiler, and optimization flags.
Due to the nature of different tool chains, the datasets are distributed mainly based on the source and tool chain, such as vcpkg on Windows, and Github on Linux.

.. image:: assets/pipeline.png
  :width: 500
  :alt: Dataset generation pipeline

.. note::
   Currently we are only releasing binaries built from repositories that have license. 
   Please adhere to the license of the original repositories when using the dataset.

Distribution Format
---------------------

The dataset is distributed in the following format:

#. A compressed file of the binaries
#. A SQLite database containing the metadata (GitHub url, functions address, source codes, etc.) of the binaries

.. image:: assets/sqlite_schema.png
  :width: 800
  :alt: SQLite database schema

You can find the detailed schema by querying database. 
The database provides detailed information about the binaries, the compressed file contains the binaries themselves. 
The binaries are stored in the location indicated by the ``path`` field in the database.


If you are not satisfying with SQLite's querying speed (which is slow compared to other database), you can also dump the database into SQL, then load into 
your preferred database solution.

.. code-block:: sql

   .output assemblage.sql
   .dump
   .quit


Major Changes
---------------

#. Linux dataset GCC -Oz optimization issue (November 2025). 
   An issue has been identified with the `-Oz` flag in the Linux dataset: binaries labeled as being built with `-Oz` may have been compiled with `-Os` or with the compiler flags defined in the repository's original Makefile. The dataset has been updated with newly generated GCC `-O2` binaries, other flags are not impacted.

#. Introduce Deephistory dataset (May 2026). 
   We have introduced a new dataset, which contains binaries built from repositories with a long history of commits. This dataset is designed to facilitate research on binary evolution and testing.

Dataset Access
----------------

The dataset is hosted on Hugging Face.
Due to file size limit, we are deprecating the dataset hosting on Kaggle.

#. Windows GitHub dataset (88k, last update 2025 May):

   https://huggingface.co/datasets/changliu8541/Assemblage_PE
   

#. Windows vcpkg dataset (130k, last update 2024 June):

   https://huggingface.co/datasets/changliu8541/Assemblage_vcpkgDLL


#. Linux GitHub dataset (250k, last updated 2026 Apr):

   https://huggingface.co/datasets/changliu8541/Assemblage_LinuxELF


#. Deep History dataset (73k, last updated 2026 May):

   https://huggingface.co/datasets/changliu8541/assemblage-deephistory
