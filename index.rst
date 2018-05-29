Quick Start
===========

Administration QuickStart
-------------------------

This document will cover installation and administration points of
Singularity for multi-tenant HPC resources and will not cover usage of
the command line tools, container usage, or example use cases.

Installation
~~~~~~~~~~~~

There are two common ways to install Singularity, from source code and
via binary packages. This document will explain the process of
installation from source, and it will depend on your build host to have
the appropriate development tools and packages installed. For Red Hat
and derivatives, you should install the following group to ensure you
have an appropriately setup build server:

::

    $ sudo yum groupinstall "Development Tools"

You can download the source code either from the latest stable tarball
release or via the GitHub master repository. Here is an example
downloading and preparing the latest development code from GitHub:

::

    $ mkdir ~/git
    $ cd ~/git
    $ git clone https://github.com/singularityware/singularity.git
    $ cd singularity
    $ ./autogen.sh

| Once you have downloaded the source, the following installation
  procedures will assume you are running from the root of the source
  directory.

| The following example demonstrates how to install Singularity into .
  You can install Singularity into any directory of your choosing, but
  you must ensure that the location you select supports programs running
  as . It is common for people to disable with the mount option for
  various network mounted file systems. To ensure proper support, it is
  easiest to make sure you install Singularity to a local file system.
| Assuming that is a local file system:

| **NOTE: The above must be run as root to have Singularity properly
  installed. Failure to install as root will cause Singularity to not
  function properly or have limited functionality when run by a non-root
  user.**
| Also note that when you configure, is **not** required, however it is
  required for full functionality. You will see this message after the
  configuration:

::

    mksquashfs from squash-tools is required for full functionality

If you choose not to install , you will hit an error when your users try
a pull from Docker Hub, for example.

| As with most autotools-based build scripts, you are able to supply the
  argument to the configure script to change where Singularity will be
  installed. Care must be taken when this path is not a local filesystem
  or has atypical permissions. The local state directories used by
  Singularity at runtime will also be placed under the supplied and this
  will cause malfunction if the tree is read-only. You may also
  experience issues if this directory is shared between several
  hosts/nodes that might run Singularity simultaneously.
| In such cases, you should specify the variable in addition to . This
  will override the prefix, instead placing the local state directories
  within the path explicitly provided. Ideally this should be within the
  local filesystem, specific to only a single host or node.
| For example, the Makefile contains this variable by default:

::

    CONTAINER_OVERLAY = ${prefix}/var/singularity/mnt/overlay

By supplying the configure argument Singularity will instead be built
with the following. Note that that has been replaced by the supplied
value:

::

    CONTAINER_OVERLAY = /some/other/place/singularity/mnt/overlay

In the case of cluster nodes, you will need to ensure the following
directories are created on all nodes, with ownership and permissions:

::

    ${localstatedir}/singularity/mnt
    ${localstatedir}/singularity/mnt/container
    ${localstatedir}/singularity/mnt/final
    ${localstatedir}/singularity/mnt/overlay
    ${localstatedir}/singularity/mnt/session

Singularity will fail to execute without these directories. They are
normally created by the install make target; when using a local
directory for these will only be created on the node is run on.

Singularity includes all of the necessary bits to properly create an RPM
package directly from the source tree, and you can create an RPM by
doing the following:

::

    $ ./configure
    $ make dist
    $ rpmbuild -ta singularity-*.tar.gz

Near the bottom of the build output you will see several lines like:

::

    ...
    Wrote: /home/gmk/rpmbuild/SRPMS/singularity-2.3.el7.centos.src.rpm
    Wrote: /home/gmk/rpmbuild/RPMS/x86_64/singularity-2.3.el7.centos.x86_64.rpm
    Wrote: /home/gmk/rpmbuild/RPMS/x86_64/singularity-devel-2.3.el7.centos.x86_64.rpm
    Wrote: /home/gmk/rpmbuild/RPMS/x86_64/singularity-debuginfo-2.3.el7.centos.x86_64.rpm
    ...

You will want to identify the appropriate path to the binary RPM that
you wish to install, in the above example the package we want to install
is , and you should install it with the following command:

::

    $ sudo yum install /home/gmk/rpmbuild/RPMS/x86_64/singularity-2.3.el7.centos.x86_64.rpm

Note: If you want to have the binary RPM install the files to an
alternative location, you should define the environment variable
‘PREFIX’ (below) to suit your needs, and use the following command to
build:

::

    $ PREFIX=/opt/singularity
    $ rpmbuild -ta --define="_prefix $PREFIX" --define "_sysconfdir $PREFIX/etc" --define "_defaultdocdir $PREFIX/share" singularity-*.tar.gz

We recommend you look at our to get further information about container
privileges and mounting.

Security
--------

Container security paradigms
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| First some background. Most container platforms operate on the
  premise, **trusted users running trusted containers**. This means that
  the primary UNIX account controlling the container platform is either
  “root” or user(s) that root has deputized (either via or given access
  to a control socket of a root owned daemon process).
| Singularity on the other hand, operates on a different premise because
  it was developed for HPC type infrastructures where you have users,
  none of which are considered trusted. This means the paradigm is
  considerably different as we must support **untrusted users running
  untrusted containers**.

Untrusted users running untrusted containers!
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| This simple phrase describes the security perspective Singularity is
  designed with. And if you additionally consider the fact that running
  containers at all typically requires some level of privilege
  escalation, means that attention to security is of the utmost
  importance.

As mentioned, there are several containerization system calls and
functions which are considered “privileged” in that they must be
executed with a certain level of capability/privilege. To do this, all
container systems must employ one of the following mechanisms:

#. **Limit usage to root:** Only allow the root user (or users granted )
   to run containers. This has the obvious limitation of not allowing
   arbitrary users the ability to run containers, nor does it allow
   users to run containers as themselves. Access to data, security data,
   and securing systems becomes difficult and perhaps impossible.

#. **Root owned daemon process:** Some container systems use a root
   owned daemon background process which manages the containers and
   spawns the jobs within the container. Implementations of this
   typically have an IPC control socket for communicating with this root
   owned daemon process and if you wish to allow trusted users to
   control the daemon, you must give them access to the control socket.
   This is the Docker model.

#. **SetUID:** Set UID is the “old school” UNIX method for running a
   particular program with escalated permission. While it is widely used
   due to it’s legacy and POSIX requirement, it lacks the ability to
   manage fine grained control of what a process can and can not do; a
   SetUID root program runs as root with all capabilities that comes
   with root. For this reason, SetUID programs are traditional targets
   for hackers.

#. **User Namespace:** The Linux kernel’s user namespace may allow a
   user to virtually become another user and run a limited set
   privileged system functions. Here the privilege escalation is managed
   via the Linux kernel which takes the onus off of the program. This is
   a new kernel feature and thus requires new kernels and not all
   distributions have equally adopted this technology.

#. **Capability Sets:** Linux handles permissions, access, and roles via
   capability sets. The root user has these capabilities automatically
   activated while non-privileged users typically do not have these
   capabilities enabled. You can enable and disable capabilities on a
   per process and per file basis (if allowed to do so).

Singularity must allow users to run containers as themselves which rules
out options 1 and 2 from the above list. Singularity supports the rest
of the options to following degrees of functionally:

-  **User Namespace:** Singularity supports the user namespace natively
   and can run completely unprivileged (“rootless”) since version 2.2
   (October 2016) but features are severely limited. You will not be
   able to use container “images” and will be forced to only work with
   directory (sandbox) based containers. Additionally, as mentioned, the
   user namespace is not equally supported on all distribution kernels
   so don’t count on legacy system support and usability may vary.

-  **SetUID:** This is the default usage model for Singularity because
   it gives the most flexibility in terms of supported features and
   legacy compliance. It is also the most risky from a security
   perspective. For that reason, Singularity has been developed with
   transparency in mind. The code is written with attention to
   simplicity and readability and Singularity increases the effective
   permission set only when it is necessary, and drops it immediately
   (as can be seen with the –debug run flag). There have been several
   independent audits of the source code, and while they are not
   definitive, it is a good assurance.

-  **Capability Sets:** This is where Singularity is headed as an
   alternative to SetUID because it allows for much finer grained
   capability control and will support all of Singularity’s features.
   The downside is that it is not supported equally on shared file
   systems.

Where are the Singularity priviledged components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When you install Singularity as root, it will automatically setup the
necessary files as SetUID (as of version 2.4, this is the default run
mode). The location of these files is dependent on how Singularity was
installed and the options passed to the script. Assuming a default run
which installs files into of you can find the SetUID programs as
follows:

::

    $ find /usr/local/libexec/singularity/ -perm -4000
    /usr/local/libexec/singularity/bin/start-suid
    /usr/local/libexec/singularity/bin/action-suid
    /usr/local/libexec/singularity/bin/mount-suid

| Each of the binaries is named accordingly to the action that it is
  suited for, and generally, each handles the required privilege
  escalation necessary for Singularity to operate. What specifically
  requires escalated privileges?

#. Mounting (and looping) the Singularity container image

#. Creation of the necessary namespaces in the kernel

#. Binding host paths into the container

Removing any of these SUID binaries or changing the permissions on them
would cause Singularity to utilize the non-SUID workflows. Each file
with also has a non-suid equivalent:

::

    /usr/local/libexec/singularity/bin/start
    /usr/local/libexec/singularity/bin/action
    /usr/local/libexec/singularity/bin/mount

| While most of these workflows will not properly function without the
  SUID components, we have provided these fall back executables for
  sites that wish to limit the SETUID capabilities to the bare
  essentials/minimum. To disable the SetUID portions of Singularity, you
  can either remove the above files, or you can edit the setting for at
  the top of the file, which is typically located in .

::

    # ALLOW SETUID: [BOOL]
    # DEFAULT: yes
    # Should we allow users to utilize the setuid program flow within Singularity?
    # note1: This is the default mode, and to utilize all features, this option
    # will need to be enabled.
    # note2: If this option is disabled, it will rely on the user namespace
    # exclusively which has not been integrated equally between the different
    # Linux distributions.
    allow setuid = yes

You can also install Singularity as root without any of the SetUID
components with the configure option as follows:

::

    $ ./configure --disable-suid --prefix=/usr/local
    $ make
    $ sudo make install

Can I install Singularity as a user?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Yes, but don’t expect all of the functions to work. If the SetUID
components are not present, Singularity will attempt to use the “user
namespace”. Even if the kernel you are using supports this namespace
fully, you will still not be able to access all of the Singularity
features.

Container permissions and usage strategy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| As a system admin, you want to set up a configuration that is
  customized for your cluster or shared resource. In the following
  paragraphs, we will elaborate on this container permissions strategy,
  giving detail about which users are allowed to run containers, along
  with image curation and ownership.
| These settings can all be found in the Singularity configuration file
  which is installed to . When running in a privileged mode, the
  configuration file **MUST** be owned by root and thus the system
  administrator always has the final control.

| Singularity supports several different container formats:

-  **squashfs:** Compressed immutable (read only) container images
   (default in version 2.4)

-  **extfs:** Raw file system writable container images

-  **dir:** Sandbox containers (chroot style directories)

Using the Singularity configuration file, you can control what types of
containers Singularity will support:

::

    # ALLOW CONTAINER ${TYPE}: [BOOL]
    # DEFAULT: yes
    # This feature limits what kind of containers that Singularity will allow
    # users to use (note this does not apply for root).
    allow container squashfs = yes
    allow container extfs = yes
    allow container dir = yes

| One benefit of using container images is that they exist on the
  filesystem as any other file would. This means that POSIX permissions
  are mandatory. Here you can configure Singularity to only “trust”
  containers that are owned by a particular set of users.

::

    # LIMIT CONTAINER OWNERS: [STRING]
    # DEFAULT: NULL
    # Only allow containers to be used that are owned by a given user. If this
    # configuration is undefined (commented or set to NULL), all containers are
    # allowed to be used. This feature only applies when Singularity is running in
    # SUID mode and the user is non-root.
    #limit container owners = gmk, singularity, nobody

note: If you are in a high risk security environment, you may want to
enable this feature. Trusting container images to users could allow a
malicious user to modify an image either before or while being used and
cause unexpected behavior from the kernel (e.g. a `DOS
attack <https://en.wikipedia.org/wiki/Denial-of-service_attack>`__). For
more information, please see: https://lwn.net/Articles/652468/

The configuration file also gives you the ability to limit containers to
specific paths. This is very useful to ensure that only trusted or
blessed container’s are being used (it is also beneficial to ensure that
containers are only being used on performant file systems).

::

    # LIMIT CONTAINER PATHS: [STRING]
    # DEFAULT: NULL
    # Only allow containers to be used that are located within an allowed path
    # prefix. If this configuration is undefined (commented or set to NULL),
    # containers will be allowed to run from anywhere on the file system. This
    # feature only applies when Singularity is running in SUID mode and the user is
    # non-root.
    #limit container paths = /scratch, /tmp, /global

Logging
~~~~~~~

Singularity offers a very comprehensive auditing mechanism via the
system log. For each command that is issued, it prints the UID, PID, and
location of the command. For example, let’s see what happens if we shell
into an image:

::

    $ singularity exec ubuntu true
    $ singularity shell --home $HOME:/ ubuntu
    Singularity: Invoking an interactive shell within container...

    ERROR  : Failed to execv() /.singularity.d/actions/shell, continuing to /bin/sh: No such file or directory
    ERROR  : What are you doing gmk, this is highly irregular!
    ABORT  : Retval = 255

We can then peek into the system log to see what was recorded:

::

    Oct  5 08:51:12 localhost Singularity: action-suid (U=1000,P=32320)> USER=gmk, IMAGE='ubuntu', COMMAND='exec'
    Oct  5 08:53:13 localhost Singularity: action-suid (U=1000,P=32311)> USER=gmk, IMAGE='ubuntu', COMMAND='shell'
    Oct  5 08:53:13 localhost Singularity: action-suid (U=1000,P=32311)> Failed to execv() /.singularity.d/actions/shell, continuing to /bin/sh: No such file or directory
    Oct  5 08:53:13 localhost Singularity: action-suid (U=1000,P=32311)> What are you doing gmk, this is highly irregular!
    Oct  5 08:53:13 localhost Singularity: action-suid (U=1000,P=32311)> Retval = 255

**note: All errors are logged!**

We can also add the argument to any command itself at runtime to see
everything that Singularity is doing. In this case we can run
Singularity in debug mode and request use of the PID namespace so we can
see what Singularity is doing there:

::

    $ singularity --debug shell --pid ubuntu
    Enabling debugging
    Ending argument loop
    Singularity version: 2.3.9-development.gc35b753
    Exec'ing: /usr/local/libexec/singularity/cli/shell.exec
    Evaluating args: '--pid ubuntu'

(snipped to PID namespace implementation)

::

    DEBUG   [U=1000,P=30961]   singularity_runtime_ns_pid()              Using PID namespace: CLONE_NEWPID
    DEBUG   [U=1000,P=30961]   singularity_runtime_ns_pid()              Virtualizing PID namespace
    DEBUG   [U=1000,P=30961]   singularity_registry_get()                Returning NULL on 'DAEMON_START'
    DEBUG   [U=1000,P=30961]   prepare_fork()                            Creating parent/child coordination pipes.
    VERBOSE [U=1000,P=30961]   singularity_fork()                        Forking child process
    DEBUG   [U=1000,P=30961]   singularity_priv_escalate()               Temporarily escalating privileges (U=1000)
    DEBUG   [U=0,P=30961]      singularity_priv_escalate()               Clearing supplementary GIDs.
    DEBUG   [U=0,P=30961]      singularity_priv_drop()                   Dropping privileges to UID=1000, GID=1000 (8 supplementary GIDs)
    DEBUG   [U=0,P=30961]      singularity_priv_drop()                   Restoring supplementary groups
    DEBUG   [U=1000,P=30961]   singularity_priv_drop()                   Confirming we have correct UID/GID
    VERBOSE [U=1000,P=30961]   singularity_fork()                        Hello from parent process
    DEBUG   [U=1000,P=30961]   install_generic_signal_handle()           Assigning generic sigaction()s
    DEBUG   [U=1000,P=30961]   install_generic_signal_handle()           Creating generic signal pipes
    DEBUG   [U=1000,P=30961]   install_sigchld_signal_handle()           Assigning SIGCHLD sigaction()
    DEBUG   [U=1000,P=30961]   install_sigchld_signal_handle()           Creating sigchld signal pipes
    DEBUG   [U=1000,P=30961]   singularity_fork()                        Dropping permissions
    DEBUG   [U=0,P=30961]      singularity_priv_drop()                   Dropping privileges to UID=1000, GID=1000 (8 supplementary GIDs)
    DEBUG   [U=0,P=30961]      singularity_priv_drop()                   Restoring supplementary groups
    DEBUG   [U=1000,P=30961]   singularity_priv_drop()                   Confirming we have correct UID/GID
    DEBUG   [U=1000,P=30961]   singularity_signal_go_ahead()             Sending go-ahead signal: 0
    DEBUG   [U=1000,P=30961]   wait_child()                              Parent process is waiting on child process
    DEBUG   [U=0,P=1]          singularity_priv_drop()                   Dropping privileges to UID=1000, GID=1000 (8 supplementary GIDs)
    DEBUG   [U=0,P=1]          singularity_priv_drop()                   Restoring supplementary groups
    DEBUG   [U=1000,P=1]       singularity_priv_drop()                   Confirming we have correct UID/GID
    VERBOSE [U=1000,P=1]       singularity_fork()                        Hello from child process
    DEBUG   [U=1000,P=1]       singularity_wait_for_go_ahead()           Waiting for go-ahead signal
    DEBUG   [U=1000,P=1]       singularity_wait_for_go_ahead()           Received go-ahead signal: 0
    VERBOSE [U=1000,P=1]       singularity_registry_set()                Adding value to registry: 'PIDNS_ENABLED' = '1'

(snipped to end)

::

    DEBUG   [U=1000,P=1]       envar_set()                               Unsetting environment variable: SINGULARITY_APPNAME
    DEBUG   [U=1000,P=1]       singularity_registry_get()                Returning value from registry: 'COMMAND' = 'shell'
    LOG     [U=1000,P=1]       main()                                    USER=gmk, IMAGE='ubuntu', COMMAND='shell'
    INFO    [U=1000,P=1]       action_shell()                            Singularity: Invoking an interactive shell within container...

    DEBUG   [U=1000,P=1]       action_shell()                            Exec'ing /.singularity.d/actions/shell
    Singularity ubuntu:~> 

Not only do I see all of the configuration options that I (probably
forgot about) previously set, I can trace the entire flow of Singularity
from the first execution of an action (shell) to the final shell into
the container. Each line also describes what is the effective UID
running the command, what is the PID, and what is the function emitting
the debug message.

The above snippet was using the default SetUID program flow with a
container image file named “ubuntu”. For comparison, if we also use the
flag, and snip in the same places, you can see how the effective UID is
never escalated, but we have the same outcome using a sandbox directory
(chroot) style container.

::

    $ singularity -d shell --pid --userns ubuntu.dir/
    Enabling debugging
    Ending argument loop
    Singularity version: 2.3.9-development.gc35b753
    Exec'ing: /usr/local/libexec/singularity/cli/shell.exec
    Evaluating args: '--pid --userns ubuntu.dir/'

| (snipped to PID namespace implementation, same place as above)

::

    DEBUG   [U=1000,P=32081]   singularity_runtime_ns_pid()              Using PID namespace: CLONE_NEWPID
    DEBUG   [U=1000,P=32081]   singularity_runtime_ns_pid()              Virtualizing PID namespace
    DEBUG   [U=1000,P=32081]   singularity_registry_get()                Returning NULL on 'DAEMON_START'
    DEBUG   [U=1000,P=32081]   prepare_fork()                            Creating parent/child coordination pipes.
    VERBOSE [U=1000,P=32081]   singularity_fork()                        Forking child process
    DEBUG   [U=1000,P=32081]   singularity_priv_escalate()               Not escalating privileges, user namespace enabled
    DEBUG   [U=1000,P=32081]   singularity_priv_drop()                   Not dropping privileges, user namespace enabled
    VERBOSE [U=1000,P=32081]   singularity_fork()                        Hello from parent process
    DEBUG   [U=1000,P=32081]   install_generic_signal_handle()           Assigning generic sigaction()s
    DEBUG   [U=1000,P=32081]   install_generic_signal_handle()           Creating generic signal pipes
    DEBUG   [U=1000,P=32081]   install_sigchld_signal_handle()           Assigning SIGCHLD sigaction()
    DEBUG   [U=1000,P=32081]   install_sigchld_signal_handle()           Creating sigchld signal pipes
    DEBUG   [U=1000,P=32081]   singularity_signal_go_ahead()             Sending go-ahead signal: 0
    DEBUG   [U=1000,P=32081]   wait_child()                              Parent process is waiting on child process
    DEBUG   [U=1000,P=1]       singularity_priv_drop()                   Not dropping privileges, user namespace enabled
    VERBOSE [U=1000,P=1]       singularity_fork()                        Hello from child process
    DEBUG   [U=1000,P=1]       singularity_wait_for_go_ahead()           Waiting for go-ahead signal
    DEBUG   [U=1000,P=1]       singularity_wait_for_go_ahead()           Received go-ahead signal: 0
    VERBOSE [U=1000,P=1]       singularity_registry_set()                Adding value to registry: 'PIDNS_ENABLED' = '1'

(snipped to end)

::

    DEBUG   [U=1000,P=1]       envar_set()                               Unsetting environment variable: SINGULARITY_APPNAME
    DEBUG   [U=1000,P=1]       singularity_registry_get()                Returning value from registry: 'COMMAND' = 'shell'
    LOG     [U=1000,P=1]       main()                                    USER=gmk, IMAGE='ubuntu.dir', COMMAND='shell'
    INFO    [U=1000,P=1]       action_shell()                            Singularity: Invoking an interactive shell within container...

    DEBUG   [U=1000,P=1]       action_shell()                            Exec'ing /.singularity.d/actions/shell
    Singularity ubuntu.dir:~> whoami
    gmk
    Singularity ubuntu.dir:~> 

| Here you can see that the output and functionality is very similar,
  but we never increased any privilege and none of the program flow was
  utilized. We had to use a chroot style directory container (as images
  are not supported with the user namespace, but you can clearly see
  that the effective UID never had to change to run this container.
| note: Singularity can natively create and manage chroot style
  containers just like images! The above image was created using the
  command:

Summary
~~~~~~~

Singularity supports multiple modes of operation to meet your security
needs. For most HPC centers, and general usage scenarios, the default
run mode is most effective and featurefull. For the security critical
implementations, the user namespace workflow maybe a better option. It
becomes a balance security and functionality (the most secure systems do
nothing).

The Singularity Config File
---------------------------

| When Singularity is running via the SUID pathway, the configuration
  **must** be owned by the root user otherwise Singularity will error
  out. This ensures that the system administrators have direct say as to
  what functions the users can utilize when running as root. If
  Singularity is installed as a non-root user, the SUID components are
  not installed, and the configuration file can be owned by the user
  (but again, this will limit functionality).
| The Configuration file can be found at . The template in the
  repository is located at . It is generally self documenting but there
  are several things to pay special attention to:

Parameters
~~~~~~~~~~

| This parameter toggles the global ability to execute the SETUID (SUID)
  portion of the code if it exists. As mentioned earlier, if the SUID
  features are disabled, various Singularity features will not function
  (e.g. mounting of the Singularity image file format).
| You can however disable SUID support **iff** (if and only if) you do
  not need to use the default Singularity image file format and if your
  kernel supports user namespaces and you choose to use user namespaces.
| note: as of the time of this writing, the user namespace is rather
  buggy

| While the PID namespace is a neat feature, it does not have much
  practical usage in an HPC context so it is recommended to disable this
  if you are running on an HPC system where a resource manager is
  involved as it has been known to cause confusion on some kernels with
  enforcement of user limits.
| Even if the PID namespace is enabled by the system administrator here,
  it is not implemented by default when running containers. The user
  will have to specify they wish to implement un-sharing of the PID
  namespace as it must fork a child process.

The overlay file system creates a writable substrate to create bind
points if necessary. This feature is very useful when implementing bind
points within containers where the bind point may not already exist so
it helps with portability of containers. Enabling this option has been
known to cause some kernels to panic as this feature maybe present
within a kernel, but has not proved to be stable as of the time of this
writing (e.g. the Red Hat 7.2 kernel).

All of these options essentially do the same thing for different files
within the container. This feature updates the described file (, , and
respectively) to be updated dynamically as the container is executed. It
uses binds and modifies temporary files such that the original files are
not manipulated.

These configuration options control the mounting of these file systems
within the container and of course can be overridden by the system
administrator (e.g. the system admin decides not to include the /dev
tree inside the container). In most useful cases, these are all best to
leave enabled.

This feature will parse the host’s mounted file systems and attempt to
replicate all mount points within the container. This maybe a desirable
feature for the lazy, but it is generally better to statically define
what bind points you wish to encapsulate within the container by hand
(using the below “bind path” feature).

| With this configuration directive, you can specify any number of bind
  points that you want to extend from the host system into the
  container. Bind points on the host file system must be either real
  files or directories (no special files supported at this time). If the
  overlayFS is not supported on your host, or if in this configuration
  file, a bind point must exist for the file or directory within the
  container.
| The syntax for this consists of a bind path source and an optional
  bind path destination separated by a colon. If no bind path
  destination is specified, the bind path source is used also as the
  destination.

| In addition to the system bind points as specified within this
  configuration file, you may also allow users to define their own bind
  points inside the container. This feature is used via multiple command
  line arguments (e.g. , , and ) so disabling user bind control will
  also disable those command line options.
| Singularity will automatically disable this feature if the host does
  not support the prctl option . In addition, must be set to and the
  host system must support overlayFS (generally kernel versions 3.18 and
  later) for users to bind host directories to bind points that do not
  already exist in the container.

| With some versions of autofs, Singularity will fail to run with a “Too
  many levels of symbolic links” error. This error happens by way of a
  user requested bind (done with -B/–bind) or one specified via the
  configuration file. To handle this, you will want to specify those
  paths using this directive. For example:

::

    autofs bug path = /share/PI

Logging
~~~~~~~

In order to facilitate monitoring and auditing, Singularity will
syslog() every action and error that takes place to the syslog facility.
You can define what to do with those logs in your syslog configuration.

Loop Devices
~~~~~~~~~~~~

| Singularity images have file systems embedded within them, and thus to
  mount them, we need to convert the raw file system image (with
  variable offset) to a block device. To do this, Singularity utilizes
  the block devices on the host system and manages the devices
  programmatically within Singularity itself. Singularity also uses the
  loop device flag which tells the kernel to automatically free the loop
  device when there are no more open file descriptors to the device
  itself.
| Earlier versions of Singularity managed the loop devices via a
  background watchdog process, but since version 2.2 we leverage the
  functionality and we forego the watchdog process. Unfortunately, this
  means that some older Linux distributions are no longer supported
  (e.g. RHEL <= 5).
| Given that loop devices are consumable (there are a limited number of
  them on a system), Singularity attempts to be smart in how loop
  devices are allocated. For example, if a given user executes a
  specific container it will bind that image to the next available loop
  device automatically. If that same user executes another command on
  the same container, it will use the loop device that has already been
  allocated instead of binding to another loop device. Most Linux
  distributions only support 8 loop devices by default, so if you find
  that you have a lot of different users running Singularity containers,
  you may need to increase the number of loop devices that your system
  supports by doing the following:
| Edit or create the file and add the following line:

::

    options loop max_loop=128

After making this change, you should be able to reboot your system or
unload/reload the loop device as root using the following commands:

::

    # modprobe -r loop
    # modprobe loop

Container Checks
----------------

New to Singularity 2.4 is the ability to, on demand, run container
“checks,” which can be anything from a filter for sensitive information,
to an analysis of content on the filesystem. Checks are installed with
Singularity, managed by the administrator, and `available to the
user <http://singularity.lbl.gov/docs-user-checks>`__.

What is a check?
~~~~~~~~~~~~~~~~

| Broadly, a check is a script that is run over a mounted filesystem,
  primary with the purpose of checking for some security issue. This
  process is tightly controlled, meaning that the script names in the
  `checks <https://github.com/singularityware/singularity/tree/development/libexec/helpers/checks>`__
  folder are hard coded into the script
  `check.sh <https://github.com/singularityware/singularity/blob/development/libexec/helpers/check.sh>`__.
  The flow of checks is the following:

-  the user calls to invoke
   `check.exec <https://github.com/singularityware/singularity/blob/development/libexec/cli/check.exec>`__

-  specification of (3), (2), or (1) sets the level to perform. The
   level is a filter, meaning that a level of 3 will include 3,2,1, and
   a level of 1 (high) will only call checks of high priority.

-  specification of will allow the user (or execution script) to specify
   a kind of check. This is primarily to allow for extending the checks
   to do other types of things. For example, for this initial batch,
   these are all considered checks. The
   `check.help <https://github.com/singularityware/singularity/blob/development/libexec/cli/check.help>`__
   displays examples of how the user specifies a tag:

::

        # Perform all default checks, these are the same
        $ singularity check ubuntu.img
        $ singularity check --tag default ubuntu.img

        # Perform checks with tag "clean"
        $ singularity check --tag clean ubuntu.img

| A check should be a bash (or other) script that will perform some
  action. The following is required:
| **Relative to SINGULARITY\_ROOTFS** The script must perform check
  actions relative to . For example, in python you might change
  directory to this location:

::

    import os
    base = os.environ["SINGULARITY_ROOTFS"]
    os.chdir(base)

or do the same in bash:

::

    cd $SINGULARITY_ROOTFS
    ls $SINGULARITY_ROOTFS/var

| Since we are doing a mount, all checks must be static relative to this
  base, otherwise you are likely checking the host system.
| **Verbose** The script should indicate any warning/message to the user
  if the check is found to have failed. If pass, the check’s name and
  status will be printed, with any relevant information. For more
  thorough checking, you might want to give more verbose output.
| **Return Code** The script return code of “success” is defined in
  `check.sh <http://singularity.lbl.gov/check.sh>`__, and other return
  codes are considered not success. When a non success return code is
  found, the rest of the checks continue running, and no action is
  taken. We might want to give some admin an ability to specify a check,
  a level, and prevent continuation of the build/bootstrap given a fail.
| **Check.sh** The script level, path, and tags should be added to
  `check.sh <http://singularity.lbl.gov/check.sh>`__ in the following
  format:

::

    ##################################################################################
    # CHECK SCRIPTS
    ##################################################################################

    #        [SUCCESS] [LEVEL]  [SCRIPT]                                                                         [TAGS]
    execute_check    0    HIGH  "bash $SINGULARITY_libexecdir/singularity/helpers/checks/1-hello-world.sh"       security
    execute_check    0     LOW  "python $SINGULARITY_libexecdir/singularity/helpers/checks/2-cache-content.py"   clean
    execute_check    0    HIGH  "python $SINGULARITY_libexecdir/singularity/helpers/checks/3-cve.py"             security

The function will compare the level () with the user specified (or
default) and execute the check only given it is under the specified
threshold, and (not yet implemented) has the relevant tag. The success
code is also set here with . Currently, we aren’t doing anything with
and thus perform all checks.

How to tell users?
~~~~~~~~~~~~~~~~~~

If you add a custom check that you want for your users to use, you
should tell them about it. Better yet, `tell
us <https://github.com/singularityware/singularity/issues>`__ about it
so it can be integrated into the Singularity software for others to use.

Troubleshooting
---------------

This section will help you debug (from the system administrator’s
perspective) Singularity.

Not installed correctly, or installed to a non-compatible location
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| Singularity must be installed by root into a location that allows for
  SUID programs to be executed (as described above in the installation
  section of this manual). If you fail to do that, you may have user’s
  reporting one of the following error conditions:

::

    ERROR  : Singularity must be executed in privileged mode to use images
    ABORT  : Retval = 255

::

    ERROR  : User namespace not supported, and program not running privileged.
    ABORT  : Retval = 255

::

    ABORT  : This program must be SUID root
    ABORT  : Retval = 255

If one of these errors is reported, it is best to check the installation
of Singularity and ensure that it was properly installed by the root
user onto a local file system.

Installation Environments
=========================

Singularity on HPC
------------------

| One of the architecturally defined features in Singularity is that it
  can execute containers like they are native programs or scripts on a
  host computer. As a result, integration with schedulers is simple and
  runs exactly as you would expect. All standard input, output, error,
  pipes, IPC, and other communication pathways that locally running
  programs employ are synchronized with the applications running locally
  within the container.
| Additionally, because Singularity is not emulating a full hardware
  level virtualization paradigm, there is no need to separate out any
  sandboxed networks or file systems because there is no concept of
  user-escalation within a container. Users can run Singularity
  containers just as they run any other program on the HPC resource.

Workflows
~~~~~~~~~

We are in the process of developing Singularity Hub, which will allow
for generation of workflows using Singularity containers in an online
interface, and easy deployment on standard research clusters (e.g.,
SLURM, SGE). Currently, the Singularity core software is installed on
the following research clusters, meaning you can run Singularity
containers as part of your jobs:

-  The `Sherlock cluster <http://sherlock.stanford.edu/>`__ at `Stanford
   University <https://srcc.stanford.edu/>`__

-  `SDSC Comet and
   Gordon <https://www.xsede.org/news/-/news/item/7624>`__ (XSEDE)

-  `MASSIVE M1 M2 and M3 <http://docs.massive.org.au/index.html>`__
   (Monash University and Australian National Merit Allocation Scheme)

Another result of the Singularity architecture is the ability to
properly integrate with the Message Passing Interface (MPI). Work has
already been done for out of the box compatibility with Open MPI (both
in Open MPI v2.1.x as well as part of Singularity). The Open
MPI/Singularity workflow works as follows:

#. mpirun is called by the resource manager or the user directly from a
   shell

#. Open MPI then calls the process management daemon (ORTED)

#. The ORTED process launches the Singularity container requested by the
   mpirun command

#. Singularity builds the container and namespace environment

#. Singularity then launches the MPI application within the container

#. The MPI application launches and loads the Open MPI libraries

#. The Open MPI libraries connect back to the ORTED process via the
   Process Management Interface (PMI)

#. At this point the processes within the container run as they would
   normally directly on the host.

| This entire process happens behind the scenes, and from the user’s
  perspective running via MPI is as simple as just calling mpirun on the
  host as they would normally.
| Below are example snippets of building and installing OpenMPI into a
  container and then running an example MPI program through Singularity.

Tutorials
^^^^^^^^^

-  `Using Host libraries: GPU drivers and OpenMPI BTLs
    <http://singularity.lbl.gov/tutorial-gpu-drivers-open-mpi-mtls>`__

MPI Development Example
^^^^^^^^^^^^^^^^^^^^^^^

**What are supported Open MPI Version(s)?** To achieve proper
container’ized Open MPI support, you should use Open MPI version 2.1.
There are however three caveats:

#. Open MPI 1.10.x may work but we expect you will need exactly matching
   version of PMI and Open MPI on both host and container (the 2.1
   series should relax this requirement)

#. Open MPI 2.1.0 has a bug affecting compilation of libraries for some
   interfaces (particularly Mellanox interfaces using libmxm are known
   to fail). If your in this situation you should use the master branch
   of Open MPI rather than the release.

#. Using Open MPI 2.1 does not magically allow your container to connect
   to networking fabric libraries in the host. If your cluster has, for
   example, an infiniband network you still need to install OFED
   libraries into the container. Alternatively you could bind mount both
   Open MPI and networking libraries into the container, but this could
   run afoul of glib compatibility issues (its generally OK if the
   container glibc is more recent than the host, but not the other way
   around)

Code Example using Open MPI 2.1.0 Stable
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

| 

::

    $ # Include the appropriate development tools into the container (notice we are calling
    $ # singularity as root and the container is writable)
    $ sudo singularity exec -w /tmp/Centos-7.img yum groupinstall "Development Tools"
    $
    $ # Obtain the development version of Open MPI
    $ wget https://www.open-mpi.org/software/ompi/v2.1/downloads/openmpi-2.1.0.tar.bz2
    $ tar jtf openmpi-2.1.0.tar.bz2
    $ cd openmpi-2.1.0
    $
    $ singularity exec /tmp/Centos-7.img ./configure --prefix=/usr/local
    $ singularity exec /tmp/Centos-7.img make
    $
    $ # Install OpenMPI into the container (notice now running as root and container is writable)
    $ sudo singularity exec -w -B /home /tmp/Centos-7.img make install
    $
    $ # Build the OpenMPI ring example and place the binary in this directory
    $ singularity exec /tmp/Centos-7.img mpicc examples/ring_c.c -o ring
    $
    $ # Install the MPI binary into the container at /usr/bin/ring
    $ sudo singularity copy /tmp/Centos-7.img ./ring /usr/bin/
    $
    $ # Run the MPI program within the container by calling the MPIRUN on the host
    $ mpirun -np 20 singularity exec /tmp/Centos-7.img /usr/bin/ring

Code Example using Open MPI git master
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The previous example (using the Open MPI 2.1.0 stable release) should
work fine on most hardware but if you have an issue, try running the
example below (using the Open MPI Master branch):

::

    $ # Include the appropriate development tools into the container (notice we are calling
    $ # singularity as root and the container is writable)
    $ sudo singularity exec -w /tmp/Centos-7.img yum groupinstall "Development Tools"
    $
    $ # Clone the OpenMPI GitHub master branch in current directory (on host)
    $ git clone https://github.com/open-mpi/ompi.git
    $ cd ompi
    $
    $ # Build OpenMPI in the working directory, using the tool chain within the container
    $ singularity exec /tmp/Centos-7.img ./autogen.pl
    $ singularity exec /tmp/Centos-7.img ./configure --prefix=/usr/local
    $ singularity exec /tmp/Centos-7.img make
    $
    $ # Install OpenMPI into the container (notice now running as root and container is writable)
    $ sudo singularity exec -w -B /home /tmp/Centos-7.img make install
    $
    $ # Build the OpenMPI ring example and place the binary in this directory
    $ singularity exec /tmp/Centos-7.img mpicc examples/ring_c.c -o ring
    $
    $ # Install the MPI binary into the container at /usr/bin/ring
    $ sudo singularity copy /tmp/Centos-7.img ./ring /usr/bin/
    $
    $ # Run the MPI program within the container by calling the MPIRUN on the host
    $ mpirun -np 20 singularity exec /tmp/Centos-7.img /usr/bin/ring


    Process 0 sending 10 to 1, tag 201 (20 processes in ring)
    Process 0 sent to 1
    Process 0 decremented value: 9
    Process 0 decremented value: 8
    Process 0 decremented value: 7
    Process 0 decremented value: 6
    Process 0 decremented value: 5
    Process 0 decremented value: 4
    Process 0 decremented value: 3
    Process 0 decremented value: 2
    Process 0 decremented value: 1
    Process 0 decremented value: 0
    Process 0 exiting
    Process 1 exiting
    Process 2 exiting
    Process 3 exiting
    Process 4 exiting
    Process 5 exiting
    Process 6 exiting
    Process 7 exiting
    Process 8 exiting
    Process 9 exiting
    Process 10 exiting
    Process 11 exiting
    Process 12 exiting
    Process 13 exiting
    Process 14 exiting
    Process 15 exiting
    Process 16 exiting
    Process 17 exiting
    Process 18 exiting
    Process 19 exiting

Image Environment
-----------------

Directory access
~~~~~~~~~~~~~~~~

By default Singularity tries to create a seamless user experience
between the host and the container. To do this, Singularity makes
various locations accessible within the container automatically. For
example, the user’s home directory is always bound into the container as
is /tmp and /var/tmp. Additionally your current working directory
(cwd/pwd) is also bound into the container iff it is not an operating
system directory or already accessible via another mount. For almost all
cases, this will work flawlessly as follows:

::

    $ pwd
    /home/gmk/demo
    $ singularity shell container.img 
    Singularity/container.img> pwd
    /home/gmk/demo
    Singularity/container.img> ls -l debian.def 
    -rw-rw-r--. 1 gmk gmk 125 May 28 10:35 debian.def
    Singularity/container.img> exit
    $ 

| For directory binds to function properly, there must be an existing
  target endpoint within the container (just like a mount point). This
  means that if your home directory exists in a non-standard base
  directory like “/foobar/username” then the base directory “/foobar”
  must already exist within the container.
| Singularity will not create these base directories! You must enter the
  container with the option being set, and create the directory
  manually.

Singularity will try to replicate your current working directory within
the container. Sometimes this is straight forward and possible, other
times it is not (e.g. if the base dir of your current working directory
does not exist). In that case, Singularity will retain the file
descriptor to your current directory and change you back to it. If you
do a ‘pwd’ within the container, you may see some weird things. For
example:

::

    $ pwd
    /foobar
    $ ls -l
    total 0
    -rw-r--r--. 1 root root 0 Jun  1 11:32 mooooo
    $ singularity shell ~/demo/container.img 
    WARNING: CWD bind directory not present: /foobar
    Singularity/container.img> pwd
    (unreachable)/foobar
    Singularity/container.img> ls -l
    total 0
    -rw-r--r--. 1 root root 0 Jun  1 18:32 mooooo
    Singularity/container.img> exit
    $ 

But notice how even though the directory location is not resolvable, the
directory contents are available.

Standard IO and pipes
~~~~~~~~~~~~~~~~~~~~~

Singularity automatically sends and receives all standard IO from the
host to the applications within the container to facilitate expected
behavior from the interaction between the host and the container. For
example:

::

    $ cat debian.def | singularity exec container.img grep 'MirrorURL'
    MirrorURL "http://ftp.us.debian.org/debian/"
    $ 
    Making changes to the container (writable)
    By default, containers are accessed as read only. This is both to enable parallel container execution (e.g. MPI). To enter a container using exec, run, or shell you must pass the --writable flag in order to open the image as read/writable.

Containing the container
~~~~~~~~~~~~~~~~~~~~~~~~

By providing the argument to , or you will find that shared directories
are no longer shared. For example, the user’s home directory is
writable, but it is non-persistent between non-overlapping runs.

License
-------

::

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions are met:
     
    (1) Redistributions of source code must retain the above copyright notice,
    this list of conditions and the following disclaimer.
     
    (2) Redistributions in binary form must reproduce the above copyright notice,
    this list of conditions and the following disclaimer in the documentation
    and/or other materials provided with the distribution.
     
    (3) Neither the name of the University of California, Lawrence Berkeley
    National Laboratory, U.S. Dept. of Energy nor the names of its contributors
    may be used to endorse or promote products derived from this software without
    specific prior written permission.
     
    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
    AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
    IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
    DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE
    FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
    DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
    SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
    CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
    OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

    You are under no obligation whatsoever to provide any bug fixes, patches, or
    upgrades to the features, functionality or performance of the source code
    ("Enhancements") to anyone; however, if you choose to make your Enhancements
    available either publicly, or directly to Lawrence Berkeley National
    Laboratory, without imposing a separate written license agreement for such
    Enhancements, then you hereby grant the following license: a  non-exclusive,
    royalty-free perpetual license to install, use, modify, prepare derivative
    works, incorporate into other computer software, distribute, and sublicense
    such enhancements or derivative works thereof, in binary and source code form.

    If you have questions about your rights to use or distribute this software,
    please contact Berkeley Lab's Innovation & Partnerships Office at
    IPO@lbl.gov.

    NOTICE.  This Software was developed under funding from the U.S. Department of
    Energy and the U.S. Government consequently retains certain rights. As such,
    the U.S. Government has been granted for itself and others acting on its
    behalf a paid-up, nonexclusive, irrevocable, worldwide license in the Software
    to reproduce, distribute copies to the public, prepare derivative works, and
    perform publicly and display publicly, and to permit other to do so. 

In layman terms...
~~~~~~~~~~~~~~~~~~

In addition to the (already widely used and very free open source)
standard BSD 3 clause license, there is also wording specific to
contributors which ensures that we have permission to release,
distribute and include a particular contribution, enhancement, or fix as
part of Singularity proper. For example any contributions submitted will
have the standard BSD 3 clause terms (unless specifically and otherwise
stated) and that the contribution is comprised of original new code that
the contributor has authority to contribute.
