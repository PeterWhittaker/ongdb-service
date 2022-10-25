## tl;dr

**NOTE**: Do NOT run these commands unless you understand what they do;
          please read the rest of the document first.

Assuming the contents of this repo have been copied to the current folder of
the target machine,

```
sudo setenforce 0
sudo dnf -y install selinux-policy-devel
cd selinux
make
cd ..
sudo semodule -i ongdb.pp
sudo setsebool -P domain_can_mmap_files on
sudo /usr/bin/mkdir -p /tmp/ongdbJava
sudo /usr/sbin/restorecon -v /tmp/ongdbJava
sudo cp ongdb.service /lib/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable ongdb
sudo sed -i /etc/selinux/config -e 's/^ *SELINUX=permissive/#SELINUX=permissive/' -e 's/^ *# *SELINUX=enforcing/SELINUX=enforcing/'
sudo touch /.autorelabel
sudo reboot
```

(The `-v` option to `restorecon` is unnecessary, it's just nice when restorecon
tells you if it did anything.)

## Background and Contents

This repo contains a systemd service definition file and associated SELinux
policy to allow ONgDB to launch automatically at system startup and run in
SELinux enforcing mode, which provides three benefits:

    1. Automatic startup
    2. Reduce the risk of compromise to ONgDB data, executables, and processes
    3. Limit harm to the rest of the system caused by a successful compromise
       of ONgDB

While the systemd service file and SELinux policy are fairly straightforward,
relatively typical examples of their kind, there is one particularity that may
stand out: The use of the `/tmp/ongdbJava` directory.

ONgDB is itself a Java application, and one that creates and uses temporary
dynamic libraries. By default, these are created directly in `/tmp`, and ONgDB
must be able to execute code from these libraries. When the SELinux policy was
created, there were essentially two alternatives to manage this:

    1. Use `/tmp` for these libraries and allow either ONgDB or all processes
       to create and execute executable files in `/tmp`, or
    2. Use a dedicated temporary directory and restrict it to ONgDB

There were numerous problems with the first alternative, including: increased
risk of system compromise (if permissions on `/tmp` were too liberal, e.g.);
limiting functionality of other applications and systems (if permissions on
`/tmp` were made too narrow, e.g.); etc.

The second alternative was therefore adopted and takes advantage of an
environment variable interpreted by Java: `java.io.tmpdir`. This environment
variable indicates where Java should create temporary files and directories.
The service file and SELinux policy defined in this repo have "baked in"
knowledge of a directory that should exist for this purpose: `/tmp/ongdbJava`

The systemd service file ensures that this variable is set appropriately for
ONgDB via the line:
```
Environment="_JAVA_OPTIONS=-Djava.io.tmpdir=/tmp/ongdbJava"
```

Likewise,

    1. an SELinux policy rule in `ongdb.fc` ensures that this directory will
       have the appropriate SELinux context, and
    2. an SELinux policy rule in `ongdb.te` ensures a) that files created in
       this folder will have the appropriate context, and b) that only ONgDB
       will be able to access this directory and these files.

While arguably imperfect, the intent is to limit who can access ONgDB data and
files and what harm can come from any compromise of these data and files. This
approach provides a reasonably well balanced approach.

## Installation

There is no automatic deployment/installation/configuration script. However,
there is a basic recipe - each of these steps is explained in more detail,
below:

    1. Update the SELinux policy on the target machine
    2. Update systemd configuration
    3. Ensure that the required ONgDB /tmp folder exists

### Update the SELinux policy on the target machine

Summary:
```
sudo semodule -i ongdb.pp
sudo setsebool -P domain_can_mmap_files on
```

**NOTE**: Depending on the existing SELinux configuration, it may be necessary
to temporarily disable SELinux:
```
sudo setenforce 0
sudo semodule -i ongdb.pp
sudo setsebool -P domain_can_mmap_files on
sudo setenforce 1
```

If the policy (ongdb.pp) has been built elsewhere, copy it to the target
machine and run the above commands.

To build the policy on the machine,

    1. Copy the contents of the selinux folder to the target machine
    1. Ensure the `selinux-policy-devel` package is installed, e.g.,
       `sudo dnf -y install selinux-policy-devel`
    1. `cd selinux`
    1. `make`
    1. run the commands above

### Update systemd configuration
Copy the ongdb.service file to the target machine, then:
```
sudo cp ongdb.service /lib/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable ongdb
```

### Ensure that the required ONgDB /tmp folder exists

```
sudo /usr/bin/mkdir -p /tmp/ongdbJava
sudo /usr/sbin/restorecon -v /tmp/ongdbJava
```

**NOTE**: Depending on the existing SELinux configuration, it may be necessary
to temporarily disable SELinux:
```
sudo setenforce 0
sudo /usr/bin/mkdir -p /tmp/ongdbJava
sudo /usr/sbin/restorecon -v /tmp/ongdbJava
sudo setenforce 1
```

### Miscellaneous notes and commands

#### Switching SELinux to enforcing mode permanently

Use `sudo getenforce` to determine if SELinux is operating in permissive or
enforcing mode. If operating in permissive mode and enforcing mode is desired,
the following commands will ensure that the machine reboots in enforcing mode:

```
sudo setenforce 0
sudo sed -i /etc/selinux/config -e 's/^ *SELINUX=permissive/#SELINUX=permissive/' -e 's/^ *# *SELINUX=enforcing/SELINUX=enforcing/'
sudo reboot
```

#### Making absolutely sure that SELinux contexts are correct

When initially configuring a machine that will operate in enforcing mode, it is
possible to cause files and directories to have incorrect file contexts, which
will cause failures later on; a common cause of this is using `mv` instead of
`cp` or other command to put a file into a privileged folder (tl;dr: `cp`
respects the target context, `mv` preserves the source context, and the target
context is likely the right one).

While `/usr/sbin/restorecon` can be used to correct contexts, it is in many
ways far simpler to do this:

```
sudo touch /.autorelabel
sudo reboot
```

The `touch` command creates a file that causes all contexts on all directories
and files to be set properly, according to current policy, during the next
reboot, before any services start.

**NOTE**: Do this **before** running the `sed` command to make enforcing mode
permanent. Resetting contexts can have undesirable effects if policy isn't yet
correct, e.g., locking you out of your system. *hic sunt draconis*

#### Debugging SELinux problems

```
sudo ausearch -m avc -m user_avc | tee denialContext | audit2allow > denialRules
```

This command creates two files that can help you debug SELinux access issues:

    1. `denialRules`, automatically generated SELinux policy statements that
       *can* be used to resolve any access issues with the current policy.
       **DO NOT** simply implement these rules without understanding why they
       arise - they may enable things best left disabled.
    2. `denialContext`, the unfiltered results of ausearch. Very useful to
       understand what was denied and why - used to interpret and assess the
       the rule suggestions in `denialRules`.

