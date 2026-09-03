Deploying Assemblage on Linux
=============================

This page covers the current code on the ``main`` branch: Linux builders for
C/C++ (gcc, clang) and Rust (``rustc`` with the LLVM, Cranelift and GCC
backends). ``INSTALL.md`` in the repository is the full step-by-step guide
with a verification command for each step; this page is the short version.

Components
----------

Everything runs from one ``docker-compose.yml``:

* **scraper**: searches GitHub in date windows for repositories that have a
  license and a recognized build system, and sends them to the coordinator
  in bundles of 25. ``scraper_0`` searches C/C++, ``scraper_rust`` Rust.
* **coordinator**: records repositories in PostgreSQL, creates one build
  task per repository per build option, and runs one dispatch thread per
  build option that feeds its ``build_opt_{id}`` queue.
* **builders**: containers that restore or clone a repository, build it,
  extract DWARF function and line information, and upload the binaries
  plus ``assemblage_meta.json`` to MinIO. One compose service per build
  option.
* **RabbitMQ** (queues), **PostgreSQL** (metadata), **MinIO**
  (S3-compatible store for source archives and build artifacts).
* **export scripts** on the host that turn the MinIO contents into a
  publishable dataset.

Builders acknowledge a task before building it. A task in progress when its
builder is recreated is lost, not retried.

Requirements
------------

* A Linux host with Docker and Docker Compose v2.
* Disk measured in terabytes. Artifacts dominate; a few thousand
  repositories across the build matrix reach hundreds of GB.
* About 2 GB RAM per builder as a floor. DWARF extraction of large Rust
  binaries has been measured at roughly 42 times the binary size.
* `uv <https://docs.astral.sh/uv/>`_ for host-side scripts and tests
  (Python 3.12).
* A GitHub personal access token with public-repository read scope.

.. warning::
   The compose file publishes PostgreSQL (5432), RabbitMQ (5672) and MinIO
   (9010 API, 9011 console) on the host. Put the host behind a firewall and
   open these ports only to your own worker hosts.

Configure
---------

.. code-block:: bash

   git clone https://github.com/Assemblage-Dataset/Assemblage.git
   cd Assemblage
   cp secrets.env.example secrets.env

Fill in ``secrets.env``; every service loads it.

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - Key
     - Purpose
   * - ``POSTGRES_USER``, ``POSTGRES_PASSWORD``, ``POSTGRES_DATABASE``
     - metadata database; keep the user as ``assemblage``
   * - ``DB_HOST``, ``DB_PORT``
     - ``assemblage-db`` / ``5432`` inside compose
   * - ``GITHUB_TOKEN``
     - scraper API access
   * - ``S3_HOST``, ``S3_HTTPS``
     - ``minio`` / ``false`` inside compose
   * - ``S3_ACCESS_KEY``, ``S3_SECRET_ACCESS_KEY``
     - must match ``MINIO_ROOT_USER`` / ``MINIO_ROOT_PASSWORD``
   * - ``RABBITMQ_USER``, ``RABBITMQ_PASS``
     - default ``guest`` / ``guest``

Build the images
----------------

Images are built locally, not pulled. Expect about ten minutes each on a
cold cache.

.. code-block:: bash

   docker build --build-arg TOOLCHAIN=gcc   -t assemblage-gcc:default   -f docker/worker/Dockerfile .
   docker build --build-arg TOOLCHAIN=clang -t assemblage-clang:default -f docker/worker/Dockerfile .
   docker build --build-arg RUST_TOOLCHAIN=nightly-2026-06-15 \
       -t assemblage-rust:default -f docker/rust/Dockerfile .

The C/C++ image is Ubuntu 24.04 with gcc 13 and clang 18. Both variants
install the same packages; the clang variant exposes the ``gcc`` and ``cc``
names as links to clang so build systems that hardcode them still work. The
Rust image installs one pinned nightly with the Cranelift and GCC codegen
components plus ``rustfilt``. Check that Cranelift is present before
deploying, otherwise every Cranelift builder fails every task:

.. code-block:: bash

   docker run --rm --entrypoint bash assemblage-rust:default -c \
     'rustup component list --toolchain nightly-2026-06-15 | grep -i "cranelift.*installed"'

First boot
----------

Start in this order. Builders that start before RabbitMQ resolves crash-loop
on DNS, and the coordinator must be running before builders register.

.. code-block:: bash

   docker compose up -d database rabbitmq minio
   # wait until: docker inspect -f '{{.State.Health.Status}}' assemblage-db  -> healthy
   docker compose up -d coordinator
   docker compose up -d scraper_0 scraper_rust
   docker compose up -d --no-recreate builder_6 builder_rust_llvm_o2

On a fresh volume the coordinator creates the database and applies the
Alembic migrations itself. After pulling new migrations, apply them with
``docker exec -it assemblage-coordinator-1 alembic upgrade head``.

.. warning::
   A bare ``docker compose up -d`` starts every service in the file: ten
   C/C++ builder services at ten replicas each plus twenty Rust services.
   Name the services you want.

Check that the fleet is live:

.. code-block:: bash

   docker exec assemblage-rabbitmq-1 rabbitmqctl list_queues name messages consumers --quiet \
     | awk '/^build_opt_/ && $3>0 {print; c+=$3} END {print "consumers="c}'
   docker exec assemblage-db psql -U assemblage -d assemblage -t \
     -c "SELECT count(*), max(build_date) FROM binaries;"

Every running builder should be a consumer on its own ``build_opt_{id}``
queue, and ``max(build_date)`` should advance within minutes. Consumers with
zero messages on every queue means the fleet is stranded (see `Failure
modes`_), not idle.

Build matrix
------------

C/C++: ``builder_0`` to ``builder_9`` cover gcc and clang at ``-O0``,
``-O1``, ``-O2``, ``-O3`` and ``-Os`` in build mode ``RelWithDebInfo``
(``-g -DNDEBUG``). The builder detects autotools, CMake or plain Make from
the file list and runs ``make`` with a ten-minute timeout.

Rust: twenty ``builder_rust_*`` services, one per combination of codegen
backend, build mode and optimization flag, all on ``nightly-2026-06-15``
with v0 symbol mangling:

.. list-table::
   :header-rows: 1
   :widths: 20 25 55

   * - Backend
     - Build mode
     - Flags
   * - llvm
     - RelWithDebInfo
     - ``-O0 -O1 -O2 -O3 -Os -Oz``
   * - llvm
     - Debug
     - ``-O0 -O1 -O2``
   * - llvm
     - Release
     - ``-O2 -O3 -Os -Oz``
   * - cranelift
     - RelWithDebInfo
     - ``-O0 -O2``
   * - gcc
     - RelWithDebInfo
     - ``-O0 -O1 -O2 -O3 -Os``

Metadata quality differs by tier. LLVM and Cranelift builds have full
function, line and source mapping. GCC backend builds have correct names and
addresses, but repository-level source file and line information is mostly
missing. Release builds keep repository symbols in ``.symtab`` but carry
little repository DWARF. The six ``llvm``/``RelWithDebInfo`` services also
dump compiler IR (``llvm-ir``, ``mir``, ``asm``, ``hir``, ``thir``) for the
repository's own crates, which adds about 75% to build time.

Each builder registers its compiler, language, backend, build mode and flag
with the coordinator at startup. A new combination becomes a new build
option row with its own queue, and existing repositories are queued for it.
Adding a build option is a matter of adding a compose stanza.

Builder settings (environment variables, per service):

.. list-table::
   :header-rows: 1
   :widths: 34 16 50

   * - Variable
     - Default
     - Meaning
   * - ``BUILDER_REPLICAS``, ``BUILDER_MEM``
     - 10, 16g
     - replicas and memory limit of the C/C++ services
   * - ``RUST_BUILDER_REPLICAS``
     - 2
     - replicas of all twenty Rust services (see Scaling)
   * - ``BUILD_TIMEOUT_S``
     - 1800 (Rust)
     - build wall-clock limit
   * - ``CARGO_BUILD_JOBS``
     - 4
     - cargo parallelism per Rust builder; unbounded cargo uses every host
       core in every container
   * - ``DWARF_SIZE_LIMIT``
     - 150 MB (Rust: 250 MB)
     - binaries above this size skip extraction, with a logged warning
   * - ``DWARF_TIMEOUT_S``, ``DWARF_PHASE_TIMEOUT_S``, ``DWARF_MEM_LIMIT_MB``
     - 300, 900, 8192
     - extraction runs in a child process bounded by these limits; on
       timeout the binary is still stored with an empty function list. The
       Rust Debug services use 900 and 1800.
   * - ``IR_DUMP``, ``IR_STAGES``, ``IR_SCOPE``, ``IR_MAX_BYTES``
     - off, ``llvm-ir,mir``, ``repo``, 512 MB
     - Rust IR dumps

Extraction is single-threaded and is the throughput bottleneck. Its time
follows DWARF complexity rather than file size: a 41 MB Rust binary took 46
minutes of CPU. On one host, adding builders past roughly 32 did not add
throughput; the limit was disk I/O.

Operating the fleet
-------------------

Scaling
~~~~~~~

.. code-block:: bash

   BUILDER_REPLICAS=4 BUILDER_MEM=8g docker compose up -d          # C/C++ defaults
   docker compose up -d --scale builder_6=20 builder_6             # more gcc -O2
   docker compose up -d --no-recreate \
     --scale builder_rust_llvm_o2=2 --scale builder_rust_clift_o2=1 \
     builder_rust_llvm_o2 builder_rust_clift_o2

``RUST_BUILDER_REPLICAS`` applies to all twenty Rust services, including
those you left stopped, so raising it starts services you did not intend to
run. Scale per service and name only the services you want. Scaling a
service to zero removes its containers; its queue then accumulates tasks
until a consumer returns.

Watchdog loop
~~~~~~~~~~~~~

``database`` and ``rabbitmq`` have no restart policy. ``assemblage_loop.sh``
is what brings them back, so it should always be running:

.. code-block:: bash

   mkdir -p var
   nohup setsid flock -n var/loop.lock /bin/bash ./assemblage_loop.sh >/dev/null 2>&1 &

``var/`` must exist before ``flock`` runs, and the script should be invoked
through ``/bin/bash``. Add a ``@reboot`` crontab entry so it survives
reboots, and check ``crontab -l`` after each reboot:

.. code-block:: bash

   @reboot sleep 60 && cd /path/to/Assemblage && \
     /usr/bin/flock -n var/loop.lock /bin/bash ./assemblage_loop.sh >/dev/null 2>&1

The loop only starts missing infrastructure containers; it never restarts a
healthy broker or coordinator. Periodic mass restarts of workers are turned
off: restarting about 50 builders at once caused a two-hour disk I/O storm.

Blocklist
~~~~~~~~~

``backend/blocklist.txt`` lists owners, or ``owner/name`` repositories, that
are never dispatched, one per line, ``#`` for comments. The coordinator
re-reads it about every 30 seconds, so edits need no restart.

Failure modes
~~~~~~~~~~~~~

**Fleet stranded, containers healthy.** The coordinator starts a build
option's dispatch thread only when a builder registers. After the
coordinator restarts on its own, connected builders keep their registrations,
never re-register, and are never dispatched to again. All ``build_opt_*``
queues show consumers but zero messages, and ``max(build_date)`` stops.
Fix: restart the builder services; they re-register in under 20 seconds.
Do not restart the coordinator or RabbitMQ as routine maintenance.
``midnight_recover.sh`` detects this state (builder registrations older
than the coordinator) and restarts the builders; it can run from cron.

**Builders crash-loop after a host reboot.** Infrastructure stays down (no
restart policy) while builders, which are ``unless-stopped``, restart
forever against a RabbitMQ that is not there. Logs show
``socket.gaierror: Temporary failure in name resolution``. Fix:
``docker compose up -d --no-recreate database rabbitmq coordinator``.

**Builds fail on missing system libraries.** Errors naming ``openssl``,
``protoc``, ``glib-2.0``, ``wayland-client``, ``libudev``, ``libclang``,
``cmake``, ``alsa`` or ``dbus`` are image packaging gaps, not bad
repositories. Add the package to ``docker/rust/Dockerfile`` and rebuild.

**A builder sits at 100% CPU for a long time.** DWARF extraction or an IR
dump, bounded by the ``DWARF_*`` limits above.

Storage layout
--------------

MinIO holds two buckets. ``project-archive`` keeps the source checkout of
each repository as ``{owner}/{project}/{sha12}.tar.gz`` with a
``latest.txt`` pointer. ``artifacts`` keeps one directory per build, in the
same shape as the published dataset:

.. code-block:: text

   artifacts/{owner}_{project}_{sha12}-{flag}-{backend}-{mode}/
       binaries/{name}.zst
       metadata/assemblage_meta.json.zst
       ir/{stage}.tar.gz
       export.json

Builds stored before August 2026 use the older uncompressed prefixes
``{owner}_{project}_{sha12}_{compiler}_{flag}/`` and
``..._rustc-{backend}_{mode}_{flag}/``. Readers handle both;
``backend/scripts/backfill_compress.py`` converts old builds in place after
verifying every object by checksum.

Publishing a corpus
-------------------

Run on the host, in order. The scripts read MinIO on the host port (9010)
and join repository URL, commit and license from PostgreSQL, because
builders have no database access.

.. code-block:: bash

   set -a; . ./secrets.env; set +a
   python backend/scripts/export_corpus.py        --out ./assemblage-rust
   python backend/scripts/export_sources.py       --src ./assemblage-rust --out ../assemblage-hf
   python backend/scripts/pack_repos.py           --src ./assemblage-rust --out ../assemblage-hf
   python backend/scripts/refresh_release_meta.py --root ../assemblage-hf
   python backend/scripts/upload_hf.py            --folder ../assemblage-hf --repo <user>/<dataset>

* ``export_corpus.py`` copies each build into one directory. It exports only
  builds whose license is on the permissive allowlist; everything else,
  including unidentified licenses, is skipped before download.
  ``--all-licenses`` turns the gate off for private use.
* ``export_sources.py`` stages the source archive of every exported
  repository as ``sources/{owner}__{name}.tar.gz``.
* ``pack_repos.py`` bundles the builds of each repository into one
  uncompressed tar (Hugging Face rate-limits per file), writes
  ``index.jsonl``, and leaves binaries above 1 GiB uncompressed out of the
  tars. It rebuilds a tar only when its member set changed.
* ``refresh_release_meta.py`` regenerates ``LICENSES.csv`` and the counts
  in the dataset ``README.md`` from ``index.jsonl``.
* ``upload_hf.py`` uploads with ``upload_large_folder``, which resumes on
  its own. Hugging Face rejects a directory with more than 10,000 files;
  move the overflow into ``repos2/`` and ``sources2/`` and run
  ``fix_overflow_paths.py`` to re-point the manifests.

All stages can be interrupted and re-run.

Tests
-----

.. code-block:: bash

   export PATH="$HOME/.local/bin:$PATH"
   uv run pytest tests/                     # unit tests
   uv run pytest tests/ -m integration      # needs: docker compose up -d database
   make e2e                                 # golden-repo end-to-end gate (C and Rust)

Queue names, wire JSON (``tests/fixtures/messages/``), metadata keys and the
database schema are frozen. Migrations are handwritten; never commit
``alembic revision --autogenerate`` output.

DeepHistory on Linux
--------------------

DeepHistory builds fixed releases of C/C++ libraries listed in
``deephistory/packages.json`` (1,155 package versions, each with GitHub
URL, commit and license). The Linux builder is standalone: no coordinator
or RabbitMQ, only MinIO.

.. code-block:: bash

   docker compose up -d minio
   docker compose -f compose/deephistory.yml up --build -d
   docker compose -f compose/deephistory.yml up -d --scale gcc-O2=8

Each service is one compiler and flag (``COMPILER`` gcc or clang,
``COMPILER_FLAG`` ``-O0`` to ``-O3``). Progress is kept under
``deephistory/`` so runs can resume. The Windows/MSVC side of DeepHistory
uses Conan and is described on the Windows page.

Rebuilding from a recipe (legacy)
---------------------------------

The ``linux_github`` branch (last updated May 2024) could rebuild a dataset
from a JSON recipe of repositories through the ``loadrepo`` command of its
``cli.py``. The recipe for the Linux dataset is still available:
:download:`linux_recipe.zip <assets/linux_recipe.zip>`. Repositories may
since have been deleted or changed, and each one needs a full clone with
history, so the result is not identical. The current code has no recipe
loader.
