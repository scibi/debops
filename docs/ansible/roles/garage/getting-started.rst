.. Copyright (C) 2026 Patryk Ściborek <patryk@sciborek.com>
.. Copyright (C) 2026 DebOps <https://debops.org/>
.. SPDX-License-Identifier: GPL-3.0-only

.. _garage__ref_getting_started:

Getting started
===============

.. only:: html

   .. contents::
      :local:


What is Garage?
---------------

`Garage`__ is an S3-compatible object store. It stores data as immutable
blocks, keeps a small metadata index next to them, and can later grow
from one node to a geo-distributed cluster without changing the client
API.

The ``debops.garage`` role installs Garage from the official static musl
binary, writes :file:`/etc/garage.toml`, and manages a single
:file:`garage.service` unit. There is one Garage instance per host.

.. __: https://garagehq.deuxfleurs.fr/


Why a static binary?
--------------------

Garage is published as a static musl binary per architecture, not as a
Debian package. The role therefore downloads
``https://garagehq.deuxfleurs.fr/_releases/v<version>/<target>/garage``
and verifies a SHA256 checksum shipped in
:envvar:`garage__binary_sha256_map`. Upstream does not publish
checksums, so the map is computed when the pinned version is bumped.

The same waterfall used by the vmagent role is available here: an
internal HTTP mirror, a copy from the Ansible Controller, a path already
on the remote host, or :envvar:`garage__skip_install` when the binary is
baked into the image.


A working single-node install
-----------------------------

A freshly started Garage daemon is not usable until a cluster layout
exists. The role does not call :command:`garage layout assign` from
Ansible (that CLI is not idempotent). It starts the daemon with
``--single-node``, which creates a one-node layout at version 1 on first
boot and is a no-op afterwards.

Add the host to ``[debops_service_garage]`` and, if you want a bucket
and a key pair created on the first start, set
:envvar:`garage__default_bucket`:

.. code-block:: yaml

   # inventory/hosts
   [debops_service_garage]
   s3.example.com

   # inventory/host_vars/s3.example.com/garage.yml
   garage__default_bucket: 'media'

Then run:

.. code-block:: console

   debops run service/garage -l s3.example.com

After the play the S3 API is reachable at ``https://s3.example.com``
through :ref:`debops.nginx`, :command:`garage status` shows an assigned
layout (not ``NO ROLE ASSIGNED``), and
``http://127.0.0.1:3903/health`` on the node returns HTTP 200. The default key pair
lives on the Ansible Controller under
:file:`secret/garage/clusters/default/` and is also written to
:file:`/etc/garage/default-credentials.env` on the host.

A convenient client environment:

.. code-block:: bash

   export AWS_ACCESS_KEY_ID="$(sudo grep GARAGE_DEFAULT_ACCESS_KEY /etc/garage/default-credentials.env | cut -d= -f2)"
   export AWS_SECRET_ACCESS_KEY="$(sudo grep GARAGE_DEFAULT_SECRET_KEY /etc/garage/default-credentials.env | cut -d= -f2)"
   export AWS_DEFAULT_REGION=garage
   export AWS_ENDPOINT_URL=https://s3.example.com
   aws s3 cp ./README.md s3://media/README.md
   aws s3 ls s3://media/

:envvar:`garage__replication_factor` defaults to ``1`` because
``--single-node`` requires it. Do not use a replication factor of 1 for
anything other than a single-node or test deployment.


Explicit UNIX account
---------------------

The role uses a dedicated ``_garage`` system account instead of the
upstream ``DynamicUser=true`` unit. DebOps needs a stable UID for data
directories on separate mounts, ACLs and backups.
``DynamicUser`` remaps state under :file:`/var/lib/private/garage` and
breaks when :file:`/var/lib/garage` already exists. The unit still
ships the rest of the upstream hardening (``LimitNOFILE=42000``,
``ProtectHome``, ``NoNewPrivileges``) and sends logs to the journal
through ``GARAGE_LOG_TO_JOURNALD``.


Dependent role usage
--------------------

- :ref:`debops.secret` stores the cluster-wide ``rpc_secret``, admin
  token, metrics token and the optional default key pair under
  :file:`secret/garage/clusters/<name>/`. The path does not include
  ``inventory_hostname``, so every node of the same cluster receives
  the same RPC secret.

- :ref:`debops.nginx` terminates TLS for the S3 API
  (``s3.<domain>`` and ``*.s3.<domain>``) and, when
  :envvar:`garage__nginx_web_enabled` is ``True``, for website hosting
  (``*.web.<domain>``). Garage itself has no native TLS.

- :ref:`debops.ferm` opens the RPC port (3901) to the other members of
  ``[debops_service_garage]``. The S3 and website ports stay closed
  unless nginx is disabled or an explicit allow list is set. The admin
  port (3903) is bound to loopback.

- :ref:`debops.etc_services` registers the four Garage TCP ports.

The admin API exposes ``/health`` and ``/metrics`` on
``127.0.0.1:3903``. Scrape them from the same host with vmagent or
prometheus_exporter.


Example playbook
----------------

If you are using this role without DebOps, here is an example Ansible
playbook that uses the ``debops.garage`` role:

.. literalinclude:: ../../../../ansible/playbooks/service/garage.yml
   :language: yaml
   :lines: 1,6-


Growing from one node to a cluster
----------------------------------

``--single-node`` is safe only while the layout version is 0 or 1 and
the host is the only member of ``[debops_service_garage]``. Adding a
second node without turning the flag off makes the daemon refuse to
start. Before you expand:

1. Set :envvar:`garage__single_node` to ``False`` on the existing
   node and re-run the role so the unit loses the flag.

2. Raise :envvar:`garage__replication_factor` only if you are ready to
   rebuild the layout. The value must be identical on every node; the
   role asserts this.

3. Add the new hosts to ``[debops_service_garage]``, run the role on
   them, then connect them by hand with :command:`garage node connect`
   and :command:`garage layout assign` / :command:`garage layout apply`.

Declarative multi-node layout, node connect, buckets and keys are
intentionally out of scope for this version of the role and will come
in a later change.


Rolling minor upgrades
----------------------

A minor upgrade (same first non-zero version component, for example
``2.4.0`` to ``2.4.1``) is a rolling, node-by-node operation. Run the
role with ``--limit`` or ``serial: 1`` so that only one node is
replaced at a time:

.. code-block:: console

   debops run service/garage --limit s3-1.example.com

See :ref:`garage__ref_guide_upgrades` for major upgrades, metadata
snapshots and the ``garage__upgrade`` gate. The Ansible local fact
``ansible_local.garage.version`` / ``ansible_local.garage.series``
reports what is actually installed.


Ansible tags
------------

You can use Ansible ``--tags`` or ``--skip-tags`` parameters to limit
what tasks are performed during an Ansible run.

Available role tags:

``role::garage``
  Main role tag, used in the playbook to execute all of the role tasks
  as well as role dependencies.

``role::secret``
  Tag for the :ref:`debops.secret` dependency, used to create the
  cluster secret directory on the Ansible Controller.

``skip::garage``
  Skip the main role's tasks.


Other resources
---------------

- `Garage documentation <https://garagehq.deuxfleurs.fr/documentation/>`_
- `Garage configuration reference <https://garagehq.deuxfleurs.fr/documentation/reference-manual/configuration/>`_
- `Garage operations handbook <https://garagehq.deuxfleurs.fr/documentation/operations/>`_
