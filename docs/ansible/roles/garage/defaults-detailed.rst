.. Copyright (C) 2026 Patryk Ściborek <patryk@sciborek.com>
.. Copyright (C) 2026 DebOps <https://debops.org/>
.. SPDX-License-Identifier: GPL-3.0-only

.. _garage__ref_defaults_detailed:

Default variable details
========================

.. include:: ../../../includes/global.rst

Some of ``debops.garage`` default variables have more extensive
configuration than simple strings or lists, here you can find
documentation and examples for them.

.. only:: html

   .. contents::
      :local:
      :depth: 1


.. _garage__ref_binary_source:

Binary source selection (waterfall)
-----------------------------------

The role evaluates the following sources, in order, and uses the first
one that is configured. A non-empty
:envvar:`garage__local_binary_path` or
:envvar:`garage__controller_binary_path` wins even if the file is
missing on disk; there is no fall-through to the next source.

#. ``garage__skip_install: True`` - the role does not touch
   :envvar:`garage__bin_path`. Useful when the binary is provided by
   a Packer-built image or an external package.

#. The binary at :envvar:`garage__bin_path` already reports the
   matching :envvar:`garage__version` (parsed from
   ``garage --version``). The install stage short-circuits with no
   downloads.

#. :envvar:`garage__local_binary_path` (non-empty) points at a binary
   already on the remote host. The role still verifies its SHA256
   before installing it.

#. :envvar:`garage__controller_binary_path` (non-empty) points at a
   binary on the Ansible Controller. The role copies it into
   :envvar:`garage__cache_dir` with :command:`ansible.builtin.copy`,
   then verifies and installs it.

#. None of the above: the role downloads
   :envvar:`garage__release_url`
   with :command:`ansible.builtin.get_url`. Inline checksum
   verification uses :envvar:`garage__binary_sha256`.

When the installed version differs from :envvar:`garage__version`, the
upgrade gate documented in :ref:`garage__ref_guide_upgrades` runs
before any file is replaced.


.. _garage__ref_cluster_secrets:

Cluster-wide secrets
--------------------

``rpc_secret`` must be identical on every node. The role looks the
value up from:

:file:`secret/garage/clusters/{garage__cluster_name}/rpc_secret`

The path does not include ``inventory_hostname``. The admin token,
metrics token and the optional default access key use the same
directory. Changing :envvar:`garage__cluster_name` creates a new
secret namespace; nodes with different names will not form a cluster.

The role asserts that every host in ``[debops_service_garage]`` uses
the same :envvar:`garage__cluster_name` and
:envvar:`garage__replication_factor`.


.. _garage__ref_single_node:

Single-node bootstrap
---------------------

:envvar:`garage__single_node` defaults to ``True`` only when
:envvar:`garage__replication_factor` is ``1`` and the host is the
only member of ``[debops_service_garage]``. The systemd unit then
starts Garage with ``--single-node``.

When :envvar:`garage__default_bucket` is non-empty, the unit also
gets ``--default-bucket`` and an ``EnvironmentFile`` with
``GARAGE_DEFAULT_ACCESS_KEY``, ``GARAGE_DEFAULT_SECRET_KEY`` and
``GARAGE_DEFAULT_BUCKET``. The daemon creates the bucket and key on
first start and refreshes their permissions on later starts. If a key
with the same id already exists with a different secret, Garage
refuses to start; the DebOps secret lookup keeps the values stable.

The role refuses to keep ``--single-node`` enabled when the local
fact ``ansible_local.garage.layout_version`` is greater than 1, so
that a later expansion cannot crash-loop the daemon.


.. _garage__ref_nginx_frontends:

nginx frontends
---------------

Garage has no native TLS. With :envvar:`garage__nginx_enabled`
(the default) the S3 API is published at
:envvar:`garage__nginx_s3_fqdn` plus the wildcard
``*.s3.<domain>`` for virtual-hosted bucket URLs. The proxy is
configured for large objects: ``client_max_body_size 0``,
``proxy_buffering off``, ``proxy_request_buffering off`` and
``proxy_max_temp_file_size 0``. The ``Host`` header is taken from
``$http_host`` so that path-style and virtual-hosted requests both
reach Garage intact.

Website hosting is a separate server, off by default. Set
:envvar:`garage__nginx_web_enabled` to ``True`` to publish
``*.web.<domain>``.
