# SELinux rules for Verity - possibly incomplete

These SELinux rules were finalized 2022-09-23 by @pwhittaker based on work done
on ongdb rules by @seanle. They were developed and verified on the AWS test
instance (10.0.111.181) by starting and stopping and starting the Verity
service in enforcing mode and running ausearch (cf the `a2a` script) until
everything worked and only unacceptable audit2allow recommendations remained,
then repeating the cycle after rebooting into enforcing mode - as of this
writing, the machine is operating in enforcing mode and all seems to work.

## NOTE NOTE NOTE

### VERY VERY IMPORTANT - Executable dynamic libraries in /tmp

For reasons we cannot begin to fathom, ONgDB (or perhaps the underlying Java
libraries) creates executable dynamic libraries in /tmp and then wishes to
execute code from these libraries.

We cannot begin to describe what an astonishingly large potential security hole
this is. Anything that could compromise ONgDB could give itself the ability to
generate and execute any code whatsoever.

The current rules go *just far enough* to allow ONgDB to start: It can create
the required directories and folders in /tmp, but it CANNOT execute anything:
Running ONgDB with these rules will generate a denial AVC for execute of these
SO files. LEAVE IT THIS WAY. Everything seems to work, and without execute, the
security risk associated with these files is far lower.

Correcting this is tricky, because ONgDB creates other files in /tmp as well,
files that should NOT be executable, so there is no perfectly straightforward
way to limit access appropriately. It may be possible to create a special type
just for ONgDB tmp files, add a file transition to set these files to that
type, set the default context, and let 'er rip, but this seems half-baked and
fraught.

The correct approach to this problem would be for ONgDB to create these files
in a dedicated directory, e.g., /tmp/ongdb_lib, have transition rules and
contexts associated with this directory and its context and content, and limit
what these executables can access, e.g., ongdb_t or ongdb_db_t, but that still
feels flaky.

For now, leave things as they are, unless advised otherwise by a truly credible
SME.

### Moderately important

**hsperfdata... services**
Baked into ONgDB through its use of Java is a Java performance assessment tool;
without specific rules, especially file contexts, to access directories and
files associated with this tool, either ONgDB would fail or it would have too
great an access to other services. The odd-looking context rules for hsperfdata
are there to allow ONgDB to start, but do not give it access to anything else
and do not give anythign else access to those files.

**SSS Services**
ONgDB requires access to files and services associated with the RHEL SSS
authentication service. As far as we can tell, as long as it query these files
and services, it runs fine, even if the service is disabled - it can do without
it, there are just some naive assumptions about its presence and operational
status. Minimal rules have been added to allow ONgDB to start, even when SSS is
unavailable.

## The files
README.md       This file.
Makefile        File to make, install, remove the policy module (`make && make install`)
a2a             `./a2a <timestamp> [less]` List all violations since timestamp, generating rules as required
verity.te       The core policy file
verity.fc       File contexts for that policy
verity.if       Not used right now - potential interface file if other services need access to Verity instances

## Installing, updating images

### Synopsis
make
copy verity.pp to the installation image
ensure the installation image executes these two commands:
    semodule -i ongdb.pp
    setsebool -P domain_can_mmap_files true

