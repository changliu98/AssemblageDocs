deployment_linux
================


.. autosummary::
   :toctree: generated

Intro
--------

Assemblage provides range of builder and deployment tools to help you deploy it to your system, either on your local machine or on a server.
This section will guide you through the process of deploying Assemblage to a Debian based system.

The system is made from 3 main components: 

#. Coordinator, where tasks being packed and dispatched
#. Worker, where tasks are being executed and binaries are being built
#. DB, the database to store records

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

1.  Clone the repository
    
        .. code-block:: bash
    
            git clone git@github.com:Assemblage-Dataset/Assemblage.git
            cd Assemblage

2.  Modify the cluster file

    You can find examples located under `example_workers`. We will use `example_cluster.py` as example,
    you can declare the workers, crawlers and the database connection in this file.


3.  Install local dependencies, and start local clusters. The cluster file will create Docker images and boot up containers for you.

            .. code-block:: bash
    
                pip install -r requirements.txt
                PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python python3 example_cluster.py


Optional: Recover dataset
-------------------------

    Assemblage can recover the state from previous running state and remake the binary dataset from the last state, which can be useful
    if the binary itself can not be distributed. To reload the previous state, grab some of the following recipe(in JSON format), and 
    boot up the CLI, navigate to `loadrepo` option, and provide the JSON file, system will build the dataset from the provided file.
    Please note, to recover a dataset, it's not using our API version, please switch to the branch linux_github.

    :download:`linux_recipe.zip <assets/linux_recipe.zip>`


    .. warning::
        The previous state is not guaranteed to be the same as the current state, as the repository may have been hidden/deleted, some binaries might not be recovered.
        Meanwhile, to accurately recover dataset from source code, a **full git clone** with all history will be performed, which will be extremely slow and resource consuming

