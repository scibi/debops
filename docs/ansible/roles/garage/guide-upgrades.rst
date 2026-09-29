.. Copyright (C) 2026 Patryk Ściborek <patryk@sciborek.com>
.. Copyright (C) 2026 DebOps <https://debops.org/>
.. SPDX-License-Identifier: GPL-3.0-only

.. _garage__ref_guide_upgrades:

Upgrading Garage
================

.. only:: html

   .. contents::
      :local:


Why the role gates upgrades
---------------------------

Garage treats a change of the first non-zero version component as a
major upgrade. A minor upgrade (``2.4.0`` to ``2.4.1``) is a rolling
restart. A major upgrade (``1.x`` to ``2.0``) switches the RPC protocol,
must go through adjacent series only, and needs every node restarted
together after you have read the upstream changelog.

The role therefore refuses to replace an installed binary unless
:envvar:`garage__upgrade` allows that class of change. Bumping
:envvar:`garage__version` without also updating
:envvar:`garage__binary_sha256_map` will fail the checksum assertion.


Minor upgrades
--------------

A minor upgrade stays inside the same series. For versions ``X.y.z``
with ``X >= 1`` the series is ``X``; for ``0.x.y`` the series is
``0.x``.

Default :envvar:`garage__upgrade` is ``minor``, so a play that only
bumps the patch or minor number inside the series will:

1. Optionally run :command:`garage repair --all-nodes --yes tables`
   when :envvar:`garage__upgrade_repair_tables` is ``True`` (once per
   play, cluster-wide).

2. Run :command:`garage meta snapshot` on the still-running daemon
   when :envvar:`garage__upgrade_snapshot` is ``True``.

3. Stop :file:`garage.service`, replace the binary, start the service
   again.

Run the playbook node by node:

.. code-block:: console

   debops run service/garage --limit 192.0.2.10
   debops run service/garage --limit 192.0.2.11

or set ``serial: 1`` in a wrapper playbook. Clients keep working as
long as the remaining nodes still form a quorum.


Major upgrades
--------------

Set :envvar:`garage__upgrade` to ``major`` only after you have:

- read the upstream changelog and any ``garage migrate`` notes for
  that hop,
- confirmed the installed series and the target series are adjacent
  (``1`` to ``2`` is fine, ``1`` to ``3`` is refused),
- taken a metadata snapshot you can restore.

A major upgrade changes the RPC protocol. Stop writing, upgrade every
node to the new binary, then start them together. The role still
replaces one host per play; coordinate the restart yourself.

.. code-block:: yaml

   garage__version: '2.4.1'
   garage__upgrade: 'major'
   garage__upgrade_snapshot: True


Disabling upgrades
------------------

:envvar:`garage__upgrade` ``none`` makes the role fail when the
installed binary does not match :envvar:`garage__version`. Use this on
hosts where a binary change must be a separate, reviewed event.


Downgrades
----------

Downgrades are not supported. Restore the previous data directory from
backup and pin :envvar:`garage__version` to the restored binary.


Local facts
-----------

After a successful play, ``ansible_local.garage`` reports at least:

``version``
  Installed binary version.

``series``
  First non-zero version component, used by the upgrade gate.

``node_id``
  Node identifier, when the CLI can read it.

``layout_version``
  Current cluster layout version, when the daemon is reachable. The
  role uses this to refuse ``--single-node`` once the layout has grown
  past version 1.
