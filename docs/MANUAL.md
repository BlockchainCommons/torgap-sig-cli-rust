# Manual

This document describes how to use `torgap-sig-cli-rust` and how to export and use `minisign` keys
in a Tor onion v3 hidden service.

All signatures produced by `torgap-sig-cli-rust` can be verified with `minisign`, including trusted comments.

`minisign` is also able to sign files with keys generated by `torgap-sig-cli-rust`.

`torgap-sig-cli-rust` is a command-line interface. It relies on the [torgap-sig crate](https://github.com/BlockchainCommons/torgap-sig), which can be embedded in any application.

## Usage

Generate a new key pair:

```sh
rsign generate
```
The public key is printed on the screen and stored in `rsign.pub` by default. The secret key will be written at `~/.rsign/rsign.key`. You can change the default paths with `-p` and `-s` respectively.

If you want to **generate keys from a deterministic seed** you can pass a hex string value with `--seed`:

```sh
rsign generate --seed 12a9e759cb410b4ca3a57a494939704e4bf71a59f01b94ea2740f15d3dc8f9ea
```

Once you have a secret key generated you can also **derive the extended private key** according to **SLIP010**:

```sh
rsign generate-xpriv --chain m/1H/0H/5H
```

You can then **sign a file** (`myfile.txt`) with your secret key:

```sh
rsign sign myfile.txt
```

You can **add a signed trusted comment** with the `-t` flag:

```sh
rsign sign myfile.txt -t "my trusted comment"
```

If you are signing files larger than 1Gb you must use `-H` to first hash the file and sign the hash after that:

```sh
rsign sign mylargefile.bin -H
```

To **verify the signature** with a given public key you can use the `verify` argument:

```sh
rsign verify myfile.txt -p rsign.pub
```

If you have saved the signature file with a custom name other than `myfile.txt.minisig` and want to use a public key string you can do that as well:

```sh
rsign verify myfile.txt -P [PUBLIC KEY STRING] -x mysignature.file
```

You can export your **secret key** to use it for a **Tor Onion v3 hidden service**. This will produce `hs_ed25519_secret_key`,
`hs_ed25519_public_key` and `hostname` in the same directory where your `secret key` is.

```sh
rsign export-to-onion-keys
```

You can then **verify** a file **with an onion v3 address**:

```sh
rsign verify myfile.txt --onion-address fscst5exmlmr262byztwz4kzhggjlzumvc2ndvgytzoucr2tkgxf7mid.onion
```

Your secret key can be converted to **Tor v3 authentication** keys for client authorization:
```sh
rsign export-to-tor-auth-keys
```

You can generate a **DID document** from your secret key:

```sh
rsign generate-did
```

You can find more information using the `help` argument:

```text
rsign help [SUBCOMMAND]

USAGE:
    rsign [SUBCOMMAND]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    export-to-onion-keys       Convert minisign secret key to Tor hidden service keys and hostname
    export-to-tor-auth-keys    Convert minisign secret key to Tor V3 Client Authentication Keys
    generate                   Generate minisign public and secret keys
    generate-xpriv             Generate extended private and public key from a secret key according to SLIP010.
                               First password will be prompted to decrypt your current secret key. Then a new
                               password will be prompted to encrypt the xpriv key
    generate-did               Generate DID document from a minisign secret key
    help                       Prints this message or the help of the given subcommand(s)
    sign                       Sign a file with a given private key
    verify                     Verify a signed file with a given public key
```

## Importing keys to Tor hidden service

Previous section describes how to export a secret for usage with Tor v3 onion service by using
`rsign export-to-onion-keys`. You then just need to copy `hs_ed25519_secret_key` into
your hidden service location, e.g. `var/lib/tor/my_hidden_service/`.

Next, set its permissions to the `debian-tor` group:

```bash
$ sudo chmod g+r /var/lib/tor/my_hidden_service/hs_ed25519_secret_key
```

Afteward, restart Tor:

```bash
$ sudo systemctl restart tor.service
```

Tor will automatically generate `hs_ed25519_public_key` and `hostname`.

## Setting up Client Authorization for v3 onion service

Excerpt [from](https://community.torproject.org/onion-services/advanced/client-auth/):

Client authorization is a method to make an onion service private and authenticated.

The command `rsign export-to-tor-auth-keys` generates a keypair (`tor_public_key.auth`,
`tor_secret_key.auth_private`) in the same folder as your minisign secret resides.

On the **server side**, place your `tor_public_key.auth` into `/var/lib/tor/<HiddenServiceDir>/authorized_clients/`
and restart Tor with `sudo systemctl reload tor`.

You can now reach the service with Tor browser. The browser will prompt to enter the private key. This means
if the content of your `tor_secret_key.auth_private` is
`o67ocen4p75qtittgne7pfvwgwkjzxgm6wbychizjd53jc3pc7k6egad:descriptor:x25519:5DEJ7HKMRRNURZVMD72BXV5VZP3XG2TMUBHEVRB36BCWAMV3T52A` you enter the last section: `5DEJ7HKMRRNURZVMD72BXV5VZP3XG2TMUBHEVRB36BCWAMV3T52A`

Alternatively, you can place `tor_secret_key.auth_private` into `Browser/TorBrowser/Data/Tor/onion-auth` and restart your Tor browser. The browser will now automatically access your private key.

*Note* The exact location of directory `/TorBrowser/Data/Tor/onion-auth` differs from computer to computer. It's best to search for it.
