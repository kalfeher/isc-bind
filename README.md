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

#### Configuring Domains
The dictionary `{{pri_domains}}` contains the list of domains that a server will be primary for. This variable must be set. The `defaults/main.yml` entry is included as a guide.
An example zone file (`example.com.db.j2`) is included in the templates directory.

#### Configuration file zone statements
A simple zone statement will be used if you do not include a file with the statement in it within the path set by `{{zone_file_path}}` or in the `templates` directory. Zone statement files should be of the form _domain_.zone.j2. An example file (`example.com.zone.j2`) is included in the `templates` directory.

## requirements.yml
Add the following lines to `requirements.yml` :
~~~
- src: https://github.com/kalfeher/isc-bind.git
  name: isc-bind
~~~

## Updating the role
There are two variables which will make updating the role much simpler.
`{{conf_file_path}}` should only be set if you are very familiar with BIND configuration. This will allow you to specify the location of configuration files which will not be replaced if the role is updated.

`{{zone_file_path}}` should be set for any host with the _primary_ `{{serverrole}}`. This will allow you to specifiy a location for zone files and zone statement files which will not be replaced.
