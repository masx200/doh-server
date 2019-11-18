# doh-proxy

A fast and secure DoH (DNS-over-HTTPS) server written in Rust.

## Installation

Without built-in support for HTTPS:

```sh
cargo install doh-proxy
```

With built-in support for HTTPS (requires openssl-dev):

```sh
cargo install doh-proxy --features=tls
```

## Usage

```text
A DNS-over-HTTP server proxy

USAGE:
    doh-proxy [FLAGS] [OPTIONS]

FLAGS:
    -K, --disable-keepalive    Disable keepalive
    -P, --disable-post         Disable POST queries
    -h, --help                 Prints help information
    -V, --version              Prints version information

OPTIONS:
    -E, --err-ttl <err_ttl>                          TTL for errors, in seconds [default: 2]
    -l, --listen-address <listen_address>            Address to listen to [default: 127.0.0.1:3000]
    -b, --local-bind-address <local_bind_address>    Address to connect from [default: 0.0.0.0:0]
    -c, --max-clients <max_clients>                  Maximum number of simultaneous clients [default: 512]
    -X, --max-ttl <max_ttl>                          Maximum TTL, in seconds [default: 604800]
    -T, --min-ttl <min_ttl>                          Minimum TTL, in seconds [default: 10]
    -p, --path <path>                                URI path [default: /dns-query]
    -u, --server-address <server_address>            Address to connect to [default: 9.9.9.9:53]
    -t, --timeout <timeout>                          Timeout, in seconds [default: 10]
    -I, --tls-cert-password <tls_cert_password>
            Password for the PKCS12-encoded identity (only required for built-in TLS)

    -i, --tls-cert-path <tls_cert_path>              Path to a PKCS12-encoded identity (only required for built-in TLS)
```

## HTTP/2 termination

The recommended way to use `doh-proxy` is to use a TLS termination proxy (such as [hitch](https://github.com/varnish/hitch) or [relayd](https://bsd.plumbing/about.html)), a CDN or a web server with proxying abilities as a front-end.

That way, the DoH service can be exposed as a virtual host, sharing the same IP addresses as existing websites.

If `doh-proxy` and the HTTP/2 front-end run on the same host, using the HTTP protocol to communicate between both is fine.

If both are on distinct networks, such as when using a CDN, `doh-proxy` can handle HTTPS requests, provided that it was compiled with the `tls` feature.

The identity must be encoded in PKCS12 format. Given an existing certificate `cert.pem` and its secret key `cert.key`, this can be achieved using the `openssl` command-line tool:

```sh
openssl pkcs12 -export -out cert.p12 -in cert.pem -inkey cert.key
```

A password will be interactive asked for, but the `-passout` command-line option can be added to provide it non-interactively.

Once done, check that the permissions on `cert.p12` are reasonable.

In order to enable built-in HTTPS support, add the `--tls-cert-path` option to specify the location of the `cert.p12` file, as well as the password using `--tls-cert-password`.

Once HTTPS is enabled, HTTP connections will not be accepted.

## Accepting both DNSCrypt and DoH connections on port 443

DNSCrypt is an alternative encrypted DNS protocol that is faster and more lightweight than DoH.

Both DNSCrypt and DoH connections can be accepted on the same TCP port using [Encrypted DNS Server](https://github.com/jedisct1/encrypted-dns-server).

Encrypted DNS Server forwards DoH queries to Nginx or `rust-doh` when a TLS connection is detected, or directly responds to DNSCrypt queries.

It also provides DNS caching, server-side filtering, metrics, and TCP connection reuse in order to mitigate exhaustion attacks.

Unless the front-end is a CDN, an ideal setup is to use `rust-doh` behind `Encrypted DNS Server`.

## Operational recommendations

* DoH can be easily detected and blocked using SNI inspection. As a mitigation, DoH endpoints should preferably share the same virtual host as existing, popular websites, rather than being on dedicated virtual hosts.
* When using DoH, DNS stamps should include a resolver IP address in order to remove a dependency on non-encrypted, non-authenticated, easy-to-block resolvers.
* Unlike DNSCrypt where users must explicitly trust a DNS server's public key, the security of DoH relies on traditional public Certificate Authorities. Additional root certificates (required by governments, security software, enterprise gateways) installed on a client immediately make DoH vulnerable to MITM. In order to prevent this, DNS stamps should include the hash of the parent certificate.
* TLS certificates are tied to host names. But domains expire, get reassigned and switch hands all the time. If a domain originally used for a DoH service gets a new, possibly malicious owner, clients still configured to use the service will blindly keep trusting it if the CA is the same. As a mitigation, the CA should sign an intermediate certificate (the only one present in the stamp), itself used to sign the name used by the DoH server. While commercial CAs offer this, Let's Encrypt currently doesn't.
* Make sure that the front-end supports HTTP/2 and TLS 1.3.
* Internal DoH servers still require TLS certificates. So, if you are planning to deploy an internal server, you need to set up an internal CA, or add self-signed certificates to every single client.

## Example usage with `encrypted-dns-server`

Add the following section to the configuration file:

```toml
[tls]
upstream_addr = "127.0.0.1:3000"
```

## Example usage with `nginx`

In an existing `server`, a `/doh` endpoint can be exposed that way:

```text
location /doh {
  proxy_pass http://127.0.0.1:3000;
}
```

This example assumes that the DoH proxy is listening locally to port `3000`.

HTTP caching can be added (see the `proxy_cache_path` and `proxy_cache` directives in the Nginx documentation), but be aware that a DoH server will quickly create a gigantic amount of files.

Use the online [DNS stamp calculator](https://dnscrypt.info/stamps/) to compute the stamp for your server, and the `dnscrypt-proxy -show-certs` command to print the TLS certificate signatures to be added it.

## Clients

`doh-proxy` can be used with [dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy)
as a client.

`doh-proxy` is currently being used by the `doh.crypto.sx` public DNS resolver.

An extensive list of public DoH servers can be found here: [public encrypted DNS servers](https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v2/public-resolvers.md).
