API Reference and Customization
===============================

This page describes the Python package on ``main`` (``backend/assemblage``,
Python 3.12) and the contracts to keep when extending it.

Package layout
--------------

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - Module
     - Responsibility
   * - ``enums.py``, ``constants.py``, ``settings.py``
     - enumerations; queue names and intervals; settings read from
       environment variables (pydantic-settings)
   * - ``messages.py``
     - the RabbitMQ messages (pydantic v2); their JSON shape is frozen
   * - ``runtime/``
     - ``Service`` base class and ``Supervisor``: one thread per service,
       restart with backoff, clean shutdown on SIGTERM
   * - ``mq/``
     - topology, connection retry, consumer (one ack or nack per message),
       publisher with confirms
   * - ``db/``
     - SQLModel models matched to the live PostgreSQL schema, the
       coordinator store, first-boot database creation
   * - ``storage/``
     - S3 client; ``layout.py`` defines every object key; ``compress.py``
       (zstd)
   * - ``build/``
     - command runner, build-system detection, binary discovery,
       ``BuildStrategy`` and its factory, ``linux.py``, ``rust.py``
   * - ``dwarf/``
     - the DWARF extractor (pyelftools) and its out-of-process wrapper
   * - ``coordinator/``, ``builder/``, ``scraper/``
     - the three workers; ``builder/ir.py`` handles Rust IR dumps
   * - ``dataset/``
     - host-side dataset tools (``assemblage-dataset``, ``assemblage-daily``)
   * - ``legacy/``
     - frozen Windows/MSVC strategy and the Conan DeepHistory builder

Workers
-------

``backend/scripts/start_worker.py`` is the entry point of every container.
``TYPE`` selects ``coordinator``, ``builder``, ``scraper`` or
``legacy_conan``. Everything else is environment variables, read by
``settings.py``:

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - Variable
     - Used by
   * - ``MQ_HOST``, ``MQ_PORT``, ``RABBITMQ_USER``, ``RABBITMQ_PASS``
     - all workers
   * - ``DB_HOST``, ``DB_PORT``, ``POSTGRES_DATABASE``, ``POSTGRES_USER``,
       ``POSTGRES_PASSWORD``
     - coordinator
   * - ``S3_HOST``, ``S3_PORT``, ``S3_HTTPS``, ``S3_ACCESS_KEY``,
       ``S3_SECRET_ACCESS_KEY``
     - coordinator, builder
   * - ``compiler``, ``language``, ``COMPILER_FLAG``, ``CODEGEN_BACKEND``,
       ``BUILD_MODE``
     - builder identity; together they name the build option
   * - ``BUILD_TIMEOUT_S``, ``DWARF_SIZE_LIMIT``, ``DWARF_TIMEOUT_S``,
       ``DWARF_PHASE_TIMEOUT_S``, ``DWARF_MEM_LIMIT_MB``, ``IR_DUMP``,
       ``IR_STAGES``, ``IR_SCOPE``, ``IR_MAX_BYTES``, ``CARGO_HOME``
     - builder limits
   * - ``GITHUB_TOKEN``, ``SCRAPE_QUALIFIERS``, ``SCRAPE_INTERVAL``,
       ``SCRAPE_START_TIME``, ``SCRAPE_END_TIME``
     - scraper; ``SCRAPE_QUALIFIERS`` adds GitHub search qualifiers such as
       ``language:rust``
   * - ``BLOCKLIST_PATH``
     - coordinator; default ``/app/blocklist.txt``

Message flow
------------

#. The scraper publishes bundles of 25 repositories on the ``scrape`` queue.
#. The coordinator inserts them into ``projects`` and creates one
   ``b_status`` row per repository per build option.
#. One dispatch thread per build option publishes tasks to
   ``build_opt_{id}`` (topic exchange ``build_opt``). The thread starts when
   a builder registers on ``builder_reg`` with that build option.
#. Builders acknowledge a task before building it, then report on the
   ``clone``, ``build`` and ``binary`` queues. Rust builders with IR dumps
   also report on ``ir``.

Queue names and message JSON are frozen; the golden files under
``tests/fixtures/messages/`` are the specification. Enum values are
lowercase on the wire (``"success"``) and stored by name in the database
(``'SUCCESS'``).

Build strategies
----------------

A builder's behaviour is a ``BuildStrategy`` (``build/strategy.py``). The
pipeline calls ``prepare``, ``build``, ``find_binaries`` and ``debug_info``
in that order and handles source acquisition, uploads and reporting itself.

.. code-block:: python

   class BuildStrategy(ABC):
       platform: str                 # "linux"
       compiler: str                 # "gcc", "clang", "rustc"
       language: str                 # "c++", "rust"
       compiler_version: str | None
       toolset_version: str | None
       build_mode: str               # "RelWithDebInfo", "Debug", "Release"
       base_path: str                # where repositories are cloned and built

       def prepare(self, clone_dir: str, compiler_flag: str) -> object | None:
           """Pre-build configuration; the return value is passed to build()."""

       def build(self, clone_dir: str, compiler_flag: str,
                 prepared: object | None) -> tuple[str, BuildStatus]:
           """Compile; return (combined output, status)."""

       def find_binaries(self, path: str) -> set[str]:
           """Paths of the built binaries under path."""

       def debug_info(self, clone_dir: str,
                      original_files: list[str]) -> list[dict[str, object]]:
           """One Binary_info_list entry per new binary."""

       def own_dir(self, path: str) -> None:
           """Optional ownership fix-up of produced directories."""

``make_strategy(settings)`` picks the implementation: ``LinuxBuildStrategy``
for C/C++ (autotools, CMake or Make detected from the file list; ``-g
-DNDEBUG`` plus the optimization flag), ``RustBuildStrategy`` when
``language`` is ``rust``, and the frozen Windows strategy when the platform
is Windows.

``RustBuildStrategy`` drives ``cargo`` through per-profile environment
variables, selects the backend through a ``RustCodegenAdapter`` (``llvm``,
``cranelift``, ``gcc``), reads cargo's JSON output to find the workspace's
own ``bin``, ``example`` and ``cdylib`` artifacts, and adds a demangled name
(``rustfilt``) and an ``origin`` tag (``in_repo``, ``dependency``,
``stdlib``, ``other``) to every extracted function. Each adapter declares
the debug information and IR stages its backend can produce
(``DebugInfoCaps``, ``IrCaps``).

Adding a build target
~~~~~~~~~~~~~~~~~~~~~

#. Add the enum members in ``enums.py`` (``SupportedCompiler``,
   ``SupportedLanguage`` and so on).
#. Implement ``BuildStrategy`` under ``build/`` and register it in
   ``make_strategy``.
#. Make sure ``builder/artifacts.py`` handles the new outputs.
#. Add a compose service with ``TYPE=builder`` and the identity variables.
   It registers its build option at startup, and existing repositories are
   queued for it.

Adding an optimization level or build mode for an existing target needs
only the last step.

Metadata
--------

Every build produces ``assemblage_meta.json``. The key set is frozen; Rust
builds add keys.

.. code-block:: text

   Platform, Build_mode, Compiler, Compiler_version, URL, Commit, Optimization,
   Pushed_at, compiler_flag, language, library
   Codegen_backend, Toolchain, Mangling, Backend_caps, Cargo_locked      (Rust only)
   Binary_info_list: [
     { file,
       functions: [
         { function_name, source_file, intersect_ratio,
           demangled_name, origin,                                        (Rust only)
           function_info: [ { rva_start, rva_end } ],
           lines: [ { line_number, rva, length, source_code, source_file } ] } ] } ]

``Binary_info_list`` comes from ``dwarf/extract.py``, the same extractor the
dataset tools use. Extraction of a binary can be skipped
(``DWARF_SIZE_LIMIT``) or stopped (timeouts); the binary is still stored and
its entry is then empty.

Storage keys
------------

All S3 keys come from ``storage/layout.py``:

.. code-block:: text

   project-archive/{owner}/{project}/{sha12}.tar.gz
   project-archive/{owner}/{project}/latest.txt
   artifacts/{owner}_{project}_{sha12}-{flag}-{backend}-{mode}/binaries/{name}.zst
   artifacts/{owner}_{project}_{sha12}-{flag}-{backend}-{mode}/metadata/assemblage_meta.json.zst
   artifacts/{owner}_{project}_{sha12}-{flag}-{backend}-{mode}/ir/{stage}.tar.gz
   artifacts/{owner}_{project}_{sha12}-{flag}-{backend}-{mode}/export.json

``{backend}`` is the Rust codegen backend or the C compiler; ``{flag}`` has
no leading dash; ``{sha12}`` is the first 12 characters of the commit.
Compression is zstd level 12, matching the published dataset, so a release
is a copy rather than a recompression.

Database
--------

PostgreSQL tables: ``projects``, ``b_status`` (one row per repository per
build option), ``buildopt``, ``binaries``, ``ir_artifacts`` (one row per
build and IR stage) and ``scrapers``. A build option row is identified by
nine columns, including platform, compiler, language, codegen backend,
build mode and flag. ``binaries.optimization`` is always empty; join
``binaries`` to ``b_status`` and ``buildopt`` to recover compiler and flag.
Migrations are Alembic and handwritten; apply them with
``docker exec -it assemblage-coordinator-1 alembic upgrade head``.

Dataset tools
-------------

.. code-block:: bash

   uv sync --extra dataset
   uv run assemblage-dataset -g --data <folder of builds> --dbfile out.sqlite --functions --rvas --lines
   DB_HOST=localhost MINIO_ENDPOINT=localhost:9010 uv run assemblage-daily [--since YYYY-MM-DD]

``assemblage-dataset -g`` turns a folder of build directories, each with an
``assemblage_meta.json``, into the five-table SQLite database described on
the dataset page. ``assemblage-daily`` pulls builds newer than a date from
MinIO and PostgreSQL, re-extracts their DWARF and appends them to a
cumulative ``linux_licensed.sqlite`` under ``assemblage_dataset/``; the
SQLite predecessor of the Linux GitHub dataset was produced this way. The
Rust dataset is produced by the export scripts on the deployment page.
