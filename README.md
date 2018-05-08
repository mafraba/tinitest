# Why tini

This is a series of tests to experiment what happens with signal handling on docker containers.

Using a simple sleep command, we'll see how different ways of running it, and using `tini` or not, affect the ability of the container to respond to signals (just Ctrl^C in our tests).

## Without tini

Run `docker build -t test/signals:notini -f Dockerfile.notini .` to build the docker image without tini.

### Using exec

- Run `docker run test/signals:notini /sleep15exec.sh`
- Hit `ctrl+c` while it's running

The container does NOT respond to signals. Why? The `sleep` process is run as PID 1, but I guess it probably doesn't setup proper signal handlers. See final note in https://docs.docker.com/engine/reference/run/#foreground:

> Note: A process running as PID 1 inside a container is treated specially by Linux: it ignores any signal with the default action. So, the process will not terminate on SIGINT or SIGTERM unless it is coded to do so

Anyway, if instead of `sleep` we used a process which properly captures signals this would work (e.g. try `ruby -e 'sleep 15'`). But still: ___not every process does___.

### Without exec

- Run `docker run test/signals:notini /sleep15noexec.sh`
- Hit `ctrl+c` while it's running

Process does NOT respond to signals. Why?

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.3  0.1  20048  2808 ?        Ss   09:02   0:00 /bin/bash /sleep15noexec.sh
root         7  2.1  0.0   4236   684 ?        S    09:02   0:00 sleep 15
```

The PID 1 is bash, which does not forward signals anywhere (unless we code it ourselves in the bash script).


## With tini

Run `docker build -t test/signals:tini -f Dockerfile.tini .` to build the docker image with tini.

### Using exec

- Run `docker run test/signals:tini /sleep15exec.sh`
- Hit `ctrl+c` while it's running

The container DOES respond to signals. Why? The `tini` process is run as PID 1, and it does forward the signals to its child process.

### Without exec

- Run `docker run test/signals:tini /sleep15noexec.sh`
- Hit `ctrl+c` while it's running

The container does NOT respond to signals. Why?

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.2  0.0   4228   708 ?        Ss   09:19   0:00 /tini -v -- /sleep15noexec.sh
root         7  0.0  0.1  20052  2780 ?        S    09:19   0:00 /bin/bash /sleep15noexec.sh
root         8  0.0  0.0   4236   660 ?        S    09:19   0:00 sleep 15
```

The PID 1 is `tini`, which does forward signals to its child, but this time its child is `bash`, which in turn ignores the signals.



## With "tini -g"

Run `docker build -t test/signals:tini-g -f Dockerfile.tini-g .` to build the docker image with tini using `-g` flag.

### Using exec

- Run `docker run test/signals:tini-g /sleep15exec.sh`
- Hit `ctrl+c` while it's running

The container DOES respond to signals, same as before. The `-g` option doesn't play here.

### Without exec

- Run `docker run test/signals:tini-g /sleep15noexec.sh`
- Hit `ctrl+c` while it's running

The container DOES respond to signals.

Why? The PID 1 is `tini -g`, which does forward signals not only to its child but to all process in the child process group: https://github.com/krallin/tini#process-group-killing.

# Summary

Can handle signals?

|               | No tini | tini    | tini -g |
|---            |---      |---      |---      |
| with exec     |  No (*) |   Yes   |   Yes   |
| without exec  |  No     |   No    |   Yes   |   

(*) would depend on the process run
