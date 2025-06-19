Ansible
=======

## Puppet

- `puppet agent`
  - an agent
    - runs on each managed node
    - invokes `facter` to collect facts about the node
    - sends the facts to the server and receives the catalog from the server
    - applies the catalog on the node
- `puppet apply`
  - this is standalone mode where the node
    - invokes `facter` to collect facts about the node
    - compiles the facts to the catalog according to the manifest
    - applies the catalog on the node
- `puppet config print` prints the config
  - `basemodulepath` defaults to `$codedir/modules:/usr/share/puppet/modules`
    - these are global modules used by all environments
  - `codedir` defaults to `/etc/puppetlabs/code` (root) or
    `~/.puppetlabs/etc/code` (non-root)
  - `confdir` defaults to `/etc/puppetlabs/puppet` (root) or
    `~/.puppetlabs/etc/puppet` (non-root)
  - `config` defaults `$confdir/$config_file_name`
  - `config_file_name` defaults to `puppet.conf`
  - `default_manifest` defaults to `./manifests` (relative to environment)
  - `environment` defaults to `production`
  - `environmentpath` defaults to `$codedir/environments`
  - `logdir` defaults to `/var/log/puppetlabs/puppet` (root) or
    `~/.puppetlabs/var/log` (non-root)
  - `rundir` defaults to `/var/run/puppetlabs` (root) or
    `~/.puppetlabs/var/run` (non-root)
  - `statedir` defaults to `$vardir/state`
  - `vardir` defaults to `/opt/puppetlabs/puppet/cache` (root) or
    `~/.puppetlabs/opt/puppet/cache` (non-root)
- `facter`
  - env `FACTER_foo=bar` sets `foo=bar` for the facter
