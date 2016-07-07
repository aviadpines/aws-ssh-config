aws-ssh-config
======

Description
---

A very simple script that queries the AWS EC2 API with boto and generates a SSH config file ready to use. 
There are a few similar scripts around but I couldn't find one that would satisfy all my wish list:

- Connect to all regions at once
- Do AMI -> user lookup (regexp-based)
- Support public/private IP addresses (for VPNs and VPCs)
- Support multiple instances with same tags (e.g. autoscaling groups) and provide an incremental count for duplicates based on instance launch time
- Support multiple customizable tags concatenations in a user-provided order
- Support region (with AZ) in the host name concatenation
- Properly leverage tab completion

Usage
---

This assumes boto is installed and configured.

Supported arguments:

```
$ python aws-ssh-config.py --help
usage: aws-ssh-config.py [-h] [--tags TAGS] [--private]

optional arguments:
  -h, --help   show this help message and exit
  --tags TAGS  A comma-separated list of tag names to be considered for
               concatenation. If omitted, all tags will be used
  --region          Append the region name at the end of the concatenation
  --private         Use private IP addresses (public are used by default)
  --profile         Specify aws credential profile to use
  --user            Override the ssh username for all hosts
  --default-user    Default ssh username to use if we cannot detect from AMI name
  --prefix          Specify a prefix to prepend to all host names
  --name-filter     Specify a regex filter to filter the names of the instances
  --key-folder      location of the identity files folder
  --proxy           regex of the proxy server, all other hosts (not excluded) will be using it to connect
```

By default, it will name hosts by concatenating all tags:

```
$ python aws-ssh-config.py > ~/.ssh/config
$ cat ~/.ssh/config
Host dev-worker-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host dev-worker-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host prod-worker-1
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no

Host prod-worker-2
    HostName 54.174.115.242
    User ubuntu
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no
```

It's possible to customize which tags one is interested in, as well as the order used for concatenation:

```
~$ python aws-ssh-config.py --tags Name > ~/.ssh/config
~$ cat ~/.ssh/config
Host worker-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host worker-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host worker-3
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no

Host worker-4
    HostName 54.174.115.242
    User ubuntu
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no

$ python aws-ssh-config.py --tags Name,Infrastructure > ~/.ssh/config
$ cat ~/.ssh/config
Host worker-dev-1
    HostName 54.173.109.173
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host worker-dev-2
    HostName 54.173.190.141
    User ec2-user
    IdentityFile ~/.ssh/dev.pem
    StrictHostKeyChecking no

Host worker-prod-1
    HostName 54.164.168.30
    User ec2-user
    IdentityFile ~/.ssh/prod.pem
    StrictHostKeyChecking no

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

Using (almost) all the flags together:

```
$  python aws-ssh-config.py --profile my_profile --tags Name --name-filter ".*my_host.*" --key-folder "~/my_keys/" --prefix "myprefix-" --proxy ".*bastion.*"
```


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