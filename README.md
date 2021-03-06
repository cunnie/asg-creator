# ASG Creator

[Application Security
Groups](http://docs.cloudfoundry.org/adminguide/app-sec-groups.html) (ASGs) are
used to whitelist outbound container network access in [Cloud
Foundry](http://cloudfoundry.org). Application containers require ASGs to
enable outbound network access.

Creating ASGs for the `default-staging` and `default-running` ASG sets can be
intimidating as they are defined as whitelists, but you want to specify
specific IPs and networks to be blacklisted. An example of addresses you would
want to blacklist would be those for VMs running Cloud Foundry system
components, or those for Marketplace Service VMs.

The ASG Creator can be used to create baseline public-networks and
private-networks ASGs that allow all public and private networks *except* those
you want to blacklist. Additionally, it will block by default the
169.254.169.254 link-local address that's used by multiple IaaS providers for
VM metadata.

You are encouraged to modify the files created by ASG Creator to suit your
needs.

## Installation

```
go get github.com/cloudfoundry-incubator/asg-creator
```

## Usage

Config Options

* *excluded_ips*: An array of IPs to exclude; these IPs will be omitted from the baseline ASG rules
* *excluded_networks*: An array of CIDRs to exclude; all IPs in these networks will be omitted from the baseline ASG rules

Create a config, `config.yml`:

```yaml
excluded_ips:
- 192.168.100.4

excluded_networks:
- 192.168.1.0/24
```

Use the config to create ASG rules files:

```
$ asg-creator create --config config.yml
Wrote public-networks.json
Wrote private-networks.json
OK
$ cat private-networks.json | jq '.'
[
  {
    "protocol": "all",
    "destination": "10.0.0.0-10.255.255.255"
  },
  {
    "protocol": "all",
    "destination": "172.16.0.0-172.31.255.255"
  },
  {
    "protocol": "all",
    "destination": "192.168.0.0-192.168.0.255"
  },
  {
    "protocol": "all",
    "destination": "192.168.2.0-192.168.100.3"
  },
  {
    "protocol": "all",
    "destination": "192.168.100.5-192.168.255.255"
  }
]

$ cat public-networks.json | jq '.'
[
  {
    "protocol": "all",
    "destination": "0.0.0.0-9.255.255.255"
  },
  {
    "protocol": "all",
    "destination": "11.0.0.0-169.254.169.253"
  },
  {
    "protocol": "all",
    "destination": "169.254.169.255-172.15.255.255"
  },
  {
    "protocol": "all",
    "destination": "172.32.0.0-192.167.255.255"
  },
  {
    "protocol": "all",
    "destination": "192.169.0.0-255.255.255.255"
  }
]
```

Modify the ASG rules files per your network policy for application containers
running untrusted code.

Use the rules files to create ASGs with the [cf
CLI](https://github.com/cloudfoundry/cli/releases/latest):

```
$ cf create-security-group private-networks private-networks.json
$ cf bind-staging-security-group private-networks
$ cf bind-running-security-group private-networks

$ cf create-security-group public-networks public-networks.json
$ cf bind-staging-security-group public-networks
$ cf bind-running-security-group public-networks
```
