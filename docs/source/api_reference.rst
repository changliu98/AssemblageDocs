API Reference and Customizations
================================


.. autosummary::
   :toctree: generated

Overview
--------

While Assemblage provides a range of functionalities out of the box, 
you may want to customize the behavior of Assemblage by providing your own configurations, 
or implement your own classes of the Assemblage API.

If you want to use the Linux worker, simply extend the class `BuildStartegy`. There are example 
implementations in the `example_workers` folder. Then the booting of Docker containers are handled
automatically.

The deployment of Windows is more complicated, please follow the `Deployment on Windows` guide to setup the 
environment. Then, based on the task's computing requirements, you can decide whether you want to also boot the
coordinator, or use the standalone worker.

.. note::
    The coordinator is responsible for distributing tasks to workers, and managing the task queue. 
    The worker is responsible for executing the tasks, and reporting the results back to the coordinator, While
    there are other ways worker can get the tasks, such as reading from a database or some file.


Classese and Functions
----------------------

For Linux
~~~~~~~~~~~~

The class `BuildStartegy` defines the behavior of the worker. You can extend this class to implement your own.
Then, provide this class to the `builder` function to `AssmeblageCluster`.



.. code-block:: python

   class SampleBuild(BuildStartegy):

      def clone_data(self, repo):
         clonedir = os.urandom(8).hex()
         out, err, exit_code = cmd_with_output(f'git clone {repo["url"]} {clonedir}', 600, "linux")
         return_code = BuildStatus.SUCCESS if exit_code == 0 else BuildStatus.FAILED
         return out, return_code, clonedir



      def run_build(self, repo, target_dir, compiler_version,
                     library, build_mode,
                     optimization, platform, slnfile):
         """ how to constuct a build command  """
         files = []
         for filename in glob.iglob(target_dir + '**/**', recursive=True):
               files.append(filename.split("/")[-1])
         logging.info("%s files in repo", len(files))
         build_tool = get_build_system(files)
         cmd = f'cd {target_dir} && make -j16'
         logging.info("Linux cmd generated: %s", cmd)
         logging.info("Files found %s", os.listdir(target_dir))
         out, err, exit_code = cmd_with_output(cmd, 600, platform)
         return_code = BuildStatus.SUCCESS if exit_code == 0 else BuildStatus.FAILED
         return out.decode() + err.decode(), return_code

      def post_build_hook(self, dest_binfolder, build_mode, library, repoinfo, toolset,
                           optimization, commit_hexsha):
         logging.info(os.listdir(dest_binfolder))
         logging.info("Maybe move files to some Docker mapped volume")
         os.system(f"mv {dest_binfolder} /binaries/{repoinfo['name']}")

   test_cluster_c = AssmeblageCluster(name="sample"). \
                  aws(aws_profile). \
                  docker_network("assemblage-net", True). \
                  message_broker(). \
                  mysql(). \
                  scraper([github_c_repos]). \
                  build_option(
                     1, platform="linux", language="c++", 
                     compiler_name="clang",
                     build_system="all"). \
                  builder(
                     platform="linux", compiler="clang", build_opt=1,
                     custom_build_method=SampleBuild(),
                     aws_profile= aws_profile). \
                  use_new_mysql_local()


For Windows
~~~~~~~~~~~~~~

Similar to the Linux worker, you can extend the class `BuildStartegy` to implement your own behavior. 
Then, provide this class to the `builder` function to `AssmeblageCluster`.

Meanwhile, a seperated `StandaloneBuilder` is provided to handle some complex build processes without coordinator, 
such as the tasks that require complicated configuration to finish rather than calling `MSBuild`.

.. code-block:: python

   class ZLib(BuildStartegy):
      def run_build(self, repo, target_dir, build_mode, library, optimization,
                        slnfile, platform, compiler_version):
         cmd = f"cd {target_dir} && cmake  -G \"Visual Studio 16 2019\" -A x64 -DCMAKE_BUILD_TYPE=Release ."
         out, err, ret = BuildStartegy.cmd_with_output(cmd, 600, platform)
         if ret != 0:
               return out, err, ret
         cmd = f"cd {target_dir} && cmake --build . --config Release"
         return BuildStartegy.cmd_with_output(cmd, 600, platform)
      
      def is_valid_binary(self, binary_path):
         if binary_path.endswith(".pdb") or binary_path.endswith(".dll"):
               return 1
         return 0

