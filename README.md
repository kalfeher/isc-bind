# isc-bind
Ansible role to deploy the COPR packaged version of ISC BIND. Suitable for RHEL/CentOS 7/8. Should work for Fedora, but not tested. Each configuration is deliberately simple. Default options are left as-is unless specifically required for the `ServerRole`.

The `BindRelease` variable will govern which release of BIND is used. There are options for Development, Stable and Extended Support Version branches.

## Templates and Defaults
Configuration templates will depend on the `ServerRole`.

### `ServerRole`: recursive
A recursive server requires no additional variables.
A basic recursive configuration file will be added to BIND.
If either `{{ListenrndcV4}}` or `{{ListenrndcV6}}` are not empty, then a rule will be added to firewalld allowing DNS inbound on the public zone:  `{{firewalldZone}}`.

A bind.keys file will be provided unless `key_file: "no"` (default is yes). This provides a method for BIND to prime its trust anchors. If the file is not provided, BIND will use its internal trust anchors. The file will only ever be used the first time BIND runs. An internet connected host should validate this file (within the `files` directory in the role ), prior to executing this role in a playbook. Signatures can be found here: https://downloads.isc.org/isc/bind9/keys/9.11/

#### Response Policy Zones (RPZ)
A basic RPZ configuration is included for `recursive` servers. If you do not use an RPZ feed and do not want to create a complex policy, you can use the included `basic_rpz` policy. The default policy focuses on blocking domains and nameservers that might host malicious or inappropriate content. It does not contain any client based policies.

To use the default rpz policy add domains to the `{{rpz_block_domains}}` list. _example:_
~~~
rpz_block_domains:
  - "xxx"
  - "sex"
  - "adult"
  - "evil.example.net"
  - "another.bad.domain.example.info"
~~~
The default rpz policy also allows you to specify nameserver names and nameserver IP addresses that you wish to block from your clients. Use `{{rpz_block_ns_label}}` for nameserver names to be blocked and `{{rpz_block_ns_ip}}` for nameserver IPs to be blocked. Take note of the IP format, which reverses the address and adds the bitmask at the start of the label.

RPZ source files should be named _policy_.db.j2 and will be searched for in the same order as described in **Zone File Naming Syntax**.

### `ServerRole`: primary
A primary server (authoritative) that can host either dynamically or statically updated zones. The default is for zones to be locally dynamic and signed.
Statically managed zones will need a template for the zone file and the zone statement for BIND's configuration file. An example of a statically managed zone (example.com) is included.

#### Configuring Domains
The dictionary `{{pri_domains}}` contains the list of domains that a server will be primary for. This variable must be set. The variable entry in the `defaults/main.yml` file is included as a guide.
An example zone file (`example.com.db.j2`) is included in the templates directory.

#### Zone File Naming Syntax
Zone files for domains within the `{{pri_domains}}` variable will be searched in the following order, with the first match used:
1. _domain_.db.j2 within the path specified by `{{ zone_file_path }}`
2. _domain_.db.j2 within the `templates` directory
3. _zonetype_.db.j2 within the path specified by `{{ zone_file_path }}`
4. _zonetype_.db.j2 within the `templates` directory

**Although the defaults within the provided _zonetype_.db.j2 files have been carefully considered, it is highly recommended that you use explicitly configured zone files.**

#### Static and Dynamic Zones
In order for this role to behave in an idempotent manner, dynamic zones are only managed for presence, not the content within the zone file.
The dictionary for each domain in `{{pri_domains}}` contains the variable *zonetype* which defines whether the domain will be updated dynamically via DNS TSIG (`zonetype:dyn`) or via changes to the file deployed by Ansible (`zonetype:static`).

The role will only update static zone files, after initially checking that all zone files are present. This is to avoid changes in a template from triggering the replacement of a dynamically updated zone file. Since static zone files are always overwritten on change when the playbook is run, it makes sense to keep the serial (which is a unix epoc timestamp) in a seperate file, otherwise any playbook run will result in an apparent change and BIND reload. Therefore static zone files must have the following entry for their SOA (the template `example.com.db.j2` can be copied and used for this purpose.):

~~~
$INCLUDE pri/{{ item.name }}/{{ item.name }}.serial
~~~

Because this role will not update dynamic zones after the first time they are added to the primary server, all subsequent changes must be made via standard DNS practices (typically nsupdate). This restriction only applies to the zone file of a dynamic zone. Changes to the zone statement (discussed in next section) within the configuration will be managed by this role.

#### Configuration file zone statements
A simple zone statement for BIND's configuration file will be used if you do not include a file with the statement in it. The path set by `{{zone_file_path}}` or in the `templates` directory will be searched for any zone statement files. Zone statement file names should be of the form _domain_.zone.j2. An example file (`example.com.zone.j2`) is included in the `templates` directory.

#### Parked Domains
TODO

## requirements.yml
Add the following lines to `requirements.yml` :
~~~
- src: https://github.com/kalfeher/isc-bind.git
  name: isc-bind
~~~

## Updating the role
There are two variables which will make updating the role much simpler.
`{{conf_file_path}}` should only be set if you are very familiar with BIND configuration. This will allow you to specify the location of configuration files which will not be replaced if the role is updated.

`{{zone_file_path}}` should be set for any host with that sets _primary_ as their `ServerRole`. This will allow you to specify a location for zone files and zone statement files which will not be replaced. You can and should create explicit zone files and place them in the `zone_file_path` directory. You do not need to also create the zone statement files for each domain. If the role defaults are acceptable, leave these out of the `zone_file_path` directory and the role will automatically use the zone statement file in the `templates` directory.

## Workarounds
During testing, BIND occasionally failed to listen to all its configured interfaces on boot up. This was due to Network Manager having not completed interface configuration, prior to BIND starting. The role will update the standard service file to wait for network manager to finish, before starting bind. This issue has been logged (https://gitlab.isc.org/isc-projects/bind9/issues/1594) and the workaround may be removed or altered based on how it is resolved by ISC. _update: A change has been made to fix this issue. This note will be removed soon._

The **dnssec-policy** feature has flaws that make it unusable in BIND 9.15.8. This feature should not be used (its currently only in-dev) and is known not to work. A fix has been create by ISC and this entry will be updated once that fix is released in any of the copr isc=bind releases.
