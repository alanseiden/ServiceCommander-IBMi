# Service Commander for IBM i
A utility for unifying the daunting task of managing various services and applications running on IBM i. Its objective is to provide an intuitive, easy-to-use command line interface for managing services or jobs.

This tool can be used to manage a number of services, for instance:
- IBM i host server jobs
- IBM i standard TCP servers (*TCP, *SSHD, etc.)
- Programs you wrote using open source technology (Node.js, Python, PHP, etc.)
- Apache Tomcat instances
- Apache Camel routes
- Kafka, Zookeeper, ActiveMQ servers, etc
- Jenkins
- The Cron daemon
- OSS Database servers (PostgreSQL, MariaDB)

# Current features
Some of the features of the tool include:
- The ability to specify dependencies (for instance, if one application or service dependds on another), and it will start any dependencies as needed
- The ability to submit jobs to batch easily, even with custom batch settings (use your own job description or submit as another user, for instance)
- The ability to check the "liveliness" of your service by either port status or job name
- Customize the runtime environment variables of your job
- Define custom groups for your services, and perform operations on those groups (by default, a group of "all" is defined)
- Query basic performance attributes of the services
- Assistance in providing/managing log files. This is a best-guess only and naively assumes the service uses stdout/stderr as its logging mechanism. Service Commander has its own primitive logging system that works well only for certain types of services
- Ability to define manage ad hoc services specified on the command line

# Hands-on Exercise
Want to walk through a quick exercise to get some basic "hands-on" experience with this tool? If so, please see [our very simple hands-on exercise](quickstart/HANDS_ON.md)

# Have feedback or want to contribute?
Feel free to [open an issue](https://github.com/ThePrez/ServiceCommander-IBMi/issues/new/choose) with any questions, problems, or other comments. If you'd like to contribute to the project, see [CONTRIBUTING.md](https://github.com/ThePrez/ServiceCommander-IBMi/blob/main/CONTRIBUTING.md) for more information on how to get started. 

In any event, we're glad to have you aboard in any capacity, whether as a user, spectator, or contributor!

# Important differences from other service management tools
Service Commander's design is fundamentally different from other tools that accomplish similar tasks, like init.d, supervisord, and so on. Namely, the functions within Service Commander are intended to work regardless of:
- Who else may start or stop the service
- What other tools may be used to start or stop the service. For instance, Service Commander may start/stop an IBM i host server, but so could the `STRHOSTSVR`/`ENDHOSTSVR` CL commands.
- Whether the service runs in the initially spawned job or a secondary job

Also, this tool doesn't have the privilege of being the unified, integrated solution with the operating system that other tools may have. Therefore, Service Commander cannot take the liberty of assuming that it can keep track of the resources tied to the services that it manages. So, for example, this tool does not keep track of process IDs of launched processes. Similarly, it doesn't have special access to kernel data structures, etc. 

Instead, this tool makes strong assumptions based on checks for a particular job name or port usage (see `check_alive_criteria` in the file format documentation). A known limitation, therefore, is that Service Commander may mistake another job for a configured service based on one of these attributes. For example, if you configure a service that is supposed to be listening on port 80, Service Commander will assume that any job listening on port 80 is indeed that service.

Service Commander's unique design is intended to offer a great deal of flexibility and ease of management through the use of simple `.yaml` files.

# Installation

## System Requirements

For most of the features of this tool, the following is required to be installed (the `make install` of the installation steps should handle these for you):
- db2util (`yum install db2util`)
- OpenJDK (`yum install openjdk-11`)
- bash (`yum install bash`)
- GNU coreutils (`yum install coreutils-gnu`)

The performance information support (`perfinfo`) has additional requirements, including:
- Python 3 with the ibm_db database connector (`yum install python3-ibm_db`)
- Required operating system support, which depends on your IBM i operating system level, as follows:
    - IBM i 7.4: included with base OS
    - IBM i 7.3: Group PTF SF99703 Level 11
    - IBM i 7.2: Group PTF SF99702 Level 23
    - IBM i 7.1 (and earlier): not supported


## Option 1: Binary distribution
You can install the binary distribution by copying the link to the `.rpm` file from the releases page of this project and using `yum` to install it. For instance, to install the v0.3.3 release:
```
yum install https://github.com/ThePrez/ServiceCommander-IBMi/releases/download/v0.4.0/sc-0.4.0-0.ibmi7.2.ppc64.rpm
```
If you cannot access github directly from your IBM i system, simply download the RPM from the ["Releases" page](https://github.com/ThePrez/ServiceCommander-IBMi/releases), upload the file to your IBM i system, and install via `yum`, like so:
```
yum install ./sc-0.4.0-0.ibmi7.2.ppc64.rpm
```

## Option 2: Build from source
To ensure you're getting the latest and greatest code, feel free to build from source. This process assumes your `PATH` environment variable is set up properly, otherwise:
```
PATH=/QOpenSys/pkgs/bin:$PATH
export PATH
```
The build itself can be done with the following steps:
```
yum install git ca-certificates-mozilla make-gnu
git clone https://github.com/ThePrez/ServiceCommander-IBMi/
cd ServiceCommander-IBMi
make install
```

# Basic usage

Usage of the command is summarized as:
```text
Usage: sc  [options] <operation> <service(s)>

    Valid options include:
        -v: verbose mode
        --disable-colors: disable colored output
        --splf: send output to *SPLF when submitting jobs to batch (instead of log)
        --sampletime=x.x: sampling time(s) when gathering performance info (default is 1)
        --ignore-globals: ignore globally-configured services

    Valid operations include:
        start: start the service (and any dependencies)
        stop: stop the service (and dependent services)
        restart: restart the service
        check: check status of the service
        info: print configuration info about the service
        jobinfo: print which jobs the service is running in
        perfinfo: print basic performance info about the service
        loginfo: get log file info for the service (best guess only)
        list: print service short name and friendly name
        
    Valid formats of the <service(s)> specifier include:
        - the short name of a configured service
        - A special value of "all" to represent all configured services (same as "group:all")
        - A group identifier (e.g. "group:groupname")
        - An ad hoc service specification by port (for instance, "port:8080")
        - An ad hoc service specification by job name (for instance, "job:ZOOKEEPER")
        - An ad hoc service specification by subsystem and job name (for instance, "job:QHTTPSVR/ADMIN2")
```
The above usage assumes the program is installed with the above installation steps and is therefore
launched with the `sc` script. Otherwise, if you've hand-built with maven (`mvn compile`), 
you can specify arguments in `exec.args` (for instance, `mvn exec:java -Dexec.args='start kafka'`).


**Specifying options in environment variables**
If you would like to set some of the tool's options via environment variable, you may do so with one of the following:
- `SC_TCPSVR_OPTIONS`, which will be processed when invoked via the `STRTCPSVR`/`ENDTCPSVR` commands
- `SC_OPTIONS`, which will be processed on all invocations
For example, to gather verbose output when using `STRTCPSVR`, run the following before your `STRTCPSVR` command:
```
ADDENVVAR ENVVAR(SC_OPTIONS) VALUE('-v') REPLACE(*YES)
```

## Usage examples
Start the service named `kafka`:
```
sc start kafka
```
Stop the service named `zookeeper`:
```
sc stop zookeeper
```
Check status of all configured services (all services belong to a special group named "all")
```
sc check group:all
```
Try to start all configured services
```
sc start group:all
```
Print information about all configured services
```
sc info group:all
```
Try to start all services in "host_servers" group
```
sc start group:host_servers
```
List all services
```
sc list group:all
```
List jobs running on port 8080
```
sc jobinfo port:8080
```
Stop jobs running on port 8080
```
sc stop port:8080
```
Check if anything is running on port 8080
```
sc check port:8080
```

# Configuring Services

## Initializing your configuration with defaults
If you'd like to start with pre-made configurations for common services, simply run:
```
sc_install_defaults
```
This will install service definitions for:
- IBM i TCP servers
- IBM i Host Servers
- The Cron daemon (if you have cron installed)
- MariaDB (if you have mariadb installed)

## Through YAML configuration files
This tool allows you to define any services of interest in `.yaml` files. These files can be stored in any of the following locations:
- A global directory (/QOpenSys/etc/sc/services). This, of coures, requires you to have admin access (`*ALLOBJ` special authority).
- A user-specific directory($HOME/.sc/services)
- If defined, whatever the value of the `services.dir` system property is. 
The file name must be in the format of `service_name.yaml` (or `service_name.yml`), where "service_name" is the "simple name" of the service as to be used with this tool's CLI. The service name must consist of only lowercase letters, numbers, hyphens, and underscores.

### YAML File Format
See the [samples](https://github.com/ThePrez/ServiceCommander-IBMi/tree/main/samples) directory for some sample service definitions. 
The following attributes may be specified in the service definition (`.yaml`) file:

**Required fields**

- `start_cmd`: the command used to start the service
- `check_alive`: the technique used to check whether the service is alive or not. This is either "jobname" or "port".
- `check_alive_criteria`: The criteria used when checking whether the service is alive or not. If `check_alive` is set to "port", this is expected to be a port number. If `check_alive` is set to "jobname", this is expect to be be a job name, either in the format "jobname" or "subsystem/jobname".

**Optional fields that are often needed/wanted**

- `name`: A "friendly" name of the service
- `dir`: The working directory in which to run the startup/shutdown commands

**Other optional fields**

- `stop_cmd`: The service shutdown command. If unspecified, the service will be located by port number or job name.
- `startup_wait_time`: The wait time, in seconds, to wait for the service to start up (the default is 60 seconds if unspecified)
- `stop_wait_time`: The wait time, in seconds, to wait for the service to stop (the default is 45 seconds if unspecified)
- `batch_mode`: Whether or not to submit the service to batch
- `sbmjob_jobname`: If submitting to batch, the custom job name to be used for the batch job
- `sbmjob_opts`: If submitting to batch, custom options for the SBMJOB command (for instance, a custom JOBD) 
- `environment_is_inheriting_vars`: Whether the service inherits environment variables from the current environment (default is true)
- `environment_vars`: Custom environment variables to be set when launching the service. Specify as an array of strings in `"KEY=VALUE"` format
- `service_dependencies`: An array of services that this service depends on. This is the simple name of the service (for instance, if the dependency is defined as "myservice", then it is expected to be defined in a file named `myservice.yaml`), not the "friendly" name of the service.
- `groups`: Custom groups that this service belongs to. Groups can be used to start and stop sets of services in a single operation. Specify as an array of strings.


## Using the 'scinit' tool
You can use the `scinit` tool can be used to create the YAML configuration files for you. Basic usage of the tool is simply:
```
scinit <program start command>
```
The idea is that you would simply:
1. `cd` to the directory where you'd normally start the service
2. Run the command you'd normally use to start the service, prefixed by `scinit`
3. Answer a series of questions about how you would like the service deployed
In doing so, the `scinit` will create the YAML configuration file for you and also show you information about the newly-configured service.

For instance, if you would normally launch a Node.js application from `/home/MYUSR/mydir` by running `node app.js`, you would run:
```
cd /home/MYUSR/mydir
scinit <program start command>
```
The `scinit` tool will ask you for a "short name" among other things. When done, a service configuration will be saved under that short
name. So, for instance, if your short name is "my_node_app", you can run `sc start my_node_app`.

## Ad hoc service definition
Ad hoc services can be specified on the sc command line in the format `job:jobname` or `port:portname`. 
In these instances, the operations will be performed on the specified jobs. This is determined by looking for
jobs matching the given job name or listening on the given port. The job name can be specified either in
`jobname` or `subsystem/jobname` format.

If an existing service definition is found (configured via YAML, as in the preceding section) that matches the
job name or port criteria, that service will be used. For instance, if you have a service configured to run on
port 80, then specifying `sc info port:80` will show information about the service configured to run on port 80.

Ad hoc service definition is useful for quick checks without the need to create a YAML definition. It's also
useful if you do not recall the service name, but remember the job name or port. 

It is also useful for cases where you just want to find out who (if anyone) is using a certain port. For instance,
`sc jobinfo port:8080` will show you which job is listening on port 8080. Similarly, `sc stop port:8080` will kill
whatever job is running on port 8080.

# Demo (video)
[![asciicast](https://asciinema.org/a/398648.svg)](https://asciinema.org/a/398648)

# Automatically restarting a service if it fails
Currently, this tool doees not have built-in monitoring and restart capabilities. This may be a future enhancement. In the meantime, one can use simple scripting to accomplish a similar task. For instance, to check every 40 seconds and ensure that the `navigator` service is running, you could submit a job like this (replace the sleep time, service name, and submitted job name to match your use case):
```
SBMJOB CMD(CALL PGM(QP2SHELL2) PARM('/QOpenSys/usr/bin/sh' '-c' 'while :; do sleep 40 && /QOpenSys/pkgs/bin/sc start navigator >/dev/null 2>&1 ; done')) JOB(NAVMON) JOBD(*USRPRF) JOBQ(QUSRNOMAX)                         
```
This will result in several jobs that continuously check on the service and attempt to start it if the service is dead. If you wish to stop this behavior, simply kill the jobs. In the above example, the job name is `NAVMON`, so the WRKACTJOB command to do this interactively looks like:
```
 WRKACTJOB JOB(NAVMON) 
```

# Testimonials
> "I use this a lot for my own personal use. Might be useless for the rest of the world. I don't know, though."
>
> &nbsp; --[@ThePrez](https://github.com/ThePrez/), creator of Service Commander

# STRTCPSVR Integration

Service Commander now has integration with system STRTCPSVR and ENDTCPSVR commands. This feature is experimental and may be removed
if too problematic.

To integrate with the STRTCPSVR and ENDTCPSVR commands, you can run the following command as an admin user:
```
/QOpenSys/pkgs/lib/sc/tcpsvr/install_sc_tcpsvr
```
This will install create the `SCOMMANDER` library and compile/install the TCP program into that library. To use a different
library, just set the `SCTARGET` variable. For instance:
```
SCTARGET=mylib /QOpenSys/pkgs/lib/sc/tcpsvr/install_sc_tcpsvr
```
After doing so, you can run the `*SC` TCP server commands, specifying the simple name of the sc-managed service as the instance name. For example:
```
STRTCPSVR SERVER(*SC) INSTANCE('kafka')
```
**Important Notes about AUTOSTART(*YES)**

You can set the `*SC` server to autostart via `CHGTCPSVR SVRSPCVAL(*SC) AUTOSTART(*YES)`. However, great care must be taken in order for this to work properly and not create a security exposure. When STRTCPSVR runs at IPL time, the task will run under the QTCP user profile. This user profile does not have `*ALLOBJ` authority, nor does it have authority to submit jobs as other user profiles. Thus, in order for the autostart job to function properly, the QTCP user profile must have access to run the commands needed to start the service, and the service must not submit jobs to batch as a specific user. Be are that adding QTCP to new group profiles or granting special authorities may represent a security exposure. Also, due to the highly-flexible nature of this tool, it is not good practice to run this command as an elevated user in an unattended fashion. 
In summary, it is likely not a good idea to use `AUTOSTART(*YES)`.


**Special groups used by STRTCPSVR/ENDTCPSVR**
There are a couple special groups used by the TCP server support. You can define your services to be members of one or more of these groups:
- `default`, which is what's started or ended if no instance is specified (i.e. `STRTCPSVR SERVER(*SC)`)
- `autostart`, which is what's started when invoked on the `*AUTOSTART` instance (i.e. `STRTCPSVR SERVER(*SC) INSTANCE(*AUTOSTART)`)
