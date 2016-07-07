aws-ssh-config
======

Description
---

A very simple script that queries the AWS EC2 API with boto and generates a SSH config file ready to use. 
There are a few similar scripts around but I couldn't find one that would satisfy all my wish list:

- Connect to all regions at once
- Do AMI -> user lookup (regexp-based)
- Supports public/private IP addresses (for VPNs and VPCs)
- Supports multiple instances with same tags (e.g. autoscaling groups) and provide an incremental count for duplicates based on instance launch time
- Supports multiple customizable tags concatenations in a user-provided order
- Supports region (with AZ) in the host name concatenation
- Supports aws profiles
- Properly leverage tab completion
- Supports proxy server
- Supports dynamic forward (via proxy server)
- Supports global settings (keep-alive, host key check and strict host key check)

Usage
---

This assumes boto is installed and configured.

Supported arguments:

```
$ python aws-ssh-config.py --help
usage: aws-ssh-config.py [-h] [--tags TAGS] [--private]

optional arguments:
  -h, --help                show this help message and exit
  --tags TAGS               A comma-separated list of tag names to be considered for
                            concatenation. If omitted, all tags will be used
  --region REGION           Append the region name at the end of the concatenation
  --private                 Use private IP addresses (public are used by default)
  --profile PROFILE         Specify aws credential profile to use
  --user USER               Override the ssh username for all hosts
  --default-user USER       Default ssh username to use if we cannot detect from AMI name
  --prefix PREFIX           Specify a prefix to prepend to all host names
  --name-filter FILTER      Specify a regex filter to filter the names of the instances
  --key-folder FOLDER       location of the identity files folder
  --proxy PROXY             regex of the proxy server, all other hosts (not excluded) will
                            be using it to connect
  --dynamic-forward PORT    Use dynamic forwarding when opening the proxy defined
                            with --proxy
  --keep-alive KEEP         Set keep alive (seconds)
  --no-strict-check         Disable strict host key checking
  --no-host-key-check       Disable host key checking
```

By default, it will name hosts by concatenating all tags:

```
$ python aws-ssh-config.py > ~/.ssh/config
$ cat ~/.ssh/config
Host dev-worker-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host dev-worker-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host prod-worker-1
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem

Host prod-worker-2
    HostName 54.174.115.242
    User ubuntu
    IdentityFile ~/.ssh/prod.pem
```

It's possible to customize which tags one is interested in, as well as the order used for concatenation:

```
~$ python aws-ssh-config.py --tags Name > ~/.ssh/config
~$ cat ~/.ssh/config
Host worker-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host worker-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host worker-3
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem

Host worker-4
    HostName 54.174.115.242
    User ubuntu
    IdentityFile ~/.ssh/prod.pem

$ python aws-ssh-config.py --tags Name,Infrastructure > ~/.ssh/config
$ cat ~/.ssh/config
Host worker-dev-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host worker-dev-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem

Host worker-prod-1
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem

Host worker-prod-2
    HostName 54.174.115.242
    User ubuntu
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no

```

The default location of the keys is `~/.ssh/` but can be configured using `--key-folder`

```
$  python aws-ssh-config.py --key-folder "~/my_keys/"
```

By default, the ssh user is calculated from a regular expression based on the AMI name. A default user can be set with `--default-user` to use if no matches are found, otherwise a warning is printed on standard error and one can edit the script and add the rule to the `AMIS_TO_USER` dictionary:

```
$ python aws-ssh-config.py > ~/.ssh/config
Can't lookup user for AMI 'ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-20140926', add a rule to the script
```

The `--user` param can also be used to use a single username for all hosts.

The `--profile` param can be used to specify an aws credential profile to use

`--name-filte` can be used to filter the instances. Suppose all of your instances have "-dev" in them, you can use

```
$  python aws-ssh-config.py --name-filter ".*-dev.*"
```

If you want to configure a proxy server, use `--proxy` and specify the its regex, e.g.

```
$  python aws-ssh-config.py --proxy ".*-bastion"
```

The proxy will be used on all machines that are using private ip (except the proxy server itself).

When using a proxy you can define dynamic port to be opened when connecting to the proxy using `--dynamic-forward`.

#### Global Settings

Few of the options are configured using wild cards to avoid duplicating the lines throughout the entire list.
A host `<prefix>*` will be created with the following options:

`--no-strict-check` will disable strict host key checking (adding `StrictHostKeyChecking no`)

`--no-host-key-check` disables the warning `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` (adding `UserKnownHostsFile /dev/null`)

`--keep-alive` sets the keep-alive of all hosts in seconds.

example host that will be created (when not using a prefix it will simply be `*`)

```
 Host my-prefix-*
   StrictHostKeyChecking no
   UserKnownHostsFile /dev/null
   ServerAliveInterval 60
```

Using (almost) all the flags together:

```
$ python aws-ssh-config.py --profile my_profile             \
                           --tags Name                      \
                           --name-filter ".*my_server.*"    \
                           --key-folder "~/keys/"           \
                           --prefix "my-prefix-"            \
                           --proxy ".*bastion.*"            \
                           --dynamic-forward 8000           \
                           --no-strict-check                \
                           --no-host-key-check              \
                           --keep-alive 60
```

Public ips are used by default, and private are only used if a public ip cannot be found. If you want to use private ips on all machines, use `--private`.


Tab Completion
---

ssh completion will immediately work:

```
$ ssh d[TAB]
dev-worker-1
dev-worker-2
```
If the ssh completion will not immediately work you should add the following script to your `.bash_profile`

```
_complete_ssh_hosts ()
{
        COMPREPLY=()
        cur="${COMP_WORDS[COMP_CWORD]}"
        comp_ssh_hosts=`cat ~/.ssh/known_hosts | \
                        cut -f 1 -d ' ' | \
                        sed -e s/,.*//g | \
                        grep -v ^# | \
                        uniq | \
                        grep -v "\[" ;
                cat ~/.ssh/config | \
                        grep "^Host " | \
                        awk '{print $2}'
                `
        COMPREPLY=( $(compgen -W "${comp_ssh_hosts}" -- $cur))
        return 0
}
complete -F _complete_ssh_hosts ssh
```
and run `$ source ~/.bash_profile`