---
author:
    name: Elle Krout
    email: ekrout@linode.com
description: 'Basic instructions to setup and configure a Puppet master and agents using Ubuntu or CentOS servers.'
keywords: 'puppet installation,configuration change management,server automation'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
alias: ['websites/puppet/basic-puppet-setup-and-configuration/','websites/puppet/manage-and-automate-systems-configuration-with-puppet/','applications/puppet/set-up-puppet-master-agent/']
modified: Friday, September 9th, 2016
modified_by:
    name: Phil Zona
published: 'Thursday, September 17th, 2015'
title: Install and Configure Puppet
external_resources:
    - '[Puppet Labs](https://puppetlabs.com/)'
    - '[Puppet Open Source Documentation](https://docs.puppetlabs.com/puppet/)'
---

[Puppet](https://puppetlabs.com/) is a configuration automation platform that simplifies various system administrator tasks. Puppet uses a client/server model where the managed servers, called *Puppet agents*, talk to and pull down configuration profiles from the *Puppet master*.

![Install and Configure Puppet](/docs/assets/install-puppet-title.png "Install and Configure Puppet")

Puppet is written in its own custom language, meant to be accessible to system administrators. A module, located on the Puppet master, describes the desired system. The Puppet software then translates the module into code and alters the agent servers as needed when the `puppet agent` command is run on an agent node or automatically at designated intervals.

Puppet can be used to manage multiple servers across various infrastructures, from a group of personal servers up to an enterprise level operation. It is intended to run on Linux and other Unix-like operating systems, but has also been ported to Windows. For the purpose of this guide, however, we will be working with an Ubuntu 14.04 LTS master server and two agent nodes: one Ubuntu 14.04, and one CentOS 7.

{: .note}
>
>Begin this guide as the `root` user. A limited user with administrative privileges will be configured in later steps.

## Before You Begin

1.  You should have three available Linodes, one of which has at least four CPU cores for the Puppet master. A [Linode 8GB](/pricing) plan is recommended. The two other nodes can be of any plan size, depending on how you intend to use them after Puppet is installed and configured.

2.  Follow the [Getting Started](/docs/getting-started) guide and ensure your Linodes are configured to use the same timezone.

    {: .note}
    >
    >For ease of use, set the Puppet master server's hostname to `puppet`, and have a valid fully-qualified domain name (FQDN).
    >
    >To check your hostname, run `hostname` and to check your FQDN, run `hostname -f`.

## Puppet Master

### Install Puppet Master

1.  Enable the `puppetlabs-release` repository on Ubuntu 14.04, unpackage it and update your system. This process downloads a `.deb` file that will configure the repositories for you:

        wget https://apt.puppetlabs.com/puppetlabs-release-trusty.deb
        dpkg -i puppetlabs-release-trusty.deb
        apt-get update
        
    {: .note}
    >
    >If you wish to run another Linux distribution as your master server, the initial `.deb` file can be substituted for another distribution based on the following formats:
    >
    >-  Red Hat-based systems:
    >       
    >        wget puppetlabs-release-pc1-OS-VERSION.noarch.rpm
    >
    >-  Debian-based systems:
    >
    >        wget puppetlabs-release-pc1-VERSION.deb
    >
    >Any Ubuntu-specific commands will then have to be amended for the proper distribution. More information can be found in [Puppet's Installation Documentation](https://docs.puppetlabs.com/puppet/4.0/reference/install_linux.html#install-a-release-package-to-enable-puppet-labs-package-repositories) or our guide to [package management](https://www.linode.com/docs/tools-reference/linux-package-management).

2.  Install the `puppetmaster-passenger` package:

        apt-get install puppetmaster-passenger

3.  Ensure you have the latest version of Puppet by running:

        puppet resource package puppetmaster ensure=latest

### Configure Puppet Master

1.  Update `/etc/puppet/puppet.conf` and add the `dns_alt_names` line to the section `[main]`, replacing `puppet.example.com` with your own FQDN:

    {: .file-excerpt}
    /etc/puppet/puppet.conf
    :   ~~~ conf
        [main]
        dns_alt_names = puppet,puppet.example.com
        ~~~

    Remove the line `templatedir=$confdir/templates`, which has been deprecated.

2.  Start the puppet master:

        service puppetmaster start

## Puppet Agents

### Install Puppet Agent

On agent nodes running **Ubuntu 14.04** or other Debian-based distributions, use this command to install Puppet:

        apt-get install puppet

On agent nodes running **CentOS 7** or other Red Hat systems, follow these steps:

1.  Add the Puppet Labs repository:

        rpm -ivh https://yum.puppetlabs.com/el/7/products/x86_64/puppetlabs-release-7-11.noarch.rpm
        
    {: .note}
    >
    >If on a Red Hat system other than CentOS 7, skip this step.
        
2.  Install the Puppet agent:

        yum install puppet

### Configure Puppet Agent

1.  Add the `server` value to the `[main]` section of the node's `puppet.conf` file, replacing `puppet.example.com` with the FQDN of your Puppet master:

    {: .file-excerpt}
    /etc/puppet/puppet.conf
    :   ~~~ conf
        [main]
        server = puppet.example.com
        ~~~

2.  Restart the Puppet service. For Debian/Ubuntu users:

        service puppet restart
    
    For CentOS 7/Red Hat users:

        systemctl restart puppet

### Generate and Sign Certificates

1.  Generate a certificate for the Puppet master to sign:

        puppet agent -t
        
    It will output an error, stating that no certificate has been found. This error is because the generated certificate needs to be approved by the Puppet master.

2.  Log in to your **Puppet master** and list the certifications that need approval:

        puppet cert list

    It should output a list with your agent node's hostname.

3.  Approve the certificate, replacing `hostname.example.com` with your node's name:

        puppet cert sign hostname.example.com

4.  Return to the **Puppet agent** node and run the Puppet agent again:

        puppet agent -t

5.  Repeat Steps 1-4 on any other Puppet agent nodes.

## Add Modules to Configure Agent Nodes

Both the Puppet master and agent nodes configured above are functional, but not fully secure. Based on concepts from the [Securing Your Server](/docs/security/securing-your-server/) guide, a limited user and a firewall should be configured. This can be done on all nodes through the creation of basic Puppet modules, shown below. This section is optional, but recommended.

{: .note}
>
>This is not meant to provide a basis for a fully-hardened server, and is intended only as a starting point. Alter and add firewall and other configuration options, depending on your specific needs.

### Add a Limited User

1.  From the **Puppet master**, navigate to the `modules` directory and create your new module for adding user accounts, then `cd` into that directory:

        cd /etc/puppet/modules
        mkdir accounts
        cd accounts

2.  Create the following directories, which are needed to have a functioning module:

        mkdir {examples,files,manifests,templates}

    The `examples` directory allows you to test the module locally. `files` contains any static files that may need to be edited or added. `manifests` contains the actual Puppet code for the module, and `templates` contains any non-static files that may be needed.

3.  Move to the `manifests` directory and create your first class, called `init.pp`. All modules require an `init.pp` file to be used as the main definition file of a module.

4.  Within the `init.pp` file, define a limited user to use instead of `root`, replacing all instances of `username` with your chosen username:

    {: .file}
    /etc/puppet/modules/accounts/manifests/init.pp
    :   ~~~ pp
        class accounts {

          user { 'username':
            ensure      => present,
            home        => '/home/username',
            shell       => '/bin/bash',
            managehome  => true,
            gid         => 'username',
            }

        }
        ~~~

    The `init.pp` file initially defines the `accounts` class. It then calls for the `user` resource, where a `username` is defined. The `ensure` value is set to ensure that the user is present. The `home` value should be set to the user's home directory path. `shell` defines the shell type, in this instance the bash shell. `managehome` notes that the home directory should be created. Finally, `gid` sets the primary group for the user.
    
5.  Although the primary group is set to share the username, the group itself has not been created. Save and exit `init.pp`. Then, create a new file called `groups.pp` and add the following contents. This file will be used to create the user's group. Again, replace `username` with your chosen username:

    {: .file}
    /etc/puppet/modules/accounts/manifests/groups.pp
    :   ~~~ pp
        class accounts::groups {
        
          group { 'username':
            ensure  => present,
          }
          
        }
        ~~~
        
     Include this file by adding `include groups` to the `init.pp` file, within the `accounts` class:
     
     {: .file-excerpt}
     /etc/puppet/modules/accounts/manifests/init.pp
     :  ~~~ pp
        class accounts {
        
          include groups
          
        ...
        ~~~
    
6.  This user should have privileges so that administrative tasks can be performed. Because we have agent nodes on both Debian- and Red Hat-based systems, the new user needs to be in the `sudo` group on Debian systems, and the `wheel` group on Red Hat systems. This value can be set dynamically through the use of a selector and *facter*, a program included in Puppet that keeps track of information, or *facts*, about every server. Add a selector statement to the top of the `init.pp` file within the accounts class brackets, defining the two options:

    {: .file-excerpt}
    /etc/puppet/modules/accounts/manifests/init.pp
    :   ~~~ pp
        class accounts {
        
          $rootgroup = $osfamily ? {
            'Debian'  => 'sudo',
            'RedHat'  => 'wheel',
            default   => warning('This distribution is not supported by the Accounts module'),
          }
          
          ...
          
        }        
        ~~~
        
    This command sequence tells Puppet that within the *accounts* module the variable `$rootgroup` should evaluate, using facter, the operating system family (`$osfamily`), and if the value returned is `Debian`, to set the `$rootgroup` value to `sudo`. If the value returned is `RedHat`, this same value should be set to `wheel`; otherwise, the `default` value will output a warning that the distribution selected is not supported by this module.
    
7.  Add the `groups` value to the user resource, calling to the `$rootgroup` variable defined in the previous step:

    {: .file-excerpt}
    /etc/puppet/modules/accounts/manifests/init.pp
    :   ~~~ pp
          user { 'username':
            ensure      => present,
            home        => '/home/username',
            shell       => '/bin/bash',
            managehome  => true,
            gid         => 'username',
            groups      => "$rootgroup",
          }
        ~~~
        
    The value `"$rootgroup"` is enclosed in double quotes (") instead of single quotes (') because it is a variable. Any value enclosed within single quotes will be added as typed in your module; anything enclosed in double quotes can accept variables, excepting hashed passwords.  

8.  A final value that needs to be added is the `password` value. Since we do not want to use plain text, it should be fed to Puppet as an SHA1 digest, which is supported by default. Set a password:

        openssl passwd -1

    You will be prompted to enter your password and confirm. A hashed password will be output. This should then be copied and added to the `user` resource:

    {: .file}
    /etc/puppet/modules/accounts/manifests
    :   ~~~ pp
        class accounts {

          user { 'username':
            ensure      => present,
            home        => '/home/username',
            shell       => '/bin/bash',
            managehome  => true,
            gid         => 'username',
            groups      => "$rootgroup",
            password    => '$1$07JUIM1HJKDSWm8.NJOqsP.blweQ..3L0',
            }

        }
        ~~~

    {: .caution}
    >
    >The hashed password **must** be included in single quotes (').

9.  Use the puppet parser to ensure that the code is correct:

        puppet parser validate init.pp

    If nothing is returned, your code is correct. Otherwise, any errors that need to be addressed will be output.

10. Before the module can be tested further, navigate to the `examples` directory and create another `init.pp` file, this time to call to the `accounts` module:

    {: .file}
    /etc/puppet/modules/accounts/examples/init.pp
    :   ~~~ pp
        include accounts
        ~~~

11. Test the module without making changes:

        puppet apply --noop init.pp
        
    {: .note}
    >
    >The `--noop` parameter prevents Puppet from actually running the module.

    It should return:

        Notice: Compiled catalog for puppet.example.com in environment production in 0.07 seconds
        Notice: /Stage[main]/Accounts/User[username]/ensure: current_value absent, should be present (noop)
        Notice: Class[Accounts]: Would have triggered 'refresh' from 1 events
        Notice: Stage[main]: Would have triggered 'refresh' from 1 events
        Notice: Finished catalog run in 0.01 seconds

12. Run `puppet apply` to make these changes to the Puppet master server:

        puppet apply init.pp

13. Log out as `root` and log in to the Puppet master as your new user. The rest of this guide will be run by this user.

### Edit SSH Settings

Although a new user has successfully been added to the Puppet master, the account is still not secure. Root access should be disabled for the server before continuing.

1.  Navigate to `files` within the `account` module directory:

        cd /etc/puppet/modules/accounts/files
        
2.  Copy the `sshd_config` file to this directory:

        sudo cp /etc/ssh/sshd_config .
        
3.  Open the file with `sudo`, and set the `PermitRootLogin` value to `no`:

    {: .file-excerpt}
    /etc/puppet/modules/accounts/files/sshd_config
    :   ~~~ config
        PermitRootLogin no
        ~~~
        
4.  Navigate back to the `manifests` directory and create a file called `ssh.pp`. Use the `file` resource to replace the default configuration file with the one managed by Puppet:

    {: .file}
    /etc/puppet/modules/accounts/manifests/ssh.pp
    :   ~~~ pp
        class accounts::ssh {
        
          file { '/etc/ssh/sshd_config':
            ensure  => present,
            source  => 'puppet:///modules/accounts/sshd_config',
          }
        
        }
        ~~~

    {: .note}
    >
    > The `file` directory is omitted from the `source` line because the `files` folder is the default location of files.
    
5.  Create a second resource to restart the SSH service and set it to run whenever `sshd_config` is changed. This will also require a selector statement because the SSH service is called `ssh` on Debian systems and `sshd` on Red Hat:

    {: .file-excerpt}
    /etc/puppet/modules/accounts/manifests/ssh.pp
    :   ~~~ pp

          ...

          $sshname = $osfamily ? {
            'Debian'  => 'ssh',
            'RedHat'  => 'sshd',
            default   => warning('This distribution is not supported by the Accounts module'),
          }

          file { '/etc/ssh/sshd_config':
            ensure  => present,
            source  => 'puppet:///modules/accounts/sshd_config',
            notify  => Service["$sshname"],
          }
        
          service { "$sshname":
            hasrestart  => true,
          }
        ~~~
        
6.  Include the `ssh` class within `init.pp`:

    {: .file-excerpt}
    /etc/puppet/modules/accounts/manifests/init.pp
    :   ~~~ pp
        class account {
        
          include groups        
          include ssh
        
        ...
        ~~~
        
7.  Run the Puppet parser, then navigate to the `examples` directory to test and run `puppet apply`:

        sudo puppet parser validate ssh.pp
        cd ../examples
        sudo puppet apply --noop init.pp
        sudo puppet apply init.pp
        
8.  To ensure that the `ssh` class is working properly, log out and then try to log in as `root`. You should not be able to. Consequently, log in as your limited user again.
        
### Add and Configure IPtables

1.  Install Puppet Lab's firewall module from the Puppet Forge:

        sudo puppet module install puppetlabs-firewall

2.  Navigate to the `manifests` directory under the new `firewall` module.

        cd /etc/puppet/modules/firewall/manifests/

3.  Create a file titled `pre.pp`, which will contain all basic networking rules that should be run first:

    {: .file}
    /etc/puppet/modules/firewall/manifests/pre.pp
    :   ~~~ pp
        class firewall::pre {

          Firewall {
            require => undef,
          }

           # Accept all loopback traffic
          firewall { '000 lo traffic':
            proto       => 'all',
            iniface     => 'lo',
            action      => 'accept',
          }->

           #Drop non-loopback traffic
          firewall { '001 reject non-lo':
            proto       => 'all',
            iniface     => '! lo',
            destination => '127.0.0.0/8',
            action      => 'reject',
          }->

           #Accept established inbound connections
          firewall { '002 accept established':
            proto       => 'all',
            state       => ['RELATED', 'ESTABLISHED'],
            action      => 'accept',
          }->

           #Allow all outbound traffic
          firewall { '003 allow outbound':
            chain       => 'OUTPUT',
            action      => 'accept',
          }->

           #Allow ICMP/ping
          firewall { '004 allow icmp':
            proto       => 'icmp',
            action      => 'accept',
          }

           #Allow SSH connections
          firewall { '005 Allow SSH':
            dport    => '22',
            proto   => 'tcp',
            action  => 'accept',
          }->

           #Allow HTTP/HTTPS connections
          firewall { '006 HTTP/HTTPS connections':
            dport    => ['80', '443'],
            proto   => 'tcp',
            action  => 'accept',
          }

        }
        ~~~

    Each rule is explained via commented text. More information can also be found on the [Puppet Forge Firewall](https://forge.puppetlabs.com/puppetlabs/firewall) page.

4.  In the same directory create `post.pp`, which will run any firewall rules that need to be input last:

    {: .file}
    post.pp
    :   ~~~ pp
        class firewall::post {

          firewall { '999 drop all':
            proto  => 'all',
            action => 'drop',
            before => undef,
          }

        }
        ~~~

    These rules will direct the system to drop all inbound traffic that is not already permitted in the firewall.

5.  Run the Puppet parser on both files to ensure the code does not return any errors:

        sudo puppet parser validate pre.pp
        sudo puppet parser validate post.pp

6.  Move up a directory, create a new `examples` directory, and navigate to it:

        cd ..
        sudo mkdir examples
        cd examples

7.  Within `examples`, create an `init.pp` file to test the firewall on the Puppet master:

    {: .file}
    init.pp
    :   ~~~ pp
        resources { 'firewall':
          purge => true,
        }

        Firewall {
          before        => Class['firewall::post'],
          require       => Class['firewall::pre'],
        }

        class { ['firewall::pre', 'firewall::post']: }

        firewall { '200 Allow Puppet Master':
          dport          => '8140',
          proto         => 'tcp',
          action        => 'accept',
        }
        ~~~

    This code block ensures that `pre.pp` and `post.pp` run properly, and adds a Firewall rule to the Puppet master to allow nodes to access it.

8.  Run the `init.pp` file through the Puppet parser and then test to see if it will run:

        sudo puppet parser validate init.pp
        sudo puppet apply --noop init.pp

    If successful, run the `puppet apply` without the `--noop`:

        sudo puppet apply init.pp

9.  Once Puppet is done running, check the iptables rules:

        sudo iptables -L

    It should return:

        Chain INPUT (policy ACCEPT)
        target     prot opt source               destination
        ACCEPT     all  --  anywhere             anywhere             /* 000 lo traffic */
        REJECT     all  --  anywhere             127.0.0.0/8          /* 001 reject non-lo */ reject-with icmp-port-unreachable
        ACCEPT     all  --  anywhere             anywhere             /* 002 accept established */ state RELATED,ESTABLISHED
        ACCEPT     icmp --  anywhere             anywhere             /* 004 allow icmp */
        ACCEPT     tcp  --  anywhere             anywhere             multiport ports ssh /* 005 Allow SSH */
        ACCEPT     tcp  --  anywhere             anywhere             multiport ports http,https /* 006 HTTP/HTTPS connections */
        ACCEPT     tcp  --  anywhere             anywhere             multiport ports 8140 /* 200 Allow Puppet Master */
        DROP       all  --  anywhere             anywhere             /* 999 drop all */

        Chain FORWARD (policy ACCEPT)
        target     prot opt source               destination

        Chain OUTPUT (policy ACCEPT)
        target     prot opt source               destination
        ACCEPT     tcp  --  anywhere             anywhere             /* 003 allow outbound */


### Add Modules to the Agent Nodes

Now that the `accounts` and `firewall` modules have been created, tested, and run on the Puppet master, it is time to add them to the Puppet agent nodes created earlier.

1.  From the **Puppet master**, navigate to `/etc/puppet/manifests`.

        cd /etc/puppet/manifests

2.  List all available agent nodes:

        sudo puppet cert list -all

3.  Create the file `site.pp` to define which nodes will take what modules:

    {: .file}
    site.pp
    :   ~~~ pp
        node 'ubuntuagent.example.com' {
          include accounts

          resources { 'firewall':
            purge => true,
          }

          Firewall {
            before        => Class['firewall::post'],
            require       => Class['firewall::pre'],
          }

          class { ['firewall::pre', 'firewall::post']: }

        }
        
        node 'centosagent.example.com' {
          include accounts

          resources { 'firewall':
            purge => true,
          }

          Firewall {
            before        => Class['firewall::post'],
            require       => Class['firewall::pre'],
          }

          class { ['firewall::pre', 'firewall::post']: }

        }
        ~~~

    This includes the `accounts` module and uses the same firewall settings as above to ensure that the firewall rules are applied properly.

4.  On each **Puppet agent node**, enable the `puppet agent` command:

        puppet agent --enable

5.  Run the Puppet agent:

        puppet agent -t

6.  To ensure the Puppet agent worked, log in as the limited user that was created and check the iptables:

        sudo iptables -L

Congratulations! You've successfully installed Puppet on a master and two agent nodes. Now that you've confirmed everything is working, you can create additional modules to automate configuration management on your agent nodes. For more information, see [Puppet module fundamentals](https://docs.puppet.com/puppet/latest/reference/modules_fundamentals.html). You can also install and use those modules others have created on the [Puppet Forge](https://docs.puppet.com/puppet/latest/reference/modules_installing.html). 
