# acmetool-dns
`acmetool` dns hook script using `knsupdate` or `nsupdate` (DNS UPDATE)

A deBASHed rewrite of the [acmetool](https://github.com/hlandau/acmetool/)
"official" [dns.hook](https://github.com/hlandau/acmetool/blob/master/_doc/contrib/dns.hook)
script to use either [knsupdate](https://www.knot-dns.cz/docs/3.5/html/man_knsupdate.html)
or [nsupdate](https://linux.die.net/man/8/nsupdate) (DNS UPDATE).

Script is organized into several sections to make it easier to audit:

- 1st section sets up all the script parameters
- 2nd section sets up all the config `env` vars
- 3rd get the SOA for the challenge hostname or bail
- 4th do the update if the SOA exists on the primary Name Server
- 5th verify the update was pushed to secondary Name Servers

### 1st: get parameters

`dns` challenge script now takes `exit 42` sooner than later if not a DNS event.

### 2nd: get config

`dns` challenge script now requires `NSERVERS` is set to at least one
PRIMARY Name Server. More than one Name Servers can be set to accommodate
multi-master or PRIMARY-PRIMARY Name Server setups where the DNS sync is
configure out-of-band. e.g. `rsync`

Presuming that the `TSIG_FILE` or `TSIG_KEY` is valid only on the primary
Name Server(s) under your control, this script is careful to attempt DNS
update only on the Name Servers listed in `NSERVERS` to avoid inadvertently
failing to authenticate with other DNS servers which you do _not_ control.

`dns` challenge script will use the `DNS UPDATE' utility set to the full
path of the executable in `NSUPDATE`. Otherwise the script will try to find
[knsupdate](https://www.knot-dns.cz/docs/3.5/html/man_knsupdate.html) or
fall back to using [nsupdate](https://linux.die.net/man/8/nsupdate).

### 3rd: `getSOA`

`get_apex` rewrittern as `getSOA` using a `while` loop instead of recursion.

`test` conditions rewritten to help readability.
IMHO this is easier to comprehend at a glance:
```sh
if [ "$(${DIG} @${NS} ${NAME} SOA +noall +answer | awk '{print $4}')" == SOA ]; then
```
than [this](https://github.com/hlandau/acmetool/blob/master/_doc/contrib/dns.hook#L40):
```sh
local ans="`dig +noall +answer SOA "${name}."`"
if [ "`echo "$ans" | grep SOA | wc -l`" == "1" -a "`echo "$ans" | grep CNAME | wc -l`" == "0" ]; then
```
not to mention that testing for SOA _and_ CNAME will never reach the CNAME
part of the test. It's impossible for a DNS record to be both type SOA and
type CNAME at the same time.

### 4th: `update`

`update` rewritten to use the new optional `TSIG_FILE` setting instead of
`TSIG_KEY` or nither if authenticating updates by source IP.

the `nsupdate` code is piped from an easy to read `sh` HERE document
instead of a sereas of `echo` statments. The syntax complies with both
[nsupdate](https://linux.die.net/man/8/nsupdate) and
[knsupdate](https://www.knot-dns.cz/docs/3.5/html/man_knsupdate.html).

`update` is now in a loop to accomodate multi-master or PRIMARY-PRIMARY
Name Server setups where the DNS sync is configure out-of-band. e.g. `rsync`

### 5th: `waitns`

`test` conditions rewritten to help readability.
IMHO this is easier to comprehend at a glance:
```sh
[ "$(${DIG} @${NS} _acme-challenge.${CH_HOSTNAME}. TXT +short | sed 's/"//g')" == "${CH_TXT_VALUE}" ] && return 0
```
than [this](https://github.com/hlandau/acmetool/blob/master/_doc/contrib/dns.hook#L51):
```sh
[ "$(dig +short "@${ns}" TXT "_acme-challenge.${CH_HOSTNAME}." | grep -- "$CH_TXT_VALUE" | wc -l)" == "1" ] && return 0
```
this is the only place in the script where `dig` and `nsupdate` do not use
the preset `NSERVERS` since by this point in the script, the deiscovered
Name Servers should be the same as `NSERVERS` plus secondary if using
in-band DSN sync.

### LICENSE

MIT [LICENSE](LICENSE), same as [acmetool](https://github.com/hlandau/acmetool#licence)
