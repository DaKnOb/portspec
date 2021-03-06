# PortSpec

PortSpec is a tool to ensure all hosts in your network or under your control
have open ports that are up to the specification you set.

By setting a list of hosts and the ports you expect to be open, PortSpec will
run every predefined amount of time and scan these hosts for open TCP ports.
After all open ports are found, PortSpec will compare this list with the one
you defined, and it will alert you if something is not in spec.

Currently it is being under active development, and has very minimal
functionality, fit for a small particular purpose that it was developed for,
but hopefully some day it can work and have more features that will address
more needs.

## Configuration

PortSpec is configured solely by the `conf.yml` file. The path of this file
can be specified by the `-c` command line flag. So if you launch it with
`portspec -c /etc/portspec/portspec.yml`, it will read the configuration from
there. You can have multiple instances of PortSpec running with different
configuration files, in the same server, without issues.

The included `conf.yml` contains an example configuration file. Some of the
included options can be found here:

### interval

The `interval` setting controls how often (in seconds) should PortSpec scan
the hosts and report on the results. This clearly depends on your needs, but the
recommended values are 3600 (one hour) to 86400 (one day). Keep in mind that if
this value is too low, the previous scan may not finish before the new one
starts, but this usually happens if you have a large number of hosts, and this
is set to too low.

### ipv6

The `ipv6` setting is currently not implemented. However, when it is, if this
flag is `true`, it will also scan the IPv6 address of all hosts that are given
as a hostname (`example.com` vs `192.0.2.50`). This flag will default to `true`
very soon after the feature is implemented.

### parallelscans

The `parallelscans` setting is also not currently implemented. It will define
how many host scans should run in parallel. This is *host* scans and not *port*
scans, so adjust this value accordingly.

### hosts

The `hosts` setting is a list of hosts that will be scanned by PortSpec. It can
currently be either a domain name (`example.com`), an IPv4 Address
(`192.0.2.10`), or an IPv6 Address (`2001:db8::1337`). Each host needs to have
an array of ports that *SHOULD* be open. This list is the *expected* ports that
you want open, not the list of ports that will be scanned on this host. This
list can be empty, in case you expect no open TCP ports on this host.

An example of this configuration is:

```
hosts:
    "www.example.org":
        - 22
        - 80
        - 443
    "www.example.com":
        - 25
        - 587
    "example.net":
```

### scanports

The `scanports` setting is a list of ports that PortSpec will scan for in *all*
hosts. By default a pretty large list is included, but you can add and remove
ports as needed. Port ranges are currently not supported, but it is a feature
that is coming very soon.

## Alert Modules

Currently PortSpec supports mutliple modes of alerting the users. In this
section you can find configuration options for each implemented module.

### E-Mail

E-Mail is the most basic alert method and will cause PortSpec to send an e-mail
to a set of specified e-mail addresses. To configure e-mail, you will need to
add to the `conf.yml` the following:

#### sendemail

If you set `sendemail` to `true`, the e-mail alerts will be activated. Do not
include this or set this to `false`, and PortSpec will not send e-mail
notifications.

#### smtpserver

The `smtpserver` setting contains the address of the SMTP server to use for
outgoing e-mail. This can be a hostname (`smtp.example.com`), an IPv4 Address
(`192.0.2.10`), or an IPv6 Address (`2001:db8::1337`). Currently PortSpec
requires the server to support TLS in order to be able to login successfully.

#### smtpport

The `smtpport` setting is the port that PortSpec should contact the SMTP Server
at. The most common setting is `587`, but other options such as `2525` are also
possible.

#### smtpusername

The `smtpusername` setting is the username that PortSpec should connect to the
SMTP server as. Note that PortSpec uses TLS for the connection and Auth/Plain
as the login method.

#### smtppassword

The `smtppassword` setting is the password for the mail server. This password is
sent only if the server supports TLS, therefore it is encrypted during the
connection.

#### alertemail

The `alertemail` setting contains a list of e-mail addresses that will receive
an e-mail alert by PortSpec. One e-mail will be sent for each address in the
list, and not a single e-mail with multiple recipients.

#### fromemail

The `fromemail` setting contains the e-mail address all alerts will be sent as.
This value will be sent to the mail server as well as be included in the `From`
header of the e-mail. You can currently only control the e-mail address, and
not the name. This is currently hardcoded as `PortSpec`.

## Logs

Currently PortSpec tries to produce a large amount of logs. This is done so you
can go back and check for information that may be useful at some point in the
future. Great care has been given to ensure no private information is being
logged, other than user provided values such as hosts and IP Addresses.

The framework utilized for logging supports two modes, one that will output
data as human readable format, which is common for other services, such as web
servers, mail servers, etc. and an other that will output all logs in JSON, to
be used directly with a log processing tool of your choice. A third format is
available that is used when PortSpec runs in a terminal, and also supports
colors for greater visibility.

By default the first option is used, but you can switch between them using
command line flags. The `-j` flag will cause PortSpec to output JSON. This
option is not configurable using the config file to ensure that all logs, such
as configuration file parsing errors, are in the same format. This hopefully
minimizes parsing errors.

PortSpec logs to the Standard Output (`stdout`), so redirection or a proper
daemon handling system is needed to save the logs to files. For example,
`systemd` will save `stdout` to `/var/log/syslog`.

## Installation

To download and install PortSpec, either clone this repository and use Go to
build the executable, or download one of the prebuilt binaries.

PortSpec runs as a daemon, and must be running continuously. This can be done
in any way you want, such as a supervisor like `supervisord`, but for
distributions with `systemd`, this method is the one recommended. In this
repository you can find the `portspec.service` file. This is a `systemd` unit
file you can place to `/etc/systemd/system/` in order for Linux to recognize as
a service.

After you install this unit file, you need to run:

```
sudo systemctl enable portspec
```

This will enable the `portspec` service and will have it execute at system
boot. In order to start the service, you can use `sudo service portspec start`,
and to stop it, use `sudo service portspec stop`. 

Now you need to save the configuration file somewhere. The recommended path is
`/etc/portspec/portspec.yml`. Since this directory does not exist in your
system, you need to create it:

```
sudo mkdir /etc/portspec
```

Now either copy `conf.yml` from this repository, or start a new one from
scratch.

By default, `systemd` units are running as `root`. This is something you do not
want in your server, since this code may not be trusted, and in general you
should avoid running things as a privileged account. For this reason, you need
to create a `portspec` user for your system:

```
sudo adduser -s /usr/sbin/nologin -r -M portspec
```

Now that you have the `portspec` user, go and change the `/etc/portspec` folder
owner to this user:

```
sudo chown -R portspec:portspec /etc/portspec
```

Finally, make sure this file is only readable by the `portspec` user:

```
sudo chmod 600 /etc/portspec/portspec.yml
```

This is recommended since the configuration file may contain sensitive
information such as e-mail passwords, or API keys.