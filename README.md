# cronlock

<!-- badges/ -->
[![Build Status](https://secure.travis-ci.org/kvz/cronlock.svg?branch=master)](http://travis-ci.org/kvz/cronlock "Check this project's build status on TravisCI")
[![Gittip donate button](http://img.shields.io/gittip/kvz.svg)](https://www.gittip.com/kvz/ "Sponsor the development of cronlock via Gittip")
[![Flattr donate button](http://img.shields.io/flattr/donate.png?color=yellow)](https://flattr.com/submit/auto?user_id=kvz&url=https://github.com/kvz/cronlock&title=cronlock&language=&tags=github&category=software "Sponsor the development of cronlock via Flattr")
[![PayPayl donate button](http://img.shields.io/paypal/donate.png?color=yellow)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=kevin%40vanzonneveld%2enet&lc=NL&item_name=Open%20source%20donation%20to%20Kevin%20van%20Zonneveld&currency_code=USD&bn=PP-DonationsBF%3abtn_donate_SM%2egif%3aNonHosted "Sponsor the development of cronlock via Paypal")
[![BitCoin donate button](http://img.shields.io/bitcoin/donate.png?color=yellow)](https://coinbase.com/checkouts/19BtCjLCboRgTAXiaEvnvkdoRyjd843Dg2 "Sponsor the development of cronlock via BitCoin")
<!-- /badges -->

## Install

On most Linux & BSD machines, cronlock will install just by downloading it & making it executable.
Here's the one-liner:

```bash
sudo curl -q -L https://raw.github.com/kvz/cronlock/master/cronlock -o /usr/bin/cronlock && sudo chmod +x $_
```

With [Redis](http://redis.io/) present on `localhost`, cronlock should now already work in basic form.
Let's test by letting it execute a simple `pwd`:

```bash
CRONLOCK_HOST=localhost cronlock pwd
```

If this returns the current directory we're good to go. More examples below.

## Introduction

Uses a central [Redis](http://redis.io/) server to globally lock cronjobs across a distributed system.
This can be usefull if you have 30 webservers that you deploy crontabs to (such as
mailing your customers), but you don't want 30 cronjobs spawned.

Of course you could also deploy your cronjobs to 1 box, but in volatile environments
such as EC2 it can be helpful not to rely on 1 'throw away machine' for your scheduled tasks,
and have 1 deploy-script for all your workers.

Another common problem that cronlock will solve is overlap by a single server/cronjob.
It happens a lot that developers underestimate how long a job will run.
This can happen because the job waits on something, acts different under high load/volume, or enters an endless loop.

In these cases you don't want the job to be fired again at the next cron-interval, making your problem twice as bad,
some intervals later, there's a huge `ps auxf` with overlapping cronjobs, high server load, and eventually a crash.

By settings locks, cronlock can also prevent the overlap in longer-than-expected-running cronjobs.

## Design goals

 - Lightweight
 - As little dependencies as possible / No setup
 - Follows locking logic from [this Redis documentation](http://redis.io/commands/setnx)
 - Well tested & documented

## Requirements

 - Bash with `/dev/tcp` enabled. Older Debian/Ubuntu systems disable `/dev/tcp`
 - `md5` or `md5sum`
 - A [Redis](http://redis.io/) server that is accessible by all cronlock machines

## Options

 - `CRONLOCK_CONFIG` location of config file. this is optional since all config can also be
 passed as environment variables. default: `<DIR>/cronlock.conf`, `/etc/cronlock.conf`

Using the `CRONLOCK_CONFIG` file or by exporting in your environment, you can set these variables
to change the behavior of cronlock:

 - `CRONLOCK_HOST` the Redis hostname. default: `localhost`
 - `CRONLOCK_PORT` the Redis port. default: `6379`
 - `CRONLOCK_READ_TIMEOUT` the length of time we wait for a response from redis before we consider it in an errored state.
 This ensures that if the redis connection goes away that we don't wait forever waiting for a response. default: `30`
 - `CRONLOCK_GRACE` determines how many seconds a lock should at least persist.
 This is to make sure that if you have a very small job, and clocks aren't in sync, the same job
 on server2/3/4/5/6/etc (maybe even slightly behind in time) will just fire right after server1 releases the lock. default: `40` (I recommend using a grace of at least 30s)
 - `CRONLOCK_RELEASE` determines how long a lock can persist at most.
 Acts as a failsafe so there can be no locks that persist forever in case of failure. default is a day: `86400`
 - `CRONLOCK_RECONNECT_ATTEMPTS` the number of times we try to reconnect before erroring.
  If the redis connection is closed, we will attempt to reconnect to redis upto this amount of times. default: `5`
 - `CRONLOCK_RECONNECT_BACKOFF` the lenght of time to increase the wait between reconnects.
  Acts as a failsafe to allow redis to be started before we try to reconnect. Set to 0 to retry the connection immediately. default: `5`
 - `CRONLOCK_KEY` a unique key for this command in the global Redis server. default: a hash of cronlock's arguments
 - `CRONLOCK_PREFIX` Redis key prefix used by all keys. default: `cronlock`
 - `CRONLOCK_VERBOSE` set to `yes` to print debug messages. default: `no`
 - `CRONLOCK_NTPDATE` set to `yes` update the server's clock againt `pool.ntp.org` before execution. default: `no`
 - `CRONLOCK_TIMEOUT` how long the command can run before it gets issues a `kill -9`. default: `0`; no timeout

## Examples

### Single box

```bash
crontab -e
* * * * * cronlock ls -al
```

In this configuration, `ls -al` will be launched every minute. If the previous
`ls -al` has not finished yet, another one is not started.
This works on 1 server, as the default `CRONLOCK_HOST` of `localhost` is used.

In this setup, cronlock works much like [Tim Kay](http://timkay.com/)'s [solo](https://github.com/timkay/solo),
except cronlock requires [Redis](http://redis.io/), so I recommend using Tim Kay's solution here.

### Distributed

```bash
echo '0 8 * * * CRONLOCK_HOST=redis.mydomain.com cronlock /var/www/mail_customers.sh' | crontab
```

In this configuration, a central Redis server is used to track the locking for
`/var/www/mail_customers.sh`. So you see that throughout a cluster of 100 servers,
just one instance of `/var/www/mail_customers.sh` is ran every morning. No less, no more.

As long as your Redis server and at least 1 volatile worker is alive, this happens.

### Distributed using a config file

To avoid messy crontabs, you can use a config file for shared config instead.
Unless `CRONLOCK_CONFIG` is set, cronlock will look in `./cronlock.conf`, then
in `/etc/cronlock.conf`.

Example:
```bash
cat << EOF > /etc/cronlock.conf
CRONLOCK_HOST="redis.mydomain.com"
CRONLOCK_GRACE=50
CRONLOCK_PREFIX="mycompany.cronlocks."
CRONLOCK_NTPDATE="yes"
EOF

crontab -e
* * * * * cronlock /var/www/mail_customers.sh # will use config from /etc/cronlock.conf
```

### Lock commands even though they have different arguments

By default cronlock uses your command and it's arguments to make a unique identifier
by which the global lock is acquired. However if you want to run: `ls -al` or `ls -a`,
but just 1 instance of either, you'll want to provide your own key:

```bash
crontab -e
# One of two will be executed because they share the same KEY
* * * * * CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_KEY="ls" cronlock ls -a
```

### Per application

If you use the same script and Redis server for multiple applications, an unwanted lock could deny app2 it's script.
You could make up your own unique `CRONLOCK_KEY` to circumvent, but it's probably
better to use the `CRONLOCK_PREFIX` for that:

```bash
crontab -e
* * * * * CRONLOCK_PREFIX="mylocks.app1." cronlock /var/www/mail_customers.sh
```

```bash
crontab -e
* * * * * CRONLOCK_PREFIX="mylocks.app2." cronlock /var/www/mail_customers.sh
```

Now both /var/www/mail_customers.sh will run, because they have a different application in their prefixes.

## Exit codes

 - = `200` Success (delete succeeded or lock not acquired, but normal execution)
 - = `201` Failure (cronlock error)
 - = `202` Failure (cronlock timeout)
 - < `200` Success (acquired lock, executed your command), passes the exit code of your command

## Versioning

This project implements the Semantic Versioning guidelines.

Releases will be numbered with the following format:

`<major>.<minor>.<patch>`

And constructed with the following guidelines:

* Breaking backward compatibility bumps the major (and resets the minor and patch)
* New additions without breaking backward compatibility bumps the minor (and resets the patch)
* Bug fixes and misc changes bumps the patch


For more information on SemVer, please visit [http://semver.org](http://semver.org).

## License

Copyright (c) 2013 Kevin van Zonneveld, [http://kvz.io](http://kvz.io)  
Licensed under MIT: [http://kvz.io/licenses/LICENSE-MIT](http://kvz.io/licenses/LICENSE-MIT)
