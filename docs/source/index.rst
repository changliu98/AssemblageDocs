Assemblage documentation
========================

Assemblage builds large, labeled corpora of binaries from open-source code.
It searches GitHub for licensed C/C++ and Rust repositories, builds each one
with several compilers and optimization levels, and stores the resulting
binaries together with function-level and line-level debug metadata. The
corpora are intended as training and evaluation data for machine learning on
binaries, and for static analysis, dynamic analysis and reverse engineering
research.

* Code: https://github.com/Assemblage-Dataset/Assemblage (MIT license)
* Paper: https://arxiv.org/abs/2405.03991
* DeepHistory paper: https://arxiv.org/abs/2605.21615
* Project site: https://assemblage-dataset.net

The current code base builds Linux ELF binaries: C/C++ with gcc and clang,
and Rust with the LLVM, Cranelift and GCC code generation backends of
``rustc``. The Windows/MSVC and vcpkg workers that produced the Windows PE
datasets are kept on separate branches and in a frozen ``legacy`` package;
their pages say so.

Contents
--------

.. toctree::

   dataset
   deployment_linux
   deployment_windows
   deployment_vcpkg
   api_reference

Contact
-------

For dataset access, deployment help or other questions, email the current
maintainers:

| Kristopher Micinski: kkmicins@syr.edu
| Chang Liu: cliu57@syr.edu

Contributors, by last name:

| Naveen Ashok: nashok@syr.edu
| Alex Duly: apduly@syr.edu
| Maya Fuchs: fuchs_maya@bah.com
| James Holt: holt@lps.umd.edu
| Mia Kerchen: mhkerche@syr.edu
| Chang Liu: cliu57@syr.edu
| Kristopher Micinski: kkmicins@syr.edu
| Townsend Southard Pantano: tgsoutha@syr.edu
| Edward Raff: Raff.Edward@gmail.com
| Rebecca Saul: Saul_Rebecca@bah.com
| Yihao Sun: ysun67@syr.edu
