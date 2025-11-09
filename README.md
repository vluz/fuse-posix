# Summary of All Changes Made to Fix the FUSE-Rucio Driver

## Problem 1: Broken Scope Listing

### Original Issue: Scopes were displayed as broken fragments like {account:alice and scope:test}
 
Root Cause: The Rucio API's /scopes/ endpoint changed from returning a Python list format to JSON objects with metadata.

### Files Modified:

include/utils.h:164-167
- Added parse_scope_json() function declaration

source/utils.cpp:290-354
- Implemented parse_scope_json() to parse JSON objects and extract scope values
- Handles both {"scope": "value"} and {scope: value} formats

source/REST-API.cpp:228-245
- Modified rucio_list_scopes() to merge all response lines
- Added auto-detection of JSON object vs Python list format
- Calls appropriate parser based on format

## Problem 2: Incorrect DID Construction for Nested Files

### Original Issue: Files inside containers were assigned the wrong scope, causing download failures
 
Root Cause: The code assumed all files in a path use the top-level scope from the path, but Rucio returns each DID with its actual scope.

### Files Modified:

include/utils.h:177-182
- Added did_scope_cache static map
- Added cache_did_scope() function to store scope mappings
- Added get_cached_scope() function to retrieve cached scopes

source/utils.cpp:368-379
- Implemented cache_did_scope() and get_cached_scope() functions

source/utils.cpp:175-189
- Modified get_did() to check cache for actual scope first
- Falls back to path-based scope extraction if not cached

source/REST-API.cpp:287-292
- Updated rucio_list_dids() to cache actual DID scopes
- Fixed cache keys to use did.scope instead of parent scope

source/REST-API.cpp:330-338
- Updated rucio_list_container_dids() to cache actual DID scopes
- Fixed cache keys to use did.scope instead of parent scope

source/REST-API.cpp:349-357
- Updated rucio_is_container() to use cached scope

source/REST-API.cpp:386-394
- Updated rucio_is_file() to use cached scope

source/REST-API.cpp:422-429
- Updated rucio_get_size() to use cached scope

## Problem 3: Downloading Flag Not Cleared on Failure

### Original Issue: Failed downloads left paths marked as "downloading" forever, causing "Operation now in progress" errors
 
Root Cause: The downloading flag was only cleared on successful cache hits, never on download failures.

### Files Modified:

include/rucio-download.h:86
Added fpath field to rucio_download_info struct to store original path

include/rucio-download.h:94
Modified constructor to initialize fpath field

include/download-pipeline.h:72-77
Updated rucio_notifier::process_input() to call set_downloaded() for both success and failure
Clears downloading flag after any download completion

---

All my changes are Public Domain or CC0.

**Please note the original code License is:**

```
Copyright European Organization for Nuclear Research (CERN)
Licensed under the Apache License, Version 2.0 (the "License");
You may not use this file except in compliance with the License.
You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0
Authors:
 - Gabriele Gaetano Fronzé, <gfronze@cern.ch>, 2019-2020
 - Vivek Nigam <viveknigam.nigam3@gmail.com>, 2020
```

---

Original README.md:

# Rucio `FUSE-posix` interface

This repository carries the best attempt at making the Rucio namespace `posix`-mountable.

It is based on `C` code (for the custom `FUSE` specialization) and on `C++11` for utilities and wrappers.

This tool is intended as an **alpha** version unless explicitly said and has to be considered WIP.

Please note that the first target is to get a **read-only file system**.

## Access pattern
The Rucio file catalog is much flatter than that of a usual `POSIX` filesystems and as such its representation has to be structured according to that:

- the root of the mount is intended to be a `cvmfs`-like "`/rucio`";
- the first level of the tree (namely `/rucio/*`) should be filled with all the available scopes;
- each scope should appear as a directory filled with its DiDs;
- file DiDs will appear as files;
- container and dataset DiDs will appear as directories;
- datasets and containers might include already represented DiDs: a routine to handle such loops should be present.

## Getting Started

### Cloning the repository
Fork the repository and clone it on your local machine for development

```BASH
$ git clone https://github.com/<your-username>/fuse-posix.git
$ cd fuse-posix/
$ git remote add upstream https://github.com/rucio/fuse-posix.git
```

### Install the dependencies
Next step is to install the dependencies which shall be required to build the software.
To complete the build `libcurl-devel`, `fuse-libs` and `fuse-devel` packages (or equivalent) must be present:
`cmake` will try to locate them for you and trigger some build messages if unable to do so.
Please note that `cmake` version 3 or greater is needed.

**Ubuntu/Debian (apt)**

```BASH
$ sudo apt-get install cmake git g++ libfuse-dev libcurl4-gnutls-dev
```

**CentOS (yum)**

```BASH
$ yum install -y cmake3 libcurl-devel libfuse-devel
```

### Build the software
To build the software please run:

```[shell]
$ ./build.sh
```

### Setting up the FUSE mount

* **STEP 1** - Check FUSE version

```BASH
$ fusermount -V
fusermount version: 2.9.7
```

Also check if the `fuse` group is available on the system and your current user is a part of it or not

```BASH
$ groups $USER | grep fuse
```

If this highlights `fuse` then your current USER is a part of the `fuse` group and you may skip STEP 2. 
If not, then add the `fuse` group as explained in STEP 2.

* **STEP 2** - Add `fuse` group for current USER

```BASH
$ sudo groupadd fuse
$ sudo usermod -aG fuse $USER
```

The above step adds a new group named `fuse` and makes the current user a part of it. 
If you wish to access the FUSE mount as `root`, then you must also uncomment (or add if not present) the line `user_allow_other` in `/etc/fuse.conf` to enable root access for FUSE filesystem. 
Once this is done successfully, reload the fuse module with `modprobe` or restart the machine.

```
$ modprobe fuse
```

### Setting up Rucio-FUSE connections
Now that `fusermount` is set up on the machine, we need to set up the Rucio-FUSE connections before proceeding with the mounting process.

The Rucio FUSE module will parse the connection parameters from configuration files in the same format as typical `rucio.cfg` files. 
All the `.cfg` files found in the path pointed by the environment variable `RUCIOFS_SETTINGS_FILES_ROOT` will be parsed looking for the following required fields in the `[client]` section:
 - `rucio_host`: the server URL
 - `username`: the server username
 - `password`: the server password
 - `account`: the connection account

At the moment only `userpass` authentication method is supported.
To add a new server it is enough to import the existing `.cfg` and restart the FUSE module.

**NOTE:** The configuration files are passed at runtime to the internal routine performing rucio downloads. Do not remove them without restarting the FUSE module!

### Mounting with Rucio-FUSE mount
The FUSE mount can be either used as `root` or the current user.

* **OPTION A** - Mounting as `root`

To use FUSE as root, you need to set up fuse module as given in STEP 2, then follow the steps below.

```BASH
$ sudo mkdir /ruciofs
$ cd fuse-posix/
$ sudo ./cmake-build-debug/bin/rucio-fuse-main
```

* **OPTION B** - Mounting as current USER

This step requires root to change the ownership of the mount point from `root` and group `root` to `$USER` and group `fuse`

```BASH
$ sudo mkdir /ruciofs
$ sudo chown $USER:fuse /ruciofs
$ ./cmake-build-debug/bin/rucio-fuse-main
```

Performing the above steps successfully shall parse the `settings.json` file and mount the server to `/ruciofs` mount point.

### CLI Options

#### Log levels
The following log levels flags are supported:
* `no-opt`: log level ERROR
* `-v`: log level INFO
* `-vv`: log level DEBUG

### Mount point
The `-f` flag, followed by a valid path, overrides the internally defined `ruciofs` default mount path.

### Config files repository
The `-c` or `--config` flag, followed by a valid path, overrides the directory from which the `rucio.cfg` files are parsed at startup.
It will default to `./rucio-settings` if not set up via CLI.
This is useful for un-the-fly config changes and tests.

## Extra Notes

* This has been tested on CentOS7, ~~Mac OS X Mojave 10.14.6~~, and Ubuntu 18.04 LTS.
* Mac OS X special files created by the OS FS service generate a lot of issues which should be dealt with and prevent the Fuse module from correct mounting under Mac.
