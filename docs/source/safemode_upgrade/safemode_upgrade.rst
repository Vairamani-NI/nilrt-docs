==================================================
Upgrading Safemode from a Package Feed
==================================================

.. contents:: Table of Contents
   :depth: 2
   :local:

Introduction
============

Safemode is the recovery operating system of an NI Linux Real-Time target. It
lives on the boot partition, separate from the runmode system image, and is
usually updated with NI MAX or the Hardware Configuration utility.

The ``ni-safemode-image`` package lets you update safemode with opkg instead.
The package carries a complete safemode image. Installing it applies that image
to the boot partition, so a target can update its safemode from a package feed
over ssh or a serial console, from runmode or from safemode itself, without MAX
or the Hardware Configuration utility.

This is useful when you:

- update safemode on many targets with scripts, for example on a manufacturing line.
- maintain targets that have no Windows host with MAX available.
- need to bring targets up to the minimum safemode version a system image requires.

Supported Targets
=================

``ni-safemode-image`` is built for:

- **x64 targets.** The package carries the safemode kernel, ramdisk and grub
  configuration.
- **ARM (Zynq) targets**, such as Zynq-based CompactRIO, Single-Board RIO and
  FlexRIO controllers. The package carries the safemode FIT image
  (``linux_safemode.itb``). One image serves every Zynq device code, so there is
  one package for all ARM targets.

Before the image is applied, the installer checks that it was built for the
target's device code.

How It Works
============

Installing or upgrading the package runs ``niinstallsafemode`` from the
package's post-install script. The script follows these rules.

Version policy
--------------

opkg decides which version of the package is installed. The package version is
derived from the safemode version it carries, so opkg orders packages the same
way NI's safemode version check does. For example, safemode ``27.0.0d23`` is
packaged as version ``27.0.0.0.23``.

When the package is installed, its safemode image is compared with the safemode
already on the target:

- **Newer or older:** the image is applied. Downgrades are applied as well;
  opkg itself only installs an older package when you ask it to, for example with
  ``--force-downgrade``.
- **Same version:** nothing is written, and the install succeeds.
- **Unreadable version in the package:** the install is refused.

The new safemode is used the next time the target boots into safemode.

Interrupted upgrades
--------------------

The package keeps a small amount of state in ``/boot/.ni-safemode-image/`` on
the boot partition, which is persistent in both runmode and safemode:

- A **copy journal** is written before the image is copied and removed once the
  copy completes. If the copy is interrupted, for example by a power loss, run
  ``opkg configure ni-safemode-image``, or install the same package version
  again after a reboot. Either one completes the copy. A different package
  version is refused until the recorded one completes.
- On x64 targets with encrypted volumes, a **reseal marker** lists the volumes
  that still need to be resealed. See `Encrypted x64 targets`_.

While the package is being installed, and while a reseal is pending, a software
reboot or shutdown waits until the work completes. An unfinished copy does not
hold off a reboot, because the copy cannot run during shutdown; complete it
after the reboot as described above.

``opkg remove ni-safemode-image`` is refused while a copy or a reseal is
unfinished. Upgrading or reinstalling the package is still allowed.

Encrypted x64 targets
---------------------

On x64 targets whose ``niconfig`` or user volumes are encrypted, the volumes are
unlocked by TPM policies sealed to the measured safemode boot files. Before a
new safemode image is applied to such a target:

- ``ni-reseal-luks`` must be installed, and a dry run must show that each
  encrypted volume can be resealed. Otherwise the install is refused.
- After the copy, each volume is resealed. If resealing fails, reboot is held
  off and ``opkg configure ni-safemode-image`` retries the volumes that are left.

ARM targets
-----------

On ARM targets, ``niinstallsafemode`` installs the safemode FIT image together
with the default FPGA bitfile (``/boot/.defbit/default.bit.*``). The package
does not carry bitfiles: it installs the new safemode image together with the
target's current default bitfile, so the default bitfile is not changed.

ARM boot partitions are small. Before anything is written, the package checks
that ``/boot`` has room for the new image and refuses the install if it does
not.

Requirements
============

- A package feed that contains ``ni-safemode-image`` for your target, either the
  NI feed for your NI Linux Real-Time release or a local feed.
- About twice the size of the safemode image in free memory or storage while
  the package is installed. The package is downloaded and unpacked by opkg, and
  ``niinstallsafemode`` extracts it again. In safemode, the root file system is
  held in RAM.
- On ARM targets, free space on ``/boot`` for the difference in size between
  the new and the installed safemode image.

Upgrading Safemode
==================

The same commands work in runmode and in safemode.

#. Check the installed safemode version.

   .. code:: bash

       nisafemodeversion

#. Make sure opkg has a feed that contains ``ni-safemode-image``. In runmode,
   the configured NI feeds are usually enough. In safemode, the configured feeds
   point at the feeds of the release the safemode came from; add the feed you
   want to upgrade from if needed, for example:

   .. code:: bash

       echo "src/gz safemode-upgrade http://<feed server>/<path>" > /etc/opkg/safemode-upgrade.conf

   .. note::
       Package feeds are signed. If your feed is not signed, opkg rejects it.

#. Update the package lists and check the available version.

   .. code:: bash

       opkg update
       opkg info ni-safemode-image

#. Install the package. To move to a newer safemode later, use
   ``opkg upgrade ni-safemode-image``.

   .. code:: bash

       opkg install ni-safemode-image

#. Check the result. ``nisafemodeversion`` prints the version of the safemode
   now on the boot partition.

   .. code:: bash

       nisafemodeversion

#. Reboot into safemode when you want to use the new safemode.

To go back to an older safemode, make the older package version available in a
feed (or as a package file), then install it with
``opkg install --force-downgrade ni-safemode-image``. If the package is not
installed, ``opkg install ni-safemode-image`` from a feed that has only the
older version is enough.

.. note::
    In safemode, the opkg package database is held in RAM. After a reboot,
    ``ni-safemode-image`` is no longer listed as installed in safemode; the
    safemode image on the boot partition stays.

Troubleshooting
===============

The post-install script prints the reason when it does not change safemode.

``safemode is already at the packaged version``
    The target already has this safemode version. Nothing was written.

``cannot read the safemode version from ...``
    The package's safemode image has no readable version. Use a different
    package.

``an unfinished upgrade to safemode <version> is pending``
    A previous upgrade did not complete. Install that package version again,
    or run ``opkg configure ni-safemode-image`` if it is still installed.

``niinstallsafemode failed with exit code <n>``
    The image could not be applied. If the message also says
    ``run 'opkg configure ni-safemode-image' to complete the copy``, part of
    the image was written; complete the copy before rebooting into safemode.

``encrypted volumes found but ni-reseal-luks is not installed`` or ``<volume> cannot be resealed``
    x64 only. Install ``ni-device-encryption`` or make sure the volume can be
    unlocked, then install the package again.

``resealing <volume> failed; reboot is held off until 'opkg configure ni-safemode-image' reseals it``
    x64 only. Fix the cause and run ``opkg configure ni-safemode-image``.

``no default bitfile found in /boot/.defbit``
    ARM only. The target has no default FPGA bitfile to install with the
    safemode image. Install safemode with NI MAX or the Hardware Configuration
    utility instead.

``not enough space on /boot for the new safemode image``
    ARM only. Free space on ``/boot`` and install the package again.

``a safemode upgrade is unfinished; run 'opkg configure ni-safemode-image' to complete it before removing ni-safemode-image``
    Shown by ``opkg remove``. Complete the upgrade first.
