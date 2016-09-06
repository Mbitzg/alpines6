# docker image: alpines6 [Layer 2]

This layer 2 docker image is built upon [mbitz/alpinebase](https://hub.docker.com/r/mbitz/alpinebase/)

## Description

Install native [s6](http://skarnet.org/software/s6/overview.html) apk and init scripts from [s6-overlay](https://github.com/just-containers/s6-overlay) as a process supervisor to support multi processes per container applications.


## Additions
Packages: ```s6 s6-portable-utils```

Files and Folders:
```
/:
	init

/etc/:
    s6
    cont-init.d
    ont-finish.d
    fix-attrs.d
    services.d

/usr/bin/:
	fix-attrs
	host-ip
    logutil-newfifo
    logutil-service
    logutil-service-main
    set-contenv
    printcontenv
    with-contenv
	with-retries
```

## Init stages

Init is a properly customized one to run appropriately in containerized environments.

1. **stage 1**: Its purpose is to prepare the image to enter into the second stage. Among other things, it is responsible for preparing the container environment variables, block the startup of the second stage until `s6` is effectively started, ...
2. **stage 2**: This is where most of the end-user provided files are mean to be executed:
  1. Fix ownership and permissions using `/etc/fix-attrs.d`.
  2. Execute initialization scripts contained in `/etc/cont-init.d`.
  3. Copy user services (`/etc/services.d`) to the folder where s6 is running its supervision and signal it so that it can properly start supervising them.
3. **stage 3**: This is the shutdown stage. Its purpose is to clean everything up, stop services and execute finalization scripts contained in `/etc/cont-finish.d`. This is when our init system stops all container processes, first gracefully using `SIGTERM` and then (after `S6_KILL_GRACETIME`) forcibly using `SIGKILL`. And, of course, it reaps all zombies :-).

## Usage

You have a couple of options:

* Run your service/program as your image's `CMD`
* Write a service script

Service Processing Stages:
### Fixing ownership & permissions

Sometimes it's interesting to fix ownership & permissions before proceeding because, for example, you have mounted/mapped a host folder inside your container. S6 provides a way to tackle this issue using files in `/etc/fix-attrs.d`. This is the pattern format followed by fix-attrs files:

```
path recurse account fmode dmode
```
* `path`: File or dir path.
* `recurse`: (Set to `true` or `false`) If a folder is found, recurse through all containing files & folders in it.
* `account`: Target account. It's possible to default to fallback `uid:gid` if the account isn't found. For example, `nobody,32768:32768` would try to use the `nobody` account first, then fallback to `uid 32768` instead.
If, for instance, `daemon` account is `UID=2` and `GID=2`, these are the possible values for `account` field:
  * `daemon:                UID=2     GID=2`
  * `daemon,3:4:            UID=2     GID=2`
  * `2:2,3:4:               UID=2     GID=2`
  * `daemon:11111,3:4:      UID=11111 GID=2`
  * `11111:daemon,3:4:      UID=2     GID=11111`
  * `daemon:daemon,3:4:     UID=2     GID=2`
  * `daemon:unexisting,3:4: UID=2     GID=4`
  * `unexisting:daemon,3:4: UID=3     GID=2`
  * `11111:11111,3:4:       UID=11111 GID=11111`
* `fmode`: Target file mode. For example, `0644`.
* `dmode`: Target dir/folder mode. For example, `0755`.

Here you have some working examples:

`/etc/fix-attrs.d/01-mysql-data-dir`:
```
/var/lib/mysql true mysql 0600 0700
```
`/etc/fix-attrs.d/02-mysql-log-dirs`:
```
/var/log/mysql-error-logs true nobody,32768:32768 0644 2700
/var/log/mysql-general-logs true nobody,32768:32768 0644 2700
/var/log/mysql-slow-query-logs true nobody,32768:32768 0644 2700
```

### Executing initialization And/Or finalization tasks

After fixing attributes (through `/etc/fix-attrs.d/`) and just before starting user provided services up (through `/etc/services.d`) our overlay will execute all the scripts found in `/etc/cont-init.d`, for example:

[`/etc/cont-init.d/02-confd-onetime`](https://github.com/just-containers/nginx-loadbalancer/blob/master/rootfs/etc/cont-init.d/02-confd-onetime):
```
#!/usr/bin/execlineb -P

with-contenv
s6-envuidgid nginx
multisubstitute
{
  import -u -D0 UID
  import -u -D0 GID
  import -u CONFD_PREFIX
  define CONFD_CHECK_CMD "/usr/sbin/nginx -t -c {{ .src }}"
}
confd --onetime --prefix="${CONFD_PREFIX}" --tmpl-uid="${UID}" --tmpl-gid="${GID}" --tmpl-src="/etc/nginx/nginx.conf.tmpl" --tmpl-dest="/etc/nginx/nginx.conf" --tmpl-check-cmd="${CONFD_CHECK_CMD}" etcd
```

### Writing a service script

Creating a supervised service cannot be easier, just create a service directory with the name of your service into `/etc/services.d` and put a `run` file into it, this is the file in which you'll put your long-lived process execution. You're done! If you want to know more about s6 supervision of servicedirs take a look to [`servicedir`](http://skarnet.org/software/s6/servicedir.html) documentation. A simple example would look like this:

`/etc/services.d/myapp/run`:
```
#!/usr/bin/execlineb -P
nginx -g "daemon off;"
```

### Writing an optional finish script

By default, services created in `/etc/services.d` will automatically restart. If a service should bring the container down, you'll need to write a `finish` script that does that. Here's an example finish script:

`/etc/services.d/myapp/finish`:
```
#!/usr/bin/execlineb -S0

s6-svscanctl -t /var/run/s6/services
```

It's possible to do more advanced operations - for example, here's a script from @smebberson that only brings down the service when it crashes:

`/etc/services.d/myapp/finish`:
```
#!/usr/bin/execlineb -S1
if { s6-test ${1} -ne 0 }
if { s6-test ${1} -ne 256 }

s6-svscanctl -t /var/run/s6/services
```

### Logging

Our overlay provides a way to handle logging easily since `s6` already provides logging mechanisms out-of-the-box via [`s6-log`](http://skarnet.org/software/s6/s6-log.html)!. We also provide a helper utility called `logutil-service` to make logging a matter of calling one binary. This helper does the following things:
- read how s6-log should proceed reading the logging script contained in `S6_LOGGING_SCRIPT`
- drop privileges to the `nobody` user (defaulting to `32768:32768` if it doesn't exist)
- clean all the environments variables
- initiate logging by executing s6-log :-)

This example will send all the log lines present in stdin (following the rules described in `S6_LOGGING_SCRIPT`) to `/var/log/myapp`:

`/etc/services.d/myapp/log/run`:
```
#!/bin/sh
exec logutil-service /var/log/myapp
```

If, for instance, you want to use a fifo instead of stdin as an input, write your log services as follows:

`/etc/services.d/myapp/log/run`:
```
#!/bin/sh
exec logutil-service -f /var/run/myfifo /var/log/myapp
```

### Dropping privileges

When it comes to executing a service, no matter it's a service or a logging service, a very good practice is to drop privileges before executing it. `s6` already includes utilities to do exactly these kind of things which has similar functions as `su-exec` does:

In `execline`:

```
#!/usr/bin/execlineb -P
s6-setuidgid daemon
myservice
```

In `sh`:

```
#!/bin/sh
exec s6-setuidgid daemon myservice
```

If you want to know more about these utilities, please take a look to: [`s6-setuidgid`](http://skarnet.org/software/s6/s6-setuidgid.html), [`s6-envuidgid`](http://skarnet.org/software/s6/s6-envuidgid.html) and [`s6-applyuidgid`](http://skarnet.org/software/s6/s6-applyuidgid.html).

### Container environment

If you want your custom script to have container environments available just make use of `with-contenv` helper, which will push all of those into your execution environment, for example:

`/etc/cont-init.d/01-contenv-example`:
```
#!/usr/bin/with-contenv sh
echo $MYENV
```

This script will output whatever the `MYENV` enviroment variable contains.

### Read-Only Root Filesystem

Recent versions of Docker allow running containers with a read-only root filesystem. During init stage 2, the overlay modifies permissions for user-provided files in `cont-init.d`, etc. If the root filesystem is read-only, you can set `S6_READ_ONLY_ROOT=1` to inform stage 2 that it should first copy user-provided files to its work area in `/var/run/s6` before attempting to change permissions.

This of course assumes that at least `/var` is backed by a writeable filesystem with execute privileges. This could be done with a `tmpfs` filesystem as follows:

```
docker run -e S6_READ_ONLY_ROOT=1 --read-only --tmpfs /var:rw,exec [image name]
```

**NOTE**: When using `S6_READ_ONLY_ROOT=1` you should _avoid using symbolic links_ in `fix-attrs.d`, `cont-init.d`, `cont-finish.d`, and `services.d`. Due to limitations of `s6`, symbolic links will be followed when these directories are copied to `/var/run/s6`, resulting in unexpected duplication.

### Customizing `s6` behaviour

It is possible somehow to tweak `s6` behaviour by providing an already predefined set of environment variables to the execution context:

* `S6_KEEP_ENV` (default = 0): if set, then environment is not reset and whole supervision tree sees original set of env vars. It switches `with-contenv` into noop.
* `S6_LOGGING` (default = 0):
  * **`0`**: Outputs everything to stdout/stderr.
  * **`1`**: Uses an internal `catch-all` logger and persists everything on it, it is located in `/var/log/s6-uncaught-logs`. Nothing would be written to stdout/stderr.
* `S6_BEHAVIOUR_IF_STAGE2_FAILS` (default = 0):
  * **`0`**: Continue silently even if any script (`fix-attrs` or `cont-init`) has failed.
  * **`1`**: Continue but warn with an annoying error message.
  * **`2`**: Stop by sending a termination signal to the supervision tree.
* `S6_KILL_FINISH_MAXTIME` (default = 5000): The maximum time (in milliseconds) a script in `/etc/cont-finish.d` could take before sending a `KILL` signal to it. Take into account that this parameter will be used per each script execution, it's not a max time for the whole set of scripts.
* `S6_KILL_GRACETIME` (default = 3000): How long (in milliseconds) `s6` should wait to reap zombies before sending a `KILL` signal.
* `S6_LOGGING_SCRIPT` (default = "n20 s1000000 T"): This env decides what to log and how, by default every line will prepend with ISO8601, rotated when the current logging file reaches 1mb and archived, at most, with 20 files.
* `S6_CMD_ARG0` (default = not set): Value of this env var will be prepended to any `CMD` args passed by docker. Use it if you are migrting an existing image to a s6-overlay and want to make it a drop-in replacement, then setting this variable to a value of previously used ENTRYPOINT will improve compatibility with the way image is used.
* `S6_FIX_ATTRS_HIDDEN` (default = 0): Controls how `fix-attrs.d` scripts process files and directories.
  * **`0`**: Hidden files and directories are excluded.
  * **`1`**: All files and directories are processed.
* `S6_CMD_WAIT_FOR_SERVICES` (default = 0): In order to proceed executing CMD overlay will wait until services are up. Be aware that up doesn't mean ready. Depending if `notification-fd` was found inside the servicedir overlay will use `s6-svwait -U` or `s6-svwait -u` as the waiting statement.
* `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` (default = 5000): The maximum time (in milliseconds) the services could take to bring up before proceding to CMD executing.
* `S6_READ_ONLY_ROOT` (default = 0): When running in a container whose root filesystem is read-only, set this env to **1** to inform init stage 2 that it should copy user-provided initialization scripts from `/etc` to `/var/run/s6/etc` before it attempts to change permissions, etc. See [Read-Only Root Filesystem](#read-only-root-filesystem) for more information.


## Caveats

* For now, `s6` doesn't support running it with a user different from `root`, so consequently Dockerfile `USER` directive is not supported (except `USER root` of course ;P). Please use `su-exec` or `s6-setuidgid` to drop privileges from root to running user.

## Tags

* `latest` tracks the `edge` tag from [mbitz/alpinebase](https://hub.docker.com/r/mbitz/alpinebase/)

* `e340` tracks the `e340` tag from [mbitz/alpinebase](https://hub.docker.com/r/mbitz/alpinebase/)

# License
[Apache 2.0](https://www.tldrlegal.com/l/apache2)

# Credits

Author of s6-overlay init scripts: [Gorka Lerchundi Osa](https://github.com/just-containers/s6-overlay)
