
                                          |\_
                 B A C K U P N I N J A   /()/
                                         `\|

      a silent flower blossom death strike to lost data.

Backupninja allows you to coordinate system backup by dropping a few
simple configuration files into `/etc/backup.d/`. Most programs you
might use for making backups don't have their own configuration file
format. Backupninja provides a centralized way to configure and
coordinate many different backup utilities.

Features
========

The key features of backupninja are:

 - easy to read ini style configuration files
 - you can drop in scripts to handle new types of backups
 - backup actions can be scheduled
 - you can choose when status report emails are mailed to you
   (always, on warning, on error, never)
 - console-based wizard (ninjahelper) makes it easy to create
   backup action configuration files
 - passwords are never sent via the command line to helper programs

The following backup types are supported:

 - secure, remote, incremental filesystem backup (via rdiff-backup)
   incremental data is compressed. permissions are retained even
   with an unpriviledged backup user
 - backup of mysql databases (via mysqlhotcopy and mysqldump)
 - basic system and hardware info
 - encrypted remote backups (via duplicity or borgbackup)
 - backup of subversion repositories

Installation
============

See the [installation documentation](INSTALL.md).

Options
=======

The following options are available:

 - `-h`, `--help`: this usage message
 - `-d`, `--debug`: run in debug mode, where all log messages are
   output to the current shell
 - `-f`, `--conffile FILE`: use FILE for the main configuration
   instead of `/etc/backupninja.conf`
 - `-t`, `--test`: test run mode. This will test if the backup could
   run, without actually preforming any backups. For example, it will
   attempt to authenticate or test that ssh keys are set correctly.
 - `-n`, `--now`: perform actions now, instead of when they might
   be scheduled. No output will be created unless also run with -d.
 - `--run FILE`: runs the specified action `FILE` (e.g. one of the
   `/etc/backup.d/` files). Also puts backupninja in debug mode.

ninjahelper
===========

ninjahelper is an additional script which will walk you through the process of
configuring backupninja. Ninjahelper has a menu driven curses based interface
(using dialog).

To add an additional 'wizard' to ninjahelper, follow these steps:

1. To add a helper for the handler "blue", create the file
   `blue.helper` in the directory where the handlers live.
   (ie `/usr/share/backupninja`).

2. Next, you need to add your helper to the global `HELPERS` variable
   and define the main function for your helper (the function name
   is always `<helper>_wizard`). for example, `blue.helper`:

           HELPERS="$HELPERS blue:description_of_this_helper"
           blue_wizard() {
             ... do work here ...
           }

3. Look at the existing helpers to see how they are written. Try to re-use
   functions, such as the dialog functions that are defined in `easydialog.sh`.

4. Test, re-test, and test again. Try to break the helper by going backwards,
   try to think like someone who has no idea how to configure your handler
   would think, try to make your helper as simple as possible. Walk like a cat,
   become your shadow, don't let your senses betray you.

Configuration
=============

Configuration files
-------------------

The general configuration file is `/etc/backupninja.conf`. In this file
you can set the log level and change the default directory locations.
You can force a different general configuration file with `backupninja
-f /path/to/conf`.

To preform the actual backup, backupninja processes each configuration
file in `/etc/backup.d` according to the file's suffix:

 - `.sh`: run this file as a shell script.
 - `.rdiff`: filesystem backup (using rdiff-backup)
 - `.restic`: filesystem backup (using restic)
 - `.dup`: filesystem backup (using duplicity)
 - `.borg`: filesystem backup (using borg)
 - `.mysql`: backup mysql databases
 - `.pgsql`: backup PostgreSQL databases
 - `.sys`: general hardware, partition, and system reports.
 - `.svn`: backup subversion repositories
 - `.maildir`: incrementally backup maildirs (very specialized)

Support for additional configuration types can be added by dropping
bash scripts with the name of the suffix into `/usr/share/backupninja`.

The configuration files are processed in alphabetical order. However,
it is suggested that you name the config files in "sysvinit style."

For example:

        00-disabled.pgsql
        10-runthisfirst.sh
        20-runthisnext.mysql
        90-runthislast.rdiff

Typically, you will put a `.rdiff` config file last, so that any
database dumps you make are included in the filesystem backup.
Configurations files with names beginning with 0 (zero) or ending with
`.disabled` (preferred method) are skipped.

Unless otherwise specified, the config file format is "ini style."

For example:

        # this is a comment

        [fishes]
        fish = red
        fish = blue

        [fruit]
        apple = yes
        pear = no thanks \
        i will not have a pear.

The [example configuration files](examples) document all options
supported by the handlers shipped with backupninja.

Scheduling
----------

By default, each configuration file is processed everyday at 01:00 (1
AM). This can be changed by specifying the 'when' option in a config
file.

For example:

        when = sundays at 02:00
        when = 30th at 22
        when = 30 at 22:00
        when = everyday at 01            <-- the default
        when = Tuesday at 05:00

A configuration file will be processed at the time(s) specified by the
`when` option. If multiple `when` options are present, then they all
apply. If two configurations files are scheduled to run in the same
hour, then we fall back on the alphabetical ordering specified above.
If two configurations files are scheduled close to one another in
time, it is possible to have multiple copies of backupninja running if
the first instance is not finished before the next one starts.

Make sure that you put the `when` option before any sections in your
configuration file.

These values for `when` are equivalent:

        when = tuesday at 05:30
        when = TUESDAYS at 05

These values for `when` are invalid:

        when = tuesday at 2am
        when = tuesday at 2
        when = tues at 02

SSH keys
--------

In order for rdiff-backup to sync files over ssh unattended, you must
create ssh keys on the source server and copy the public key to the
remote user's authorized keys file. For example:

        root@srchost# ssh-keygen -t rsa -b 4096
        root@srchost# ssh-copy-id -i /root/.ssh/id_rsa.pub backup@desthost

Now, you should be able to ssh from user `root` on `srchost` to
user `backup` on `desthost` without specifying a password.

Note: when prompted for a password by `ssh-keygen`, just leave it
blank by hitting return.

The included helper program `ninjahelper` will walk you through creating
an rdiff-backup configuration, and will set up the ssh keys for you.


Amazon Simple Storage Service (S3)
----------------------------------

Duplicity can store backups on Amazon S3 buckets, taking care of encryption.
Since it performs incremental backups it minimizes the number of request per
operation therefore reducing the costs. The boto Python interface to Amazon
Web Services is needed to use duplicity with S3 (Debian package: `python-boto`).

.sh configuration files
-----------------------

Shell jobs may use the following features:

  * logging and control flow functions:
    `halt`, `fatal`, `error`, `warning`, `info`, `debug`, `passthru`.
    All such functions take a list of strings a parameters.
    Those strings are passed to whatever logging mechanism is enabled,
    and colored if relevant.

  * Using `exit N` is useless, and has unspecified consequences.
    Just don't do it.

  * `when=TIME` works as documented above; at may also be written
    `when = TIME`.

  * The `$BACKUPNINJA_DEBUG` environment variable is set when
    backupninja is invoked with the `-d` option.

Real world usage
================

Backupninja can be used to implement whatever backup strategy you
choose. It is intended, however, to be used like so:

1. First, databases are safely copied or exported to `/var/backups`.
   Typically, you cannot make a file backup of a database while it
   is in use, hence the need to use special tools to make a safe copy
   or export into `/var/backups`.

2. Then, vital parts of the file system, including `/var/backups`, are
   nightly pushed to a remote, off-site, hard disk (using
   rdiff-backup). The local user is root, but the remote user is not
   priviledged. Hopefully, the remote filesystem is encrypted.

There are many different backup strategies out there, including "pull
style", magnetic tape, rsync + hard links, etc. We believe that the
strategy outlined above is the way to go because:

1. hard disks are very cheap these days;
2. pull style backups are no good, because then the backup server must
   have root on the production server;
3. rdiff-backup is more space efficient and featureful than using
   rsync + hard links.

More information
================

FAQ
---

See the [FAQ](FAQ.md).

Mailing list
------------

The backupninja
[mailing list](https://lists.riseup.net/www/info/backupninja) is
suitable for usage and development questions.
