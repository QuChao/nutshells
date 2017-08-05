# DNSCrypt-Wrapper Docker Image

[![Project Nutshells](https://img.shields.io/badge/Project-_Nutshells_🌰-orange.svg)](https://github.com/quchao/nutshells/) [![Docker Repo](https://img.shields.io/badge/Docker-Repo-22B8EB.svg)](https://hub.docker.com/r/nutshells/dnscrypt-wrapper/) [![Alpine Based](https://img.shields.io/badge/Alpine-3.6-0D597F.svg)](http://alpinelinux.org/) [![MIT License](https://img.shields.io/github/license/quchao/nutshells.svg?label=License)](https://github.com/quchao/nutshells/blob/master/LICENSE) [![dnscrypt-wrapper](https://img.shields.io/badge/DNSCrypt--Wrapper-0.3-lightgrey.svg)](https://github.com/cofyc/dnscrypt-wrapper/)

[DNSCrypt-Wrapper](https://github.com/cofyc/dnscrypt-wrapper/) is the server-end of [DNSCrypt](http://dnscrypt.org/) proxy, which is a protocol to improve DNS security, now with xchacha20 cipher support.


## Variants:

| Tag | Description | 🐳 |
|:-- |:-- |:--:|
| `:latest` | DNSCrypt-Wrapper `0.3` on `alpine:latest`, features certficate management & rotation. | [Dockerfile](https://github.com/QuChao/nutshells/blob/master/dnscrypt-wrapper/Dockerfile) |


## Usage

### Synopsis

```
docker run [OPTIONS] nutshells/dnscrypt-wrapper [COMMAND] [ARG...]
```

> Learn more about `docker run` [here](https://docs.docker.com/engine/reference/commandline/run/).

### Getting Started

A DNScrypt proxy server cannot go without a so-called **provider key pair**.
It's rather easy to generate a new pair by running the `init` [command](#commands) as below:

> `<keys_dir>` is a host directory where you store the key pairs.
> The provider key pair should NEVER be changed as you may inform the world of the public key, unless the secret key is compromised.
> The `--net=host` option provides the best network performance. Use it if you know it exactly.

``` bash
docker run -d -p 5353:12345/udp -p 5353:12345/tcp \
           --name=dnscrypt-server --restart=unless-stopped --read-only \
           --mount=type=bind,src=<keys_dir>,dst=/usr/local/etc/dnscrypt-wrapper \
           nutshells/dnscrypt-wrapper \
           init
```

Now, a server is initialized with [default settings](#environment_variables) and running on port `5353` as a daemon.

Dig the *public key* fingerprint out of logs:

``` bash
docker logs dnscrypt-server | grep --color 'Provider public key: '
```

Then [see if it works](#how_to_test).

### Using an existing key pair

If you used to run a DNScrypt proxy server and have been keeping the keys securely, make sure they are put into `<keys_dir>` and renamed to `public.key` and `secret.key`, then run the `start` [command](#commands) instead.

> Incidentally, `start` is the default one which could be just omitted.

Lost the *public key* fingerprint but couldn't find it from logs? Try the `pubkey` command:

``` bash
docker run --rm --read-only \
           --mount=type=bind,src=<keys_dir>,dst=/usr/local/etc/dnscrypt-wrapper \
           nutshells/dnscrypt-wrapper \
           pubkey
```

### How to test

Get `<provider_pub_key>` by following the instructions above,
and check [this section](#environment_variables) to determin the default value of `<provider_basename>`.

> Please install [`dnscrypt-proxy`](https://hub.docker.com/r/nutshells/dnscrypt-proxy/) and `dig` first.

``` bash
dnscrypt-proxy --local-address=127.0.0.1:53 \
               --resolver-address=127.0.0.1:5353 \
               --provider-name=2.dnscrypt-cert.<provider_basename> \
               --provider-key=<provider_pub_key>
dig -p 53 +tcp google.com @127.0.0.1
```

### Utilities

Check the version:

``` bash
docker run --rm --read-only nutshells/dnscrypt-wrapper --version
```

Print its original options :

> Please be informed that **some** of the listed options are managed by the container intentionally, you will encounter an exception while trying to set any of them, please follow the exception message to get rid of it.
> If you do want to change the options, use these [environment variables](#environment_variables) instead.

``` bash
docker run --rm --read-only nutshells/dnscrypt-wrapper --help
```


## Reference

### Environment Variables

Since some certain options of `dnscrypt-wrapper` will be handled by [the entrypoint script](https://github.com/quchao/nutshells/blob/master/dnscrypt-wrapper/docker-entrypoint.sh) of the container, you can *ONLY* customize them by setting the environment variables below:

| Name | Default | Relevant Option | Description |
|:-- |:-- |:-- |:-- |
| `RESOLVER_IP` | `8.8.8.8` | `-r`, `--resolver-address` | Upstream dns resolver server IP |
| `RESOLVER_PORT` | `53` | `-r`, `--resolver-address` | Upstream dns resolver server port |
| `PROVIDER_BASENAME` | `example.com` | `--provider-name` | Basename of the provider, which forms the whole provide name with a prefix `2.dnscrypt-cert.` |
| `CRYPT_KEYS_LIFESPAN` | `365` | `--cert-file-expire-days` | For how long (in days) the crypt key & certs would be valid. Refer to [this topic](#rotating_the_crypt_key_and_certs) to automate the rotation. |

For instance, if you want to use [OpenDNS](https://www.opendns.com) as the upstream DNS resolver other than [Google's Public DNS](https://developers.google.com/speed/public-dns/), the default one, just [set an environment variable](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e-env-env-file) like this:

```
docker run -e RESOLVER_IP=208.67.222.222 -e RESOLVER_PORT=5353 ...
```

### Data Volumes

| Container Path | Description | Writable |
|:-- |:-- |:--:|
| `/usr/local/etc/dnscrypt-wrapper` | Directory where keys are stored | Y |

### Commands

List available commands:

```
docker run --rm --read-only nutshells/dnscrypt-wrapper help
```


## Advanced Topics

### Using Docker Compose

See [the sample file](https://github.com/quchao/nutshells/blob/master/dnscrypt-wrapper/docker-compose.yml).

### Backing-up the secret key

If you forgot to mount `<keys_dir>` into the container,
and now you want to locate the secret key in the anonymous volume,
just do some inspection first:

``` bash
docker inspect -f '{{json .Mounts }}' dnscrypt-server | grep --color '"Source":'
```

Then backup it securely.

### Rotating the crypt key and certs

Unlike the lifelong provider key pair, a **crypt key** & two certs, which are time-limited and used to encrypt and authenticate DNS queries, will be generated only if they're missing or expiring on starting. Thus the container is supposed to be restarted before certs' expiration.

> Two certs are issued right after the crypt key's generation, one of them uses xchacha20 cipher.

Let's say we're planning to rotate them about once a week.
Firstly, shrink [the cert's lifespan](#environment_variables) to `7` days:

> Actually the rotation starts when the validity remaining is under `30%`, which would be on day `5` in this case.

```
docker run -e CRYPT_KEYS_LIFESPAN=7 ...
```

Secondly, restart the container every single day by creating a daily cronjob:

``` bash
0 4 * * * docker restart dnscrypt-server
```

### Gaining a shell access

Get an interactive shell to the container by overwritting the default entrypoint:

``` bash
docker run -it --rm --entrypoint=/bin/ash nutshells/dnscrypt-wrapper
```

### Customizing the image

#### By modifying the dockerfile

You may want to make some modifications to the image.
Pull the source code from GitHub, customize it, then build one by yourself:

``` bash
git clone --depth 1 https://github.com/quchao/nutshells.git
docker build -q=false --rm=true --no-cache=true \
             -t nutshells/dnscrypt-wrapper \
             -f ./dnscrypt-wrapper/Dockerfile \
             ./dnscrypt-wrapper
```

#### By committing the changes on a container

Otherwise just pull the image from the official registry, start a container and [get a shell](#gaining_a_shell_access) to it, [commit the changes](https://docs.docker.com/engine/reference/commandline/commit/) afterwards.

``` bash
docker pull nutshells/dnscrypt-wrapper
docker run -it --name=dnscrypt-server --entrypoint=/bin/ash nutshells/dnscrypt-wrapper
docker commit --change "Commit msg" dnscrypt-server nutshells/dnscrypt-wrapper
```


## Caveats

## Declaring the Health Status

Status of this container-specified health check merely indicates whether the crypt certs are *about to expire*, you'd better **restart** the container to [rotate the keys](key-rotation) ASAP if it's shown as *unhealthy*.

To confirm the status, run this command:

``` bash
docker inspect --format='{{json .State.Health.Status}}' dnscrypt-server
```

And to check the logs:

``` bash
docker inspect --format='{{json .State.Health}}' dnscrypt-server | python -m json.tool
```

If you think this is annoying, just add [the `--no-healthcheck` option](https://docs.docker.com/engine/reference/run/#healthcheck) to disable it.


## Contributing

> Follow GitHub's [*How-to*](https://opensource.guide/how-to-contribute/) guide for the basis.

Contributions are always welcome in many ways:

- Give a star to show your fondness;
- File an [issue](https://github.com/quchao/nutshells/issues) if you have a question or an idea;
- Fork this repo and submit a [PR](https://github.com/quchao/nutshells/pulls);
- Improve the documentation.


## Todo

- [x] Serve with the old key & certs for another hour after the rotation.
- [ ] Add instructions on how to speed it up by caching the upstream dns queries.
- [x] Add a `HealthCheck` instruction to indicate the expiration status of certs.
- [ ] Add a command for checking the expire status.
- [ ] Use another container to rotate the keys.


## Acknowledgments & Licenses

Unless specified, all codes of **Project Nutshells** are released under the [MIT License](https://github.com/quchao/nutshells/blob/master/LICENSE).

Other relevant softwares:

| Ware/Lib | License |
|:-- |:--:|
| [Docker](https://www.docker.com/) | [Apache 2.0](https://github.com/moby/moby/blob/master/LICENSE) |
| [DNSCrypt-Proxy](https://github.com/jedisct1/dnscrypt-proxy) | [ISC](https://github.com/jedisct1/dnscrypt-proxy/blob/master/COPYING) |
| [DNSCrypt-Wrapper](https://github.com/cofyc/dnscrypt-wrapper/) | [ISC](https://github.com/cofyc/dnscrypt-wrapper/blob/master/COPYING) |
| [DNSCrypt-Server-Docker](https://github.com/jedisct1/dnscrypt-server-docker/) | [ISC](https://github.com/jedisct1/dnscrypt-server-docker/blob/master/LICENSE) |