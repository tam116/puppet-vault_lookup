# vault_lookup

[![Build Status](https://github.com/voxpupuli/puppet-vault_lookup/workflows/CI/badge.svg)](https://github.com/voxpupuli/puppet-vault_lookup/actions?query=workflow%3ACI)
[![Release](https://github.com/voxpupuli/puppet-vault_lookup/actions/workflows/release.yml/badge.svg)](https://github.com/voxpupuli/puppet-vault_lookup/actions/workflows/release.yml)
[![Puppet Forge](https://img.shields.io/puppetforge/v/puppet/vault_lookup.svg)](https://forge.puppetlabs.com/puppet/vault_lookup)
[![Puppet Forge - downloads](https://img.shields.io/puppetforge/dt/puppet/vault_lookup.svg)](https://forge.puppetlabs.com/puppet/vault_lookup)
[![Puppet Forge - endorsement](https://img.shields.io/puppetforge/e/puppet/vault_lookup.svg)](https://forge.puppetlabs.com/puppet/vault_lookup)
[![Puppet Forge - scores](https://img.shields.io/puppetforge/f/puppet/vault_lookup.svg)](https://forge.puppetlabs.com/puppet/vault_lookup)
[![puppetmodule.info docs](http://www.puppetmodule.info/images/badge.png)](http://www.puppetmodule.info/m/puppet-vault_lookup)
[![Apache-2 License](https://img.shields.io/github/license/voxpupuli/puppet-vault_lookup.svg)](LICENSE)

Module to integrate Puppet 6 (and newer) and Puppet Enterprise 2019 (and newer)
agents with Hashicorp Vault.

#### Table of Contents

1. [Description](#description)
2. [Setup - The basics of getting started with vault_lookup](#setup)
3. [Usage - Configuration options and additional functionality](#usage)

## Description

For users with a Puppet Enterprise 2019 or open source Puppet 6 infrastructure
wanting to leverage secrets from an existing [Hashicorp
Vault](https://www.vaultproject.io/) server. Used with Puppet 6's Deferred type,
this allows agents to retrieve secrets from Vault when a catalog is applied. In
this way, the secret data is not embedded in the catalog and the master never
sees it. See [this blog
post](https://puppet.com/blog/secret-agents-man-secrets-store-integrations-puppet-6)
for more information and other secret store integrations.

Authentication with Vault is achieved via Puppet certificates. See the
Vault documentation for more information on setting up finer grained access
controls.

## Requirements

This is expected to be run using the `Deferred` type, which requires Puppet
6.0.0 or later, and of course [Vault](https://www.vaultproject.io/) to store the
data.

## Setup

The `vault_lookup` function uses the Puppet agent's certificates in order to
authenticate to the Vault server; this means that before any agents contact a
Vault server, you must configure the Vault server with the Puppet Server's CA
certificate, and Vault must be part of the same certificate infrastructure.

To set up Vault to use the Puppet Server CA cert:

1. Set up Vault using Puppet certs (if not already set up this way)
  If the Vault host has a Puppet agent on it then you can just use the existing
  certificates. Otherwise generate a new certificate with `puppetserver ca` and
  copy the files.

```
puppetserver ca generate --certname my-vault.my-domain.me
```

  In the Vault listener configuration, set `tls_client_ca_file` as the Puppet CA
  cert, `tls_cert_file` as the agent or generated certificate, and
  `tls_key_file` as the agent or generated private key.

2. Enable cert auth for Vault
  Hashicorp’s Vault supports a variety of auth methods that are listed in their
  documentation; the auth method required for usage with the vault_lookup
  function is named cert, and can be turned on with the Vault CLI:

```
$ vault auth enable cert
```
3. Upload the Puppet Server CA certificate to Vault.
  After cert auth has been enabled for Vault, upload the CA certificate from
  your Puppet Server to Vault and add it as a trusted certificate.

```
$ vault write auth/cert/certs/puppetserver \
    display_name=puppet \
    policies=prod,test \
    certificate=@/path/to/puppetserver/ca.pem \
    ttl=3600
```

Once the certificate has been uploaded, any Puppet agent with a signed
certificate will be able to authenticate with Vault.

## Usage

Install this module as you would in any other; the necessary code will
be distributed to Puppet agents via pluginsync.

In your manifests, call the `vault_lookup::lookup` function using the Deferred
type. For example:

```puppet
$d = Deferred('vault_lookup::lookup', ["secret/test", 'https://vault.hostname:8200'])

node default {
  notify { example :
    message => $d
  }
}
```

The lookup function will be run on the agent and the value of `$d` will be
resolved when the catalog is applied. This will make a call to
`https://vault.hostname:8200/v1/secret/test` and wrap the result in Puppet's
`Sensitive` type, which prevents the value from being logged.

You can also choose not to specify the Vault URL, and then Puppet will use the
`VAULT_ADDR` environment variable. This will be either set on the command line, or
set in the service config file for Puppet, on Debian `/etc/default/puppet`, on RedHat
`/etc/sysconfig/puppet`:

```
$d = Deferred('vault_lookup::lookup', ["secret/test"])

node default {
  notify { example :
    message => $d
  }
}
```
