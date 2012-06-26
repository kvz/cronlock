cronlock
========

Use a central redis instance to globally lock cronjobs across a distributed system.
This can be usefull if you have 30 webservers that you deploy crontabs to (such as
mailing your customers), but you don't want 30 cronjobs spawned.

Of course you could also deploy your cronjobs to 1 box, but in volatile environments 
such as EC2 it can be helpful not to rely on 1 'throw away machine' for your scheduled tasks,
and have 1 deploy method for all your workers.

By settings locks, `cronlock` can also prevent overlap in longer-than-expected-running cronjobs.

## Notes

 - Follows locking logic from Redis documentation at http://redis.io/commands/setnx
 - requires a `bash` with `/dev/tcp` enabled. Older Debian/Ubuntu systems disable `/dev/tcp`.
 - requires `md5`

## Install

On most boxes `cronlock` will install just by downloading & setting the right permissions because
`md5` and `/dev/tcp` are widely available.

```bash
wget -O /usr/bin/cronlock --no-check-certificate https://raw.github.com/kvz/cronlock/master/cronlock
chmod 755 /usr/bin/cronlock
```

## Options

 - `CRONLOCK_CONFIG` location of config file. this is optional since all config can also be
 passed as environment variables. default: `/etc/cronlock.conf`, `<DIR>/cronlock.conf`

Using the `CRONLOCK_CONFIG` file or by exporting in your environment, you can set these variables
to change the behavior of `cronlock`:

 - `CRONLOCK_HOST` the redis hostname. default: `localhost`
 - `CRONLOCK_PORT` the redis hostname. default: `6379`
 - `CRONLOCK_GRACE` determines how long a lock should at least persist. default is 40s: `40`.
 This is too make sure that if you have a very small job, and clocks aren't in sync, the same job
 on server2 (slightly behind in time) will just fire right after server1 finished it.
 - `CRONLOCK_RELEASE` determines how long a lock can persist at most. acts as a failsafe so there can be no locks that persist forever in case of failure. default is a day: `86400`
 - `CRONLOCK_KEY` a unique key for this command in the global redis instance. default: a hash of cronlock's arguments
 - `CRONLOCK_PREFIX` redis key prefix used by all keys. default: `cronlock.`
 - `CRONLOCK_VERBOSE` set to `yes` to print debug messages. default: `no`


## Examples

### Test

Just executes `ls al`:

```bash
git clone git://github.com/kvz/cronlock.git
cd cronlock
./cronlock ls -al
```

### Single box

```bash
crontab -e
* * * * * cronlock ls -al
```

In this configuration, `ls -al` will be launched every minute. If the previous
`ls -al` has not finished yet, another one is not started.
This works on 1 server, as the default `CRONLOCK_HOST` of `localhost` is used.

In this setup, `cronlock` works much like Tim Kay's awesome [solo](https://github.com/timkay/solo),
except `cronlock` requires redis, so I recommend using Tim Kay's solution here.

### Distributed

```bash
crontab -e
* * * * * CRONLOCK_HOST=redis.mydomain.com cronlock ls -al
```

In this configuration, a central redis instance is used to track the locking for
`ls -al`. So now you can safely assume that throughout a cluster of 100 servers,
just one instance of `ls -al` is ran every minute. No less, no more.

### Commands with different arguments

By default cronlock uses your command and it's arguments to make a unique identifier
by which the global lock is acquired. However if you want to run: `ls -al` or `ls -a`, 
but just 1 instance of either, you\'ll want to provide your own key:

```bash
crontab -e
# One of two will be executed because they share the same KEY
* * * * * CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_KEY="ls" cronlock ls -a
```

### Per application

One ls will be excecuted for app1, and one for app2. 

```bash
crontab -e
# One of two will be executed because they share the same KEY accross 1 PREFIX
* * * * * CRONLOCK_PREFIX="app1" CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_PREFIX="app1" CRONLOCK_KEY="ls" cronlock ls -a

# One of two will be executed because they share the same KEY accross 1 PREFIX
* * * * * CRONLOCK_PREFIX="app2" CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_PREFIX="app2" CRONLOCK_KEY="ls" cronlock ls -a
```

## Exit codes

 - `200` Success (delete succeeded or lock not acquired, but normal execution)
 - `201` Failure (cronlock error)
 - < `200` Success (acquired lock, executed your command), passes the exit code of your command
