# Autoaxfr -- A wrapper script for axfr-get

The DNS system this was designed for, tinydns, is available at
http://cr.yp.to/.  A useful related site is http://tinydns.org/.

# Overview

This program might be useful to you if you are running tinydns but must
also fetch some zones with the (insecure) zone transfer protocol.

Basically, this program lets you emulate the functionality of bind's "type
slave" zones with a "masters" list of IP addresses.

It is designed to run as a daemon that periodically wakes up to do zone
transfers. It leaves the current zones in a well known directory for you to
incorporate into tinydns' `data.cdb` by hand, with a crontab, or other
means.

autoaxfr is completely passive and does not have a mechanism to notify
another program when the zones it fetches have changed.

To use this program you need djbdns (available at
http://cr.yp.to/djbdns.html), ucspi-tcp (available at
http://cr.yp.to/ucspi-tcp.html) and daemontools (available at
http://cr.yp.to/daemontools.html).

# Installation

Copy the autoaxfr script to `/usr/local/bin`. Modify the first line to
point to where your perl 5.x lives, if it isn't in `/usr/bin`.

Run `autoaxfr-conf` like so:

```sh
autoaxfr-conf acct logacct DIR
```

Where acct is usually a `'autoaxfr'` account you've created just for
`autoaxfr`, `logacct` is the `'dnslog'` account you probably set up for use
with djbdns, and `DIR` is usually `/etc/autoaxfr`.

If you didn't put `autoaxfr` in `/usr/local/bin`, modify `DIR/run` to run
it from the correct location.

If the axfr-get and tcpclient programs aren't in the system's default path,
modify `DIR/run` to add the appropriate directory to the PATH environment
variable.

Then you can put a symlink in /service to DIR and autoaxfr will run
under the svscan just like tinydns.

# Configuration

Say you have this in your bind named.conf:

```
zone "example.com" {
        type slave;
        masters { 192.168.10.10; 192.168.10.11; }
}
```

You get the same effect by doing this:

```sh
echo 192.168.10.10 >  DIR/root/slaves/example.com
echo 192.168.10.11 >> DIR/root/slaves/example.com
```

Autoaxfr will try to do zone transfers for example.com from those two IP
addresses (using the second only if the first fails). When transferred, the
zones are placed in a file of the same name in the `DIR/root/zones/`
directory.

Autoaxfr will honor the refresh and retry times in the zone's SOA and
update the zones with `axfr-get` as appropriate.

If you'd like to axfr a root zone, use `@` as the file name. Autoaxfr will
translate that into the `"."` domain name when calling `axfr-get`.

# What Next?

You have to get the zone data into tinydns somehow.

This is the Makefile I use in `/etc/tinydns/root`. I place my zone data in
`mydata`. The Makefile copies the slave zone data from `DIR/root/zones` and
places it in `axfrdata`. The two are then concatenated into data and
transformed into `data.cdb`. I run this makefile from root's crontab every
5 minutes.

```Makefile
data.cdb: data
        /usr/local/bin/tinydns-data

data: mydata axfrdata
        cat $^ > $@

axfrdata: /service/autoaxfr/root/zones/*
        sort -u $^ > $@
```

# Bugs (or features?)

No attention is paid to slave zone SOA expire time. Autoaxfr will keep
trying to transfer a zone until you remove it from `DIR/root/slaves`. It
will never delete a file from `DIR/root/zones`.

On the same token, if you delete a file in `DIR/root/slaves`, you must
also delete the file in `DIR/root/zones`.

I am assuming that `axfr-get` does auditing of the zones it receives and
doesn't allow garbage for other zones to pollute its output. Be warned that
I haven't actually verified that this is the case!
