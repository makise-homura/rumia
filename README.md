# rumia

RUMIA - Remote Unit MonItoring Application

![RUMIA screenshot](rumia.png)

## Purpose

This is a little but extensible, handy cross-platform BASH script that may be
used, at the moment, to monitor several stuff (availability, uptime, disk
consistency, CPU/HDD temperatures, battery charge level) on several remote
computers using only the text console (either computer screen or ssh session),
without a requirement of any compiler or build system (and also without ability
of doing this through SNMP, IPMI or HTTP/HTTPS, but you should rethink your life
if you are up to do it with plain `bash` + `coreutils` and virtually nothing
more).

Additionally, it contains another script that can execute arbitrary command on
the chosen remote computers.

## Requirements

* bash
* coreutils
* ssh
* an ANSI-compatible UTF-8 console (e.g. putty/xterm)
* anything you specify in `require` file for certain computer(s)

## Configuration

### Configuring RUMIA itself

You should prepare a config file (see `rumia.conf.example` as an example)
as following:

* `PREFIX` should be a path where RUMIA is installed. It should contain language
files, and, in typical installation, `computers.d` and `templates.d` dirs.

* `COMPUTERS_D` should be a path to `computers.d` directory. May be relative
to `PREFIX` if BASH variable substitution is used.

* `LOGFILE` should be a path to log file which will be appended with a complete
temperature information assembled during each shot. May be relative to `PREFIX`
if BASH variable substitution is used. Log file consists of lines that have
the current time of a shot, followed by several tab-separated records in format
like `hostname:N=T`, where `hostname` is a hostname of a machine, `N` is
a number of the temperature sensor, and `T` is a temperature that has been read
from this sensor. Log file remains closed all time except the exact moments
when information is appended to it, so you may freely delete or move it even
when RUMIA is running: no stalled file handles will be left there.

* `RUMIA_LANG` should be set according to a desired language. Currently
available languages are `en_US` and `ru_RU`. The corresponding
`language.$RUMIA_LANG` file must exist in `$PREFIX`.

* `NAME_LENGTH` should be a length of computer's name column. If not defined,
not numerical, or less than 8, defaults to 8.

* `MARKER` should be a name of a marker file that makes a computer visible
to RUMIA. Usually this is `temp_enabled`. For further information, see below
for a meaning of `temp_enabled` file.

* `SMARKER` should be a name of a marker file that makes a computer visible
to `execute_cmd`. Usually this is `ssh_enabled`. For further information, see
below for a meaning of `ssh_enabled` file.

* `DELAY` should be set to a seconds between end of one shot and begin of the
following (Note: it's not the time between start of each shot, so actual period
time is equal to time consumed by one shot and this delay).

* `SSH_OPTS` should be a bunch of additional options passed to each call to
`ssh` command. Used primarily for setting timeouts.

All variables are set using generic BASH syntax, e.g. no spaces should be
between variable name, "=" sign, and variable value; values containing
spaces must be enclosed in double or single quotes; BASH variable substitutions
are allowed, etc.

In the example of `rumia.conf`, any of these variables may be redefined while
starting `rumia`. For example, if RUMIA is configured to use Russian language,
but you want to launch it in one shot mode with English output, you may run:

```bash
RUMIA_LANG=en_US ./rumia --oneshot
```

Unless `--config` specified in command line, RUMIA reads the following files
as its config files (if any of them exist):

* `/etc/rumia.conf`: system-wide RUMIA config file
* `~/.config/rumia.conf`: user-specific RUMIA config file
* `~/.rumiarc`: user-specific additions to RUMIA config file
* `./rumia.conf`: directory-specific RUMIA config file

Each next file take precedence over the previous ones (overriding the
definitions made in it).

### Letting RUMIA know about remote computers

For each monitored computer, a specific subdirectory should be made inside
`computers.d` directory (there are various examples of it inside
`computers.d.example` subdirectory of RUMIA). Directory should have a name
that consists of 2-digit order mark, then a hyphen, and then a hostname
of a machine to which this directory is related (like `12-example`). This
directory may contain the following files or symbolic links:

* Marker files. These files are generally zero-sized and made by `touch`
command, and removed by `rm` (obviously).

    * `temp_enabled`: Let RUMIA poll this computer at all. Every computer that
    does not have this marker is skipped while gathering information. The name
    of this marker file might be redefined by a `MARKER` variable in the config
    file or in the command line; thus you may have the single `computers.d`
    directory for different polling strategies. E.g. you may have a generic
    marker, `temp_enabled`, and `high_attention` for several computers that
    need high attention, and so, high polling rate. Then you just may have two
    sessions (e.g. `tty1` and `tty2`), and run `DELAY=600 /opt/rumia/rumia`
    in `tty1`, and `MARKER=high_attention DELAY=60 /opt/rumia/rumia` in `tty2`,
    and then, in `tty1` you'll have all your computers that have `temp_enabled`
    in their directories, and in `tty2` you'll have a subset of them, of those
    who have `high_attention` marker file in their directories, and the latter
    will be polled ten times more often. Of course, it might be not a subset,
    but a completely different set that may or may not have a non-empty
    intersection with the first one.

    * `ssh_enabled`: The same for `execute_cmd`. Every computer that
    does not have this marker is skipped when `ssh_enabled is called`. As well
    as with `temp_enabled`, the name of this marker file might be redefined
    by a `SMARKER` variable in the config file or in the command line; thus
    you may have the single `computers.d` directory for different SSH commands.
    E.g. you may have a generic marker, `ssh_enabled`, and `ssh_full_enabled`
    for several computers that have more advanced set of binaries available.
    Then you just may call `/opt/rumia/execute_cmd` for generic SSH commands,
    and `SMARKER=ssh_full_enabled /opt/rumia/execute_cmd` for more advanced
    commands to execute on computers that support them.

    * `absent`: Let RUMIA know that this computer is intentionally absent. If
    it can't be reached by `ping_cmd`, it produces "Sleep" message in the table
    instead of "Dead", so it could be considered not a failure that needs
    attention.

    * other `*_enabled` files: useful when running `execute_cmd` or `rumia`
    with non-default marker files by using `MARKER` and `SMARKER` variables.

* Data files. These files are generally symlinks to data files inside
`templates.d` subdirectory, except when a file is supposed to be unique (e.g.
SSH private key or computer's fancy name).

    * `key`: private SSH key to use while connecting to computer. To
    successfully connect to computer, you must have public key of specified
    user in his/her `authorized_keys` file, and host fingerprint in
    `known_hosts` on the host where you run RUMIA. For latter one, you could
    just connect once from host where you run RUMIA, to the desired computer,
    and accept the certificate.

    * `fancyname`: a fancy name for a computer that may be used instead of its
    hostname in the table (unless `--nofancynames` is specified, or this file
    doesn't exist). For example, it may be a short form of hostname (e.g.
    "div193-host195.organization.com" could have a fancy name of "Host 195");
    or it could be a beautifized name using symbols that are not allowed in
    a typical hostname (e.g. host "twilight-sparkle" may have fancy name of
    "Твайлайт" in Russian language). If fancy name or hostname (when fancy name
    is not used) is longer than 8 characters, it will be truncated.

    * `temp_nodes`: Number of temperatures that are readable from a computer.
    If missing, no temperature or battery level check is performed. Usually it
    is a symlink to `../../templates.d/nodes.*` file which contains the desired
    number.

    * `temp_alert`: Alerting temperature value. Temperatures lower that that
    are shown in gray, higher (but below critical) are shown in yellow. Has
    no effect if `temp_enabled` or `temp_nodes` is missing. Usually it is
    a symlink to `../../templates.d/temp_alert.*` file which contains the
    desired temperature.

    * `temp_crit`: As above, but a critical temperature value. Temperatures
    higher that that are shown in red. Has no effect if `temp_enabled`
    or `temp_nodes` is missing. Usually it is a symlink to
    `../../templates.d/temp_crit.*` file which contains the desired
    temperature.

    * `require`: A list of binaries (one for a line) which is to be checked
    for existence on the host where RUMIA is running when a certain computer
    is polled. Each of them is examined using `which` command, and if at least
    one of them isn't found, then polling of this computer is skipped,
    and the corresponding message is displayed in the table.

    * `ssh_user`: User name for SSH session. Has an effect only if `ssh_cmd`
    is a symlink to `ssh_cmd.sshpass`.

    * `ssh_password`: User password for SSH session. Has an effect only if
    `ssh_cmd` is a symlink to `ssh_cmd.sshpass`.

    * `vpngw_host`: A host to which `ssh` must connect to achieve access to the
    specified computer. Has an effect only if `ssh_cmd` is a symlink to
    `ssh_cmd.gateway`, `ssh_cmd.recursive`, or `ssh_cmd.gateway+recursive`
    script, and/or `ping_cmd` is a symlink to `ping_cmd.recursive`, or
    `ping_cmd.gateway+recursive`.

    * `vpngw_port`: A port to which `ssh` must connect to achieve access to the
    specified computer. Has an effect only if `ssh_cmd` is a symlink to
    `ssh_cmd.gateway` or `ssh_cmd.gateway+recursive` script, and/or `ping_cmd`
    is a symlink to `ping_cmd.gateway+recursive`.

    * `vpngw_user`: An username using which `ssh` must connect to achieve
    access to the specified computer. Has an effect only if `ssh_cmd` is
    a symlink to `ssh_cmd.gateway` or `ssh_cmd.gateway+recursive` script,
    and/or `ping_cmd` is a symlink to `ping_cmd.gateway+recursive`.

    * `vpngw_keydir`: A directory on gateway where public keys for subordinate
    computers are located. Should be full path, but should not contain trailing
    slash. Has an effect only if `ssh_cmd` is a symlink to
    `ssh_cmd.gateway+recursive` script.

* Command files. These files are generally symlinks to the files inside
`templates.d` subdirectory which are bash scripts (they should have appropriate
permissions) that perform some command on a remote computer.
each of them could use the following three exported environment variables:
`$MACHINE` (a name of computer, as it is specified in the name of corresponding
directory inside `computers.d`), `$SCRIPTPATH` (which is the actual path to
subdirectory related to this computer under `computers.d`), and `$SSH_OPTS`
(which is the additional options passed to SSH command, if used). If you have
a specific requirement, you may put your own script(s) in `templates.d`, and
link these files to it.

    * `ssh_cmd`: Prints a string to execute SSH command on remote computer.
    Should never fail on properly configured RUMIA. Could be symlinked to:

        * `../../templates.d/ssh_cmd.local`: a simple SSH to remote computer
        as user `user` (if not exist, defaults to `root`), using `key` as a
        private key.

        * `../../templates.d/ssh_cmd.gateway`: SSH to remote computer using
        a specific host, port and user, that are specified in `vpngw_host`,
        `vpngw_port`, and `vpngw_user` (if not exist, defaults to `root`),
        correspondingly. `key` is stll a private key.

        * `../../templates.d/ssh_cmd.recursive`: SSH through other SSH tunnel.
        First we connect to `vpngw_host` using `key` as user `vpngw_user`
        (if not exist, defaults to `root`), and then, after we connected,
        we proceed to connect to specified hostname as user `user` (if not
        exist, defaults to `root`), using `$MACHINE.pk` key, which
        is located on `vpngw_host` in directory `vpngw_keydir` (if not exist,
        defaults to `/root/keys`).

        * `../../templates.d/ssh_cmd.gateway+recursive`: a combination of above
        two methods. First we connect to `vpngw_host` port `vpngw_port` as user
        `vpngw_user` (if not exist, defaults to `root`), using `key`, and then,
        after we connected, we proceed to connect to specified hostname as user
        `user` (if not exist, defaults to `root`), using `$MACHINE.pk` key,
        which is located on `vpngw_host` in directory `vpngw_keydir` (if not
        exist, defaults to `/root/keys`).

        * `../../templates.d/ssh_cmd.sshpass`: a simple SSH to remote computer
        as `ssh_user` (if not exist, defaults to `root`) using `ssh_password`
        as SSH password. Obvioulsy it is absolutely insecure, avoid using this
        method by any measures. Note: `require` should contain "`sshpass`" for
        this, and `sshpass` utility should be installed on the computer where
        RUMIA is running.

    * `ping_cmd`: Check an availability of remote computer. If machine is
    unavailable, then no further polling is performed. Should print nothing,
    and return 0 on success and non-null on failure. Could be symlinked to:

        * `../../templates.d/ping_cmd.generic`: a generic ping of remote
        computer.

        * `../../templates.d/ping_cmd.by_ssh`: "pinging" a machine by opening
        an SSH session to it (using `ssh_cmd`'s output) and checking if `sudo`
        call into `sudo_user` (if not exist, defaults to `root`) is successful.
        Useful if you connect to it through a gateway (using `ssh_cmd.gateway`
        script).

        * `../../templates.d/ping_cmd.recursive`: a ping through SSH tunnel.
        First we connect to `vpngw_host` using `key` as user `vpngw_user`
        (if not exist, defaults to `root`), and then, after we connected,
        we just ping specified hostname. Useful if you connect to it
        recursively (using `ssh_cmd.recursive` script).

        * `../../templates.d/ping_cmd.gateway+recursive`: a ping through SSH
        tunnel via gateway. First we connect to `vpngw_host` port `vpngw_port`
        using `key` as user `vpngw_user` (if not exist, defaults to `root`),
        and then, after we connected, we just ping specified hostname. Useful
        if you connect to it recursively through gateway (using
        `ssh_cmd.gateway+recursive` script).

    * `uptime_cmd`: Print an uptime of remote computer. Return value does not
    matter. Could be symlinked to:

        * `../../templates.d/uptime_cmd.generic`: calling GNU `uptime` command
        through an SSH session to remote computer (using `ssh_cmd`'s output).

        * `../../templates.d/uptime_cmd.windows`: calling Windows `uptime.exe`
        command through an SSH session to remote computer (using `ssh_cmd`'s
        output). You may download the corresponding `uptime.exe` binary
        [here](http://barnyard.syr.edu/~vefatica/#UPTIME). It is not the one
        made by Microsoft, since Microsoft's one doesn't work if being run
        by unprivileged user.

        * `../../templates.d/uptime_cmd.procfs`: using data from `/proc/uptime`
        file obtained through an SSH session to remote computer (using
        `ssh_cmd`'s output). Useful if you have BASH with enabled procfs
        on Windows, but don't have any suitable `uptime.exe`.

    * `cpuget_cmd`: Print a CPU utilization of remote computer. Return value
    does not matter. Should print exactly 6 characters, of which leading and
    trailing ones are spaces, and should not produce a newline after that (so
    actual output should look like what commands `echo -n " 95 % "`,
    `echo -n "  2 % "`, or `echo -n " 100% "` produce. Could be symlinked to:

        * `../../templates.d/cpuget_cmd.mpstat`: calling `mpstat` command
        through an SSH session to remote computer (using `ssh_cmd`'s output).
        Output will be colorized: 0..9% is light gray, 10..79% is cyan,
        80..100% is bright white.

        * `../../templates.d/cpuget_cmd.csvlog`: analyzing files specified
        in `logfile` through an SSH session to remote computer (using
        `ssh_cmd`'s output). `logfile` may contain wildcards, like
        `/c/PerfLogs/cpulog_*.csv`, but may not contain spaces. Useful if you
        have Windows and use `logman` to dump CPU load to specified files
        in format like `"09/22/2023 23:35:39.309","69.829614488284107665"`.
        Output will be colorized: 0..9% is light gray, 10..79% is cyan,
        80..100% is bright white.

    * `tempget_cmd`: Print a temperature of remote computer's sensor number
    `$1`, starting from 0, or print "`99.9`" on error. Should print exactly 4
    characters (but may produce a newline). Return value does not matter.
    Maximum 7 temperature sensors are now supported, and 6 if the last position
    is occupied by battery sensor. Actual number of sensors are specified in
    `temp_nodes` file inside computer's subdirectory in `computers.d`. Could be
    symlinked to:

        * `../../templates.d/tempget_cmd.generic`: Call `get_temperature`
        by SSH (using `ssh_cmd`'s output) passing the first parameter to it.

        * `../../templates.d/tempget_cmd.hdd`: Query `/dev/sda` (if first
        parameter is 0) or `/dev/sdb` (if first parameter is 1) for its
        temperature using `smartctl`. Note: `smartctl` utility should be
        installed on the remote computer.

        * `../../templates.d/tempget_cmd.csvlog`: Analyze a file specified
        in `templogfile` through an SSH session to remote computer (using
        `ssh_cmd`'s output). `templogfile` may not contain wildcards and
        spaces, and should look like `/c/PerfLogs/GPU-Z_Sensor_Log.txt`.
        First parameter is the index of temperature column in that file.
        Useful if you have Windows and use GPU-Z to dump GPU/CPU temperatures
        to a file in CSV-like (but with indents and without quotes) format like
        `2023-09-22 23:36:08 ,               54.0   ,               80.3   ,`.

        * `../../templates.d/tempget_cmd.difflog`: Analyze two files specified
        in `cputempfile` and `gputempfile` correspondingly, through an SSH
        session to remote computer (using `ssh_cmd`'s output). `*tempfile` may
        contain wildcards (in that case the first file found not older than 10
        min by modification time is analyzed); if it contains spaces, `(` or
        `)`, these characters must be escaped. E.g, it may look like
        `/c/Program\ Files\ \(x86\)/SpeedFan/SFLog*.csv`. If first parameter
        is 0, `cputempfile` is analyzed, and format is treated as SpeedFan log
        format: number of seconds since midnight, and then temperature
        in format like `12.3` or `45,6`. If first parameter is 1, `gputempfile`
        is analyzed, and its format treated as with `tempget_cmd.csvlog`, with
        the index of temperature column being 0. Useful if you have Windows
        and use SpeedFan to dump CPU temperature, and GPU-Z to dump GPU
        temperature to these two files.

    * `gethdd_cmd`: Check a consistency of remote computer's disk. Should print
    nothing, and return 0 on success and non-null on failure. Could be
    symlinked to:

        * `../../templates.d/gethdd_cmd.sda`: Check remote computer's
        `/dev/sda` by SSH (using `ssh_cmd`'s output and calling
        `get_hdd_state`).

        * `../../templates.d/gethdd_cmd.nvme`: Do the same, but for
        `/dev/nvme0n1`.

        * `../../templates.d/gethdd_cmd.rootfs`: Do the same, but for
        a device containing partition that is mounted as the rootfs.

    * `battery_cmd`: Print a battery level of remote computer. Return value
    does not matter. Should print exactly 6 characters, of which leading and
    trailing ones are spaces, and should not produce a newline after that (so
    actual output should look like what commands `echo -n " 95 % "`,
    `echo -n "  2 % "`, or `echo -n " 100% "` produce. Could be symlinked to:

        * `../../templates.d/battery_cmd.uncolored`: Print remote computer's
        battery level, connecting to it by SSH (using `ssh_cmd`'s output)
        and calling `get_battery`.

        * `../../templates.d/battery_cmd.colored`: Do the same, but colorize
        output: 0..20% is bright red, 21..50% is yellow, 51..100% is cyan.

**Note:** if you wish to place a separator between two lines, name a directory
using empty hostname. E.g., `31-` will place a separator after `30-...`,
but before `32-...`. The contents of directory, which is used as a separator
mark, is ignored by `rumia`, except for chosen marker file (see `temp_enabled`
above), and totally ignored by `execute_cmd`. For example, to appear in generic
run of `rumia`, such separator's directory should contain `temp_enabled`
marker.

### Configuring remote computers

For RUMIA to be able to monitor computer's availability, this computer
should be able to reply to ping (if you're using `ping_cmd.generic` or
`ping_cmd.recursive`), or let SSH clients be connected to it (if you're using
`ping_cmd.by_ssh`.

Other things require execution of some commands in a SSH session.

For RUMIA to be able to monitor uptime of remote computer, it should have
`uptime` command (most POSIX-compliant OS do).

For RUMIA to be able to monitor CPU load of remote computer, it should have
`mpstat` command (in most Debian and Red Hat based OS it is contained within
the `sysstat` package).

For RUMIA to be able to check HDD state, you should copy
`remote_scripts/get_hdd_state` to the remote computer as `get_hdd_state`
into any directory from `$PATH`. Note: if you modified `ssh_cmd` so you
log in as user other than root, you might need to add that user to `sudoers`
file.

For RUMIA to be able to check battery value, you should copy one of the
`remote_scripts/get_battery_*` scripts to the remote computer as `get_battery`
into any directory from `$PATH`. Note: if you modified `ssh_cmd` so you
log in as user other than root, you might need to add that user to `sudoers`
file. If none of these scripts are useful for you, you may write your own based
on supplied scripts as examples. It should read the battery value of your
system and print it as a decimal value from 0 to 100, or print 0 on error.
Output should not contain leading and trailing spaces, but may contain
a trailing newline.

For RUMIA to be able to check CPU temperatures, you should copy one of the
`remote_scripts/get_temperature_*` scripts to the remote computer as
`get_temperature` into any directory from `$PATH`. In most cases, best
choice is `get_temperature_configurable`, edited according to structure of
`hwmon` sysfs facility (note the comments inside this file). Note: if you
modified `ssh_cmd` so you log in as user other than root, you might need to
add that user to `sudoers` file. If none of these scripts are useful for you,
you may write your own based on supplied scripts as examples. It should read
the value of temperature sensor with number specified in `$1` (starting from 0)
and print it as a fixed point value from `0.0` to `99.9`, or print `99.9`
on error. Output should not contain leading and trailing spaces, but may
contain a trailing newline.

## Running a monitoring system

To run monitoring system, just run `rumia`.

You can specify the following parameters:

* `--nofancynames`: Don't use fancy names for computers, use actual hostnames
(as in directory names inside `computers.d`) instead.

* `--oneshot`: Perform only one shot and immediately exit. Don't write anything
into a log file.

* `--noclear`: Don't clear console screen on start, or when USR1 or USR2
is signaled; don't reposition cursor to the top of the screen on each new
shot. Print new shot data just after the previous.

* `--config <configfile>`: Use predefined config file instead of default.
Note: no other config files is read if this parameter is specified. The only
parameter in command line that has a meaning, is the last one.

* `--help`: Show usage and exit.

* `--version`: Show version and exit.

Additionally, you may specify `MACHINES` environment variable to run RUMIA on
a subset of computers defined in `computers.d` which have `temp_enabled`
marker. For example, if you have `01-compA`, `02-compB`, `03-compC`, and
`04-compD` subdirectories there, and first three have `temp_enabled` inside
them, then by default RUMIA would poll compA, compB and compC. But you may want
to run RUMIA like:

```bash
MACHINES="compA compC" ./rumia --oneshot
```

Then it will poll just compA and compB, and nothing else. Specifying
`MACHINES="compA compD"`, by the way, won't let compD to be polled; it is
implied that if a computer doesn't have its own `temp_enabled`, then RUMIA
has no knowledge of how to gather information from it.

It is useful to run `rumia` on some machine inside a `tmux` or `screen`
session, and if you're about to check computers' status, you might connect
to it, attach to session and check whatever you need, then detach and
disconnect, but `tmux` or `screen` session then continues to run.

## Function

Once started, RUMIA clears screen (if not told not to do so) and starts
checking each computer for which directory is exist inside `computers.d`.

Order of polling is determined by names of such directories. For example,
`10-computerX` will be checked before `20-computerY`, and if you create
`15-computerZ` directory, it will be checked in between.

Information gathered is printed on the screen in a fancy table. This phase
(until the last computer is checked) is called *a shot*.

After all information is gathered, RUMIA enters *pause state*, which lasts
for a certain time specified in `rumia.conf`.

Any change of contents of `computers.d` is applied immediately, and will be
used during the very next shot (more precise, to the moment when this computer
is to be polled during the shot). Changes in `rumia.conf` are applied, though,
once `rumia` is restarted.

You may send `SIGINT` to `rumia` process (or just press Ctrl+C, if it's in
foreground) to stop performing its operation and exit.

If you ran it not in one shot mode, you may also send `SIGUSR1` to it while
it's in pause state, and RUMIA will prematurely end the pause state, clear
the screen again, and immediately start next shot. The corresponding hint is
displayed in top of the screen (if not in one shot mode).

You may also send `SIGUSR2`, and this will result in the same, but with
clearing most of config file variables, like `PREFIX`, `COMPUTERS_D`,
`LOGFILE`, `RUMIA_LANG`, `MARKER`, `DELAY`, and `SSH_OPTS`, and re-reading
them from config file. It also means that if you're specified some of these
variables in command line or environment, e.g. running rumia like
`DELAY=60 rumia`, thus overriding values specified in config file, these
overrides will be cancelled.

**Note:** if `USR1` and `USR2` are not signaled, RUMIA won't clear screen once
pause state is finished. Next shot will redraw each line separately, so even
when RUMIA performs a shot, you still have all information still available
on the screen, except the one for a computer currently being polled.

## Executing an arbitrary command

To execute a command you may want to run on several machines that are in your
`computers.d`, there is a `execute_cmd` script available. Generally, you may
run it just in the form like:

```bash
./execute_cmd command
```

Here, `command` is a command that will be executed on every computer defined
inside the `computers.d` directory that has specific marker file (it is named
`ssh_enabled` by default).

You can specify the following RUMIA parameters (before the actual command you
wish to execute):

* `--config <configfile>`;
* `--help`;
* `--version`.

They have the exact same meaning as for RUMIA itself.

Additionally, you may stop parsing parameters by specifying `--` after all
`execute_cmd` parameters. So, `execute_cmd --help` will print help, but
`execute_cmd -- --help` will try to run `--help` binary instead (but it is
still useless in some cases, because bash and some other shells do not
recognize `--help` as binary name). And `execute_cmd -- --` will try to run
`--` binary in the same manner, by the way. Parameter parsing is stopped
on the first unrecognized command line token, so `execute_cmd bash --help`
will execute `bash --help`, and won't treat `--help` as `execute_cmd`
parameter.

Additionally, you may specify `MACHINES` environment variable to run
`execute_cmd` for specific machines only, same as for RUMIA.

**Note:** the command will be executed only on machines that have `ssh_enabled`
marker file in their `computers.d` subdirectories; other machines will be
skipped. You may redefine the name of this file by specifying it in `SMARKER`
variable, like this:

```bash
SMARKER=autofs_enabled ./execute_cmd ls /autofs/mounts
```

This command will list contents of `/autofs/mounts` only on those computers
that have `autofs_enabled` marker file in their `computers.d` subdirectories.

## Trivia

Actually, [Rumia](https://en.touhouwiki.net/wiki/Rumia) is a youkai girl from
Touhou Project series, and she is known for some of her gastronomic addictions
and appetite; just like Rumia from Touhou consumes her victims, RUMIA consumes
(and aggregates) information from computers configured.

Now live with this information. :-)
