# base-alpine 

## Note
Please make sure you use a tagged version of base-alpine, such as:

```Dockerfile
FROM vizzbuzz/base-alpine:0.7
```

## Introduction

This is a simple but powerful base image, based on Alpine Linux with [S6](http://skarnet.org/software/s6/) as a process supervisor and dnsmasq for DNS management, both of which have extremely small footprints adding virtually no runtime overhead and a minimal filesystem overhead.

Why a supervisor process? Firstly because it solves the [PID 1 Zombie Problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/) but most importantly because many containers need to run multiple processes.

Running multiple 'applications' in a single container is of course not The Docker Way (tm) - however running multiple *processes* is often required. [S6](http://skarnet.org/software/s6/) provides a very simple, low resource and elegant processor supervisor which fits in well with the Alpine Linux minimalism.

Also this image supports syslog logging, all syslog messages will be sent to stderr - no more losing syslog logging!

This image aims to be a suitable base image for people who want to deploy to [Tutum](http://tutum.co) - hence why it has a specific dnsmasq service for tutum.io.

## Usage Notes

### Shell

Alpine Linux uses [BusyBox](http://www.busybox.net/) to provide a lot of the core Unix/Linux utilities. As part of that we get the [Ash](http://linux.die.net/man/1/ash) shell, which is very similar to the Bourne (BASH) shell. Just make sure you realise there are differences, it is almost POSIX compliant, so if in doubt use the POSIX complaint syntax rather than BASH extensions.
 
You can of course install bash - and why not?. Doing so will add a few more meg to your *tiny* image.

[![](https://badge.imagelayers.io/vizzbuzz/base-alpine.svg)](https://imagelayers.io/?images=vizzbuzz/base-alpine:latest 'Get your own badge on imagelayers.io')

### S6

[S6](http://skarnet.org/software/s6/) is a supervisor or process management system, similar to using runit or even supervisord in nature.It's a very powerful system so I recommend reading the [docs](http://skarnet.org/software/s6/) - however the quick and dirty way to get started is:

1) Just use CMD as usual in your Dockerfile, the ENTRYPOINT is set to a script that will run the CMD under S6 and shutdown the entire image on CMD failure.

2) Add additional scripts using this format

```Dockerfile
COPY myservice.sh /etc/services.d/myservice/run
RUN chmod 755 /etc/services.d/myservice/run
```

## Good Practises

###Don't Run as Root

During the build run:

```Dockerfile
RUN addgroup -g 999 app && adduser -D  -G app -s /bin/false -u 999 app
```

This creates a non root user for you use. Then in your S6 scripts run your commands using:

```BASH
#!/usr/bin/env sh
exec s6-applyuidgid -u 999 -g 999 mycommand.sh 
```

The `exec` will write over the shell's process space reducing the memory overhead and `s6-applyuidgid -u 999 -g 999` will run it as `app` the non root user.


### Keep it Small

Don't put `RUN` instructions in your `Dockerfile`, instead create a `build.sh` script and run that:

```Dockerfile
COPY build.sh /build.sh
RUN chmod 755 build.sh
RUN /build.sh
```

Of course you can save doing this until it's a last minute optimization when you've got everything running. 

In your `build.sh` file start with:

```BASH
#!/usr/bin/env sh
set -ex
cd /tmp
apk upgrade
apk update
```

And end with

```BASH
apk del <applications that were used only for building, like gcc, make etc.>
rm -rf /tmp/*
rm -rf /var/cache/apk/*
```

This will clean up any mess you created while building. `set -e` causes the script to fail on any single commands failure and `set -x` lists all commands executed to `stderr`

##Credits

Originally taken from https://github.com/just-containers/base-alpine credit to John Regan <john@jrjrtech.com>

