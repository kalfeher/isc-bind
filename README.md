# isc-bind
Ansible role to deploy the COPR packaged version of ISC BIND. Suitable for RHEL/CentOS 7/8. Should work for Fedora, but not tested. Each configuration is deliberately simple. Default options are left as-is unless specifically required for the serverrole.

The `BindRelease` variable will govern which release of BIND is used. There are options for Development, Stable and Extended Support Version branches.

## Templates and Defaults
Configuration templates will depend on the serverrole.

### serverrole: recursive
A recursive server requires no additional variables.
A basic recursive configuration file will be added to BIND.
If either {{ListenrndcV4}} or {{ListenrndcV6}} are not empty, then a rule will be added to firewalld allowing DNS inbound on the public zone:  {{firewalldZone}}.

### serverrole: primary
A primary server (authoritative) that can host either dynamically or statically updated zones. The default is for zones to be locally dynamic and signed.
Statically managed zones will need a template for the zone file and the zone statement for BIND's configuration file. An example of a statically managed zone (example.com) is included.

## requirements.yml
Add the following lines to `requirements.yml` :
~~~
- src: https://github.com/kalfeher/isc-bind.git
  name: isc-bind
~~~
