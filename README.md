# observium-oxidized

Patch and instructions on how to use Oxidized for Observium

## Goals

1. Add proper Oxidized support to based on interaction with the Oxidized Web REST API **NOT** file or local repo access
2. Avoid breaking current RANCID support
3. Allow simultaneous RANCID & Oxidized usage
4. Present current Oxidized based config info in preference to existing RANCID based code detected config info
5. Support basic auth controlled access to Oxidized REST API from Observium, e.g. as enforced by Apache/nginx/etc or other middle box

Reference info:
1. https://docs.observium.org/rancid/
2. https://github.com/ytti/oxidized
3. https://github.com/ytti/oxidized-web

## Example Oxidized Config

Example config that I've been working with,

```
username: oxidized
password: PASSWORD
model: tmos
resolve_dns: false
interval: 3600
use_syslog: false
log: "/var/log/oxidized/oxidized.log"
debug: false
threads: 30
timeout: 20
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/
rest: 0.0.0.0:8888
next_adds_job: true
vars:
  enable: 'ENABLE_ME'
  remove_secret: false
  ssh_no_keepalive: true
  ssh_no_exec: false
  ssh_keys: [ "~/.config/oxidized/id_rsa" ]
  auth_methods: [ "publickey", "password", "keyboard-interactive", "none" ]
groups: {}
models:
  asa:
    vars:
      enable: PASSWORD
pid: "~/oxidized.pid"
crash:
  directory: "~/.config/oxidized/crashes"
  hostnames: false
stats:
  history_size: 10
input:
  default: ssh
  debug: false
  ssh:
    secure: false
output:
  default: git
  debug: false
  git:
    user: Oxidized
    email: oxidized@your.tld
    single_repo: true
    repo: "~/.config/oxidized/default.git"
source:
  default: http
  debug: false
  http:
    url: https://observium.your.tld/api/v0/oxidized
    map:
      name: hostname
      model: os
      group: type
    headers:
      Authorization: 'Basic YWRtaW46YWRtaW4='
model_map:
  cisco: ios
  juniper: junos
  f5: tmos
  bigip: tmos
  gaia: gaiaos
```

The important part above is the source http configuration.

This is telling Oxidized to talk to the Observium API to obtain a host list to attempt to obtain configurations from.

NOTE: Oxidized will currently get a list of ALL devices that Observium knows about. If it doesn't know how to talk to a particular model it won't try (refer to the model map for some mappings I use to change Observium device types to Oxidized device models, e.g. CheckPoint GAiA and Cisco...)

WARNING: For everything Oxidized thinks it does know how to talk to it WILL attempt to do so; which may lead to lots of failed auth attempt logs in your environment if the account it's trying to use can't access the device.

NOTE: Observium does not have credentials for the devices. You MUST configure these as default values within the oxidized configuration, utilise SSH key based auth, and/or modify model or group specific vars to ensure Oxidized will be able to login and run the commands it needs to run.

## Example Observium Config Modifications

You can set all of the following. The first option to enable oxidized web interaction is the only mandatory option. Default for it is false. Defaults for web url and auth header/header value are otherwise as below.

```
$config['oxidized']['web']['enable'] = TRUE;
$config['oxidized']['web']['url'] = 'http://127.0.0.1:8888';
$config['oxidized']['web']['auth']['header'] = '';
$config['oxidized']['web']['auth']['value'] = '';
```

## Observium Patching

```
cd /path/to/your/observium
patch -Np1 -i /path/to/the/patch/observium_r9768_oxidized.diff
```

Tested against latest Observium trunk and stable as of right now, refer to revisions in output below.

Output should look something like this,

```
[root@box observium]# rm -rf test
[root@box observium]# cp -R observium-trunk test
[root@box observium]# cd test
[root@box test]# svn info | grep Revision
Revision: 9773
[root@box test]#
[root@box test]# patch -Np1 -i ../observium_r9768_oxidized.diff
patching file html/api/v0/includes/oxidized.inc.php
patching file html/api/v0/index.php
patching file includes/definitions/transports.inc.php
patching file includes/alerting/oxidized.inc.php
patching file includes/defaults.inc.php
patching file html/pages/device/showtech.inc.php
patching file html/pages/device.inc.php
patching file html/pages/device/showconfig.inc.php
[root@box test]# cd
[root@box observium]# rm -rf test
[root@box observium]# cp -R observium-stable test
[root@box observium]# cd test
[root@box test]# svn info | grep Revision
Revision: 9773
[root@box test]# patch -Np1 -i ../observium_r9768_oxidized.diff
patching file html/api/v0/includes/oxidized.inc.php
patching file html/api/v0/index.php
patching file includes/definitions/transports.inc.php
patching file includes/alerting/oxidized.inc.php
patching file includes/defaults.inc.php
patching file html/pages/device/showtech.inc.php
patching file html/pages/device.inc.php
patching file html/pages/device/showconfig.inc.php
[root@box test]#
```

## Observium WebUI Interactions

### Viewing Device Configuration

Pretty simple,

1) If config and repo revisions are found using the existing RANCID code THOSE will be displayed and no communication with the Oxidized API will occur. e.g. you will either get config from plain text file, or from local cvs or git repo using existing RANCID code.
2) If config and repo revisions are NOT found using the existing RANCID code, Observium will **attempt** to communicate with the Oxidized API if configured to do so. If Oxidized knows about the device and can provide a config it will be displayed, if Oxidized can also provide a diff and revision history list these will be displayed.

Appearance within Observium is the same as how the current RANCID code will display plain text or cvs/git repo detected config, config diff and revisions.

When you first click on a device you'll see the most recently obtained running config,
![ASAv Latest Config View](/screenshots/asa_1.png?raw=true "ASAv 1 Latest Config View")

Clicking on "Show difference with previous revision (REV):" will show you the diff,
![ASAv Latest Config Diff](/screenshots/asa_2.png?raw=true "ASAv 1 Latest Config Diff")

Clicking on other config revisions will show you the config and offer a diff, provided it's not the first revision, in which case there's no diff :-)
![ASAv Latest Config First Revision](/screenshots/asa_3.png?raw=true "ASAv 1 Latest Config First Revision")

## Device with an existing RANCID code detected config, no Oxidized config

![Show Tech RANCID Config No Oxidized Config](/screenshots/existing_rancid_device_with_no_oxidized_config.png?raw=true "Show Tech RANCID Config No Oxidized Config")

![Show Tech Both RANCID Config And Oxidized Config](/screenshots/asa_with_existing_rancid_config_and_oxidized_config.png?raw=true "Show Tech Both RANCID Config And Oxidized Config")

### Creating an Oxidized API Alert Contact

You may want to do this if you're pulling syslog into Observium; as it means you can create a syslog based alert that will trigger Oxidized to go and check the device for the most recent configuration.

By doing this Oxidized will keep track of changes in near real time; and ensure, as best as it possibly can, that change diffs show individual changes by users rather big batches of changes potentially made by multiple users.

This is what an alert contact should basically look like,

![Oxidized API Alert Contact](/screenshots/oxidized_api_alert_contact.png?raw=true "Oxidized API Alert Contact")

### Observium Show Tech

Observium can dump what it knows about a device. Oxidized support has been added here alongside the existing RANCID support.

![ASAv Firewall - Observium Show Tech](/screenshots/asa_show_tech.png?raw=true "ASAv Firewall - Observium Show Tech")

# TODO

1. Move some of the code that communications with Oxidized API into helper functions within includes/functions.inc.php alongside get_rancid_filename()
2. Improve Observium API endpoint for Oxidized so it can filter for the actual devices we want it to collect config for. e.g. using names, model/device types etc.
3. Cleanup code generally - still a bit hacky.
4. Add Oxidized options to web settings editor for integrations, e.g. at /settings/section=integration/

# EOF
