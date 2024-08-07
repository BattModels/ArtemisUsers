---
layout: default
title: Connecting to Artemis
parent: Getting Started
nav_order: 2
---

## Connecting to Artemis

Artemis is located at `lighthouse.arc-ts.umich.edu` to connect, enter the following into a
terminal:

```shell
ssh uniqname@lighthouse.arc-ts.umich.edu
```

To connect to Artemis, you must either be on the [campus network] or using [UMICH's Full VPN].

[campus network]: https://its.umich.edu/enterprise/wifi-networks
[UMICH's Full VPN]: https://its.umich.edu/enterprise/wifi-networks/vpn


### Artemis's SSH key fingerprints

Public Key fingerprints are used to validate the connection to a remote server.
When connecting to a server for the first time you will be prompted to verify
the authenticity of the server, by comparing the server's reported fingerprint to
the expected fingerprint.

Artemis's public key fingerprints are:

| Fingerprint | Key Type |
| ----------- | -------- |
| `SHA256:LwKoY7mWDU+TVxVODoxdY3bHFUYVVqUhcAX50VLgWfs` | RSA |
| `SHA256:Dae1G3gu0mtro2Rm15U6l8aQg4bGFnDQJhmGH3k+fKs` | ECDSA |
| `SHA256:9ho43xHw/aVo4q5AalH0XsKlWLKFSGuuw9lt3tCIYEs` | ED25519 |

> You **should not** connect to Artemis if the fingerprints do not match.
> Please open an [issue](https://github.com/BattModels/ArtemisUsers/issues)

### Accessing Artemis via UMICH's Full VPN

If you are not on campus, you must first connect to [UMICH's Full VPN]. Once connected
you can then `ssh` into Artemis.

## Configuring SSH

The following steps are quality-of-life improvements to default ssh configuration.
They are not required to use Artemis.

### Private Key Authentication

Instead of authenticating using a password, you can authenticate using
[Private Key Authentication] and skip typing in a password every time.

[Private Key Authentication]: https://help.ubuntu.com/community/SSH/OpenSSH/Keys

First, generate a set of keys to use in authentication. Skip this step if you
already have a public/private key pair (Look for `~/.ssh/id_rsa`);

```shell
ssh-keygen -t rsa
```

You will be prompted for a location to save your keys (The default is fine) and a
passpharse to unlock your keys. Follow the prompts as directed.

> Your private key `~/.ssh/id_rsa` is your password. Treat it as such and do
> not copy it to insecure systems, display it in public space, etc.

Now that you have a key pair, we need to transfer the *public* key to Artemis.
To do so, run the following command:

```shell
ssh-copy-id uniqname@lighthouse.arc-ts.umich.edu
```

You will be prompted for your key's passphrase (if set) and your password for Artemis.
You can now log in without entering your password!

### Using a SSH Config File

You can simplify logging in to Artemis by creating an `~/.ssh/config` file to
specify common option (i.e. your username) or create an alias for Artemis.

For example, the following config will let you connect with `ssh artemis`
instead of having to type out the full `ssh uniqname@lighthouse.arc-ts.umich.edu` each time.

``` conf
Host artemis
    User uniquename
    HostName lighthouse.arc-ts.umich.edu
```

### Sample SSH Configuration File

The following is a sample `~/.ssh/config` that enables some helpful features.

``` conf
# The * wild card means these options apply everywhere
Host *
    # The next three options configure ssh to reuse sockets to the same host.
    # This speeds up your connection time when ssh in from multiple terminals
    # to the same host
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%h-%p    # This is where ssh will save the sockets too
    ControlPersist 600                  # Key connection open for 600 seconds
    AddKeysToAgent yes              # Automatically adds keys to your ssh-agent
    IdentityFile ~/.ssh/id_rsa      # This set your default key for authentication

Host artemis
    User uniquename
    ForwardAgent yes
    # Forward your SSH agent, so you use your machine's ssh-agent to authenticate
    # to other machines from Artemis. This way you don't need to manage multiple
    # ssh keypairs to connect to a common service (i.e. GitHub)
    HostName lighthouse.arc-ts.umich.edu

```

For more configuration options see `man ssh_config` or
[ssh_config's documentation](https://man.openbsd.org/ssh_config).
