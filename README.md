# ZeroNS: a name service centered around the ZeroTier Central API

ZeroNS provides names that are a part of [ZeroTier Central's](https://my.zerotier.com) configured _networks_; once provided an IPv4-capable network it:

- Listens on the local interface joined to that network -- you will want to start one ZeroNS per ZeroTier network.
- Provides general DNS by forwarding all queries to `/etc/resolv.conf` resolvers that do not match the TLD, similar to `dnsmasq`.
- Tells Central to point all clients that have the "Manage DNS" settings turned **on** to resolve to it.
- Finally, sets a provided TLD (`.home.arpa` is the default; recommended by IANA), as well as configuring `A` (IPv4) and `AAAA` (IPv6) records for:
  - Member IDs: `zt-<memberid>.<tld>` will resolve to the IPv4 & IPv6 addresses for them.
  - Names: _if_ the names are compatible with DNS names, they will be converted as such: to `<name>.<tld>`.
    - Please note that **collisions are possible** and that it's _up to the admin to prevent them_.
  - It additionally includes PTR records for members, in all scenarios other than 6plane.
  - _Wildcard everything mode_: this mode (enabled by passing the `-w` flag) enables wildcards for all names under the TLD; for example `my-site.zt-<memberid>.<tld>` will resolve to the member's IP, and named hosts work the same way.

## Installation

Before continuing, be reminded that zeronsd is **beta software**. That said, if you'd like to get started quickly with zeronsd, [click here for a user-friendly guide](docs/quickstart.md)!

Packages:

- Linux/Windows: [releases](https://github.com/zerotier/zeronsd/releases) contain packages for `*.deb`, `*.rpm` for Linux, and MSI format for Windows. **NOTE**: the Windows MSI will install a firewall exception for port 53 so zeronsd can communicate.
  - [Arch Linux](https://aur.archlinux.org/packages/zeronsd/) packages provided by [@devvick](https://github.com/devvick)!
- Mac OS X: `brew tap zerotier/homebrew-tap && brew install zerotier/homebrew-tap/zeronsd`
- Docker: `docker pull zerotier/zeronsd` (see below for more on docker)

Other methods:

### Get a release from Cargo

Please obtain a working [rust environment](https://rustup.rs/) first.

```
cargo install zeronsd
```

### From Git (via Cargo)

```
cargo install --git https://github.com/zerotier/zeronsd --branch main
```

### Docker

There is a `Dockerfile` present in the repository you can use to build images in lieu of one of our [official images](https://hub.docker.com/r/zerotier/zeronsd).

There are build arguments which control behavior:

- `IS_LOCAL`: if set, uses the local source tree and does not try to fetch.
- `VERSION`: this is the branch or tag to fetch.
- `IS_TAG`: if non-zero, tells cargo to fetch tags instead of branches.

Example:

```bash
docker build . # builds latest master
docker build --build-arg VERSION=somebranch # builds branch `somebranch`
docker build --build-arg IS_TAG=1 --build-arg VERSION=v0.1.0 # builds version 0.1.0 from tag v0.1.0
```

Once built, the image automatically runs `zeronsd` for you. The default subcommand is `help`.

### Docker (alpine edition)

See [Dockerfile.alpine](Dockerfile.alpine).

## Usage

Setting `ZEROTIER_CENTRAL_TOKEN` in the environment (or providing the `-t` flag, which points at a file containing this value) is required. You must be able to administer the ZeroTier network to use `zeronsd` with it. Also, running as `root` is required as _many client resolvers do not work over anything but port 53_. Your `zeronsd` instance will listen on both `udp` and `tcp`, port `53`.

### Bare commandline

**Tip**: running `sudo`? Pass the `-E` flag to import your current shell's environment, making it easier to add the `ZEROTIER_CENTRAL_TOKEN`, or use the `-t` flag to avoid the environment entirely.

```
zeronsd start <network id>
```

#### Configuration

zeronsd as of v0.3 takes a configuration file via the `-c` flag which correlates to all of the command-line options. `--config-type` corresponds to the format of the configuration file: `yaml` is the default, and `json` and `toml` are also supported.

The configuration directives are as follows:

- domain: (string) will set a TLD for your records; the default is `home.arpa`.
- hosts: (string) will parse a file in `/etc/hosts` format and append it to your records.
- secret: (string) path to `authtoken.secret` which is needed to talk to ZeroTier on localhost. You can provide this file with this argument, but it is auto-detected on multiple platforms including Linux, OS X and Windows.
- token: (string) path to file containing your [ZeroTier Central token](https://my.zerotier.com/account).
- wildcard: (bool) Enables wildcard mode, where all member names get a wildcard in this format: `*.<name>.<tld>`; this points at the member's IP address(es).

### Running as a service

_This behavior is currently only supported on Linux and Mac OS X; we will accept patches for other platforms._

The `zeronsd supervise` and `zeronsd unsupervise` commands can be used to manipulate systemd unit files related to your network. For the `supervise` case, simply pass the arguments you would normally pass to `start` and it will generate a unit from it.

Example:

```bash
# to enable
zeronsd supervise -t ~/.token -f /etc/hosts -d mydomain 36579ad8f6a82ad3
# generates systemd unit file named /lib/systemd/system/zeronsd-36579ad8f6a82ad3.service
systemctl daemon-reload
systemctl enable zeronsd-36579ad8f6a82ad3.service && systemctl start zeronsd-36579ad8f6a82ad3.service

# to disable
systemctl disable zeronsd-36579ad8f6a82ad3.service && systemctl stop zeronsd-36579ad8f6a82ad3.service
zeronsd unsupervise 36579ad8f6a82ad3
systemctl daemon-reload
```

### Logging

Set `ZERONSD_LOG` or `RUST_LOG` to various log levels or other parameters according to the [env_logger](https://crates.io/crates/env_logger) specification for more.

### Docker

Running in docker is a little more complicated. You must be able to have a network interface you can import (joined a network) and must be able to reach `localhost:9999` on the host. At this time, for brevity's sake we are recommending running with `--net=host` until we have more time to investigate a potentially more secure solution.

You also need to mount your `authtoken.secret`, which we use to talk to `zerotier-one`

```
docker run --net host -it \
  -v /var/lib/zerotier-one/authtoken.secret:/authtoken.secret \
  -v <token file>:/token.txt \
  zeronsd:alpine start -s /authtoken.secret -t /token.txt \
  <network id>
```

### Other notes

You must have already joined a network and obviously, `zerotier-one` should be running!

It should print some diagnostics after it has talked to your `zerotier-one` instance to figure out what IP to listen on. After that it should communicate with the central API and set everything else up automatically.

### Flags for the `start` and `supervise` subcommands:

- `-d <tld>` will set a TLD for your records; the default is `home.arpa`.
- `-f <hosts file>` will parse a file in `/etc/hosts` format and append it to your records.
- `-s <secret file>` path to `authtoken.secret` which is needed to talk to ZeroTier on localhost. You can provide this file with this argument, but it is auto-detected on multiple platforms including Linux, OS X and Windows.
- `-t <central token file>` path to file containing your [ZeroTier Central token](https://my.zerotier.com/account).
- `-w` Enables wildcard mode, where all member names get a wildcard in this format: `*.<name>.<tld>`; this points at the member's IP address(es).
- `-v` Enables verbose logging. Repeat for more verbosity.
- `-V` prints the version.

### TTLs

Records currently have a TTL of 60s, and Central's records are refreshed every 30s through the API. I felt this was a safer bet than letting timeouts happen.

### Per-Interface DNS resolution

OS X and Windows users get this functionality by default, so there is no need for it. Please note at this point in time, however, that PTR resolution does not properly work on either platform. This is a defect in ZeroTier and should be corrected soon.

Linux users are strongly encouraged to use `systemd-networkd` along with `systemd-resolved` to get per-interface resolvers that you can isolate to the domain you want to use. If you'd like to try something that can assist with getting you going quickly, check out the [zerotier-systemd-manager repository](https://github.com/zerotier/zerotier-systemd-manager).

BSD systems still need a bit of work; work that we could really use your help with if you know the lay of the land on your BSD of choice. Set up an issue if this interests you.

## Acknowledgements

ZeroNS demands a lot out of the [trust-dns](https://github.com/bluejekyll/trust-dns) toolkit and I personally am grateful such a library suite exists. It made my job very easy.

## License

[BSD 3-Clause](https://github.com/zerotier/zeronsd/blob/main/LICENSE)

## Author

Erik Hollensbe <github@hollensbe.org>
