## Breaking Change Warning
This role will be distributed within a collection in the near future. At that time, the dns_update directory will move underneath the playbooks directory as per the standard collection structure. Functionality is not expected to change. 

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

To use the default rpz policy add domains to the `{{rpz_block_domains}}` list.
_For example:_
~~~
rpz_block_domains:
  - "xxx" # Included in default variable definition
  - "sex" # Included in default variable definition
  - "adult" # Included in default variable definition
  - "evil.example.net"
  - "another.bad.domain.example.info"
~~~
The default rpz policy also allows you to specify nameserver names and nameserver IP addresses that you wish to block from your clients. Use `{{rpz_block_ns_label}}` for nameserver names to be blocked and `{{rpz_block_ns_ip}}` for nameserver IPs to be blocked. Take note of the IP format, which reverses the address and adds the bitmask at the start of the label.

RPZ source files should be named _policy_.db.j2. The script will first search for this file within the path specified by `{{ zone_file_path }}` and then the `templates` directory.

#### DNS over TLS (DoT) and DNS over HTTPS (DoH)
The role provides an opinionated way of configuring DoT and DoH. The configuration is entirely driven by the `recursive-named.conf.j2` template. If you have specified a `{{conf_file_path}}` then you will need to add the required jinja2 elements into your custom template.

Both DoT and DoH require that the variable `isc_bind_tls` exist for the host. Define the variable as follows. You can define multiple TLS entries of any type.

Currently only the Development version (configured in `BindRelease`) of BIND is supported. If you are using the production or extended support versions of BIND then do not configure the `isc_bind_tls` variable.

##### Ephemeral
The simplest entry is to use the ephemeral key type that BIND will generate for you. The functionality of this key may change between BIND releases so caution is advised.
```yaml
isc_bind_tls:
  - name: 'ephemeral'
```
##### Default Static Certificate
If you wish to use a dedicated certificate but don't wish to use a custom TLS configuration then use the following definition for `isc_bind_tls`:
```yaml
isc_bind_tls:
  - name: 'mycert'
    cert: '/path/to/cert.pem'
    key: '/path/to/key.pem'
    hostname: 'myhost.example.com'
```
This configuration will use TLS1.3 exclusively. Since DoT and DoH are very recent technologies, there should be no backwards compatibility issues. Mozilla's modern cipher list is used.
##### Custom TLS Configuration
You can import your own custom TLS configuration file (1 file per entry), using the following definition for `isc_bind_tls`:
```yaml
isc_bind_tls:
  - name: 'mycustomTLS'
    file: '/path/to/tls/file'
```
The contents of a custom TLS configuration file must adhere to the BIND `tls` statement grammar.
##### Using TLS for DoT and DoH
`tls` entries will be used for several purposes, therefore this role does not assume that each entry is intended to be used for DoT and DoH. In order to use a defined `tls` entry to listen for incoming DoT and DoH connections, add `Listen: True` to the entry:
```yaml
isc_bind_tls:
  - { name: 'ephemeral', Listen: True } # Will be used for DoH and DoT
  - { name: 'mycert', cert: 'cert.pem', key: 'key.pem', hostname: 'myhost.example.com'} # Will not be used for DoH and DoT
  - { name: 'mycert2', cert: 'cert2.pem', key: 'key2.pem', hostname: 'myhost2.example.com', Listen: True } # Will be used for DoH and DoT
```
##### SE Linux Changes
BIND will fail to access http ports under the default selinux rules. This role will update the `named_tcp_bind_http_port` selinux boolean to be `on` if any `isc_bind_tls` entry contains a `listen` parameter set to `True`. If you do not have such an entry, you must change the SE Linux boolean manually.


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

## dnssec-policy
The primary-named.conf.j2 template comes with a basic dnssec-policy. Because the `default` policy within BIND is subject to change as the result of an upgrade, you are advised to use an explicitly defined policy. The `simple` dnssec-policy is designed to be useful for most situations. Note that as of BIND 9.16.7 parent DS entries are no longer assumed based on time. Therefore you should run `rndc dnssec -checkds published <zonename>` once you have submitted your DS to your registrar and until the DS is visible. A future version of this role will add a cron task to do this.

#### Parked Domains
TODO
### `ServerRole`: secondary

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

## Updating DNS Records
A playbook and variable file is supplied underneath the `dns_update` directory as an example. You do not need to use the dns_update playbooks with this role. Nor do you need DNS servers deployed by this role to use the dns_update playbooks. For further information there is a readme.md within the dns_update directory.

## Workarounds
None at the moment :)
