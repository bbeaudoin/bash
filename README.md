# Bash Examples

A repository for bash scripts and functions that I've written for my systems and hardware or general purpose functions and templates for scripting documenting potentially reusable patterns for other work. This was coded from scratch with lots of public reference material, a short description of the files so-far is here:

1. kvmctl - sends packets to a Tesmart KVM using serial bytecode communication
1. tesmart.md - protocol documentation in a more readable format than the vendor provided
1. powerctl - simple tool to power on/off ports on my CyberPower Systems ATS PDU
1. template - A collection of functions and code I wrote for prototyping CLI tools
1. README.md - this file

# Static Variable Definitions

There are a lot of variables I routinely set in scripts that, while useful to have documented in a template, just make the script larger and more difficult to read since they aren't actually in-use. I've moved them here.

In general, these are defined as needed or if I find I set this more than once (like `progname=$(basename $0)`). My own convention for static internal variables is to prefix them with an underscore and use all caps. Whether I'm consistent with my own conventions is another story.

## Program Name
This is, in general, the name of the program without the full path. When I script quickly I use `$0` and later replace those instances with `$(basename $0)` or `local progname=$(basename $0)` eventually defining the variable as a static definition on a global scope.

    _PROGNAME=$(basename $0)

## Directory Name
Used far less frequently than `${_PROGNAME}`, 

    _DIRNAME=$(dirname $0)

## Present Working Directory
The present working directory. The static definition should not be redefined, rather refer to this if PWD needs to be reset.

    _PWD=$(pwd)

## Internal File Separator
The internal file separator. This may be redefined within a function as a local IFS without interfering with the main function but a function called from another function can reset the IFS for the caller. A copy of the IFS can be used to reset the IFS if need be.

    _IFS=${IFS}

## Networking Information
General network information. When this is information is needed, it can be handy to have both a staticly defined (in this case system provided) along with a user-defined version for purpose of comparison.

    _HOSTNAME=$(hostname -f)
    _DOMAIN=$(hostname -d)
    _IPADDRESS=$(hostname -i)
    _INTERFACE=$(ip route show default|awk '{print $5}')
    _CIDRADDRESS=(ip addr show dev ${_INTERFACE}|awk '/inet / {print $2}')

## Release Information
Full release name and version. Situationally useful, often the version itself is more applicable to the use case than the textual description of the release.

    _RELEASE=$(lsb_release -ds) # Returns something like 
    _RELEASE=$(lsb_release -rs) # Returns 8.1 for the same operating system
