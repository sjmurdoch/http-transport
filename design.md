<!-- Keep lines to 80 columns
01234567890123456789012345678901234567890123456789012345678901234567890123456789
        10        20        30        40        50        60        70        80
Format with: fmt -p -l 0 -w 80
-->

# Tor over HTTP

## Goals and threat model

### Common goals

 * To a passive adversary, the communications is not efficiently and reliably
   distinguishable from a user browsing the web connecting with a commonly used
   web browser to a commonly used web browser
   
 * Any tampering of the communication between bridge client and server, which is
   within the HTTP specification of what a proxy is permitted to do, or which
   commonly used proxies do, should not alter the data that Tor is sending to or
   receiving from the pluggable transport
   
 * Any other tampering should result in the connection being torn down (perhaps
   simply by sending corrupt data to Tor, which will be detected by TLS)

### Unauthenticated mode

 * User and bridge operator do not share a secret (user knows port and IP of
   bridge)
   
 * It is unavoidable that someone connecting to the bridge over HTTP and
   attempting a handshake will discover that it is a Tor bridge

### Authenticated mode

 * User and bridge share a cryptographically secure secret (minimum 128 bits
   long)

 * To someone scanning candidate IP addresses to check whether there is a Tor
   bridge running, the bridge is not efficiently and reliably distinguishable
   from a commonly used web server, unless the secret is known. The attacker is
   permitted to have observed previous traffic between the bridge and clients
   which do know the secret

# Implementation components

## Disguising patterns in Tor traffic

Tor traffic sent to the bridge is a TLS connection, which contains some patterns
that make it easily distinguishable from random data. Therefore the first step
of obfuscating traffic will be to hide these patterns. There are a few options
for doing this:

### Session key establishment

#### UniformDH

As used by [obfs3][obfs3], the [UniformDH][uniformdh] scheme negotiates a
session key using anonymous Diffie-Hellman but with public keys which are not
distinguishable from random numbers

#### Elligator

[Elligator][elligator] is a technique for sending elliptic curve points in such
a way that they are indistinguishable from random. Therefore it can be used for
elliptic curve Diffie-Hellman as well as for other types of elliptic curve
cryptographic operations

### Bulk encryption and authentication

Public key cryptography has the significant advantage that it can resist
eavesdropping by a passive adversary even when the two communication partners do
not share a key. However it is quite slow, so most of the communication should
be encrypted under a symmetric algorithm. Here, there are plenty to choose from,
but AES-128 counter mode is fast and secure for most purposes.

One more interesting family that may be worth considering (possibly in
combination with encryption) is [high-inertia scrambling][inertia]. This is a
function which is reversible but hard to accelerate therefore forcing the censor
to expend a very large amount of computation power to discover the content being
transferred.

## Scanning resistance

## Splitting, re-assembly and error detection

Both Tor and HTTP both use TCP but in different (and incompatible ways). Tor
maintains one TCP long-lived connection between client and bridge and requires
that data be delivered in-order and reliably. In contrast, modern browsers
create multiple simultaneous TCP streams, which are moderately short-lived.
Each TCP stream may be used for multiple requests.

In a single TCP stream, data is delivered in-order but if data item A is sent
over stream A, before data item B is sent over stream B, then it is possible
that data item B is received before data item A. To be plausibly
indistinguishable from HTTP, data will have to be split into chunks and sent
over multiple TCP streams and then reassembled in the correct order.

In some cases HTTP may also truncate data, because TCP does not [reliably inform
higher layers][linger] that a connection has been closed prematurely.  Therefore
there will need to be some detection of truncated resources, and protocol for
re-transmission. This could be implemented at the HTTP layer with chunked-mode
but proxy servers may not support chunked mode.

## Bi-directional data

Tor requires that communication exchanges be initiated either by the bridge
client or bridge server. In contrast HTTP clients initiate all communications. There are a few ways to avoid this problem:

 *  The client periodically polls the server to check if any data is available 
  
 *  The client keeps a long-running [Comet][comet] TCP connection, on which the server can send responses
  
 *  The client and server both act as HTTP clients and HTTP servers, so can each send data when they wish

## Proxy busting

Proxies will, under certain conditions, not send a request they receive to the destination server, but instead serve whatever the proxy thinks is the correct response. The HTTP specification dictates a proxy's behaviour but some proxy servers may deviate from the requirements. The pluggable transport will therefore need to either prevent the proxy from
caching responses or detect cached data and trigger a re-transmission. It may be unusual behaviour for a HTTP client to always send unique requests, so it perhaps should occasionally send dummy requests which are the same as before and so would be cached.

## Packet-size obfuscation

Tor sends 512 bytes cells and so results in very distinctive TLS Application Record sizes. It may be necessary to [alter packet length][morpher] to hide such patterns.

## Encoding

Once data has been split, coded for error detection, and its size obfuscated, it must be encoded as a HTTP request or response. With HTTP, data is generally sent from server to client, so client to server channels are problematic. For this reason implementing a HTTP server on both the pluggable transport client and pluggable transport server is desirable, but this approach stands out in other ways.

Possible channels for encoding data are:

### Client to server (requests)

#### Cookies

Short and usually do not change, so possibly not a good choice

#### HTTP POST file uploads

Quite unusual, but permit large uploads

### Server to client (responses)

#### HTTP GET file downloads

Very common, so probably a good choice

### File types

For both HTTP POST and HTTP GET, a file format must be chosen into which data is encoded. Some possibilities are:

#### Images

Both PNG and JPEG images are very common on the Internet, and do contain high-entropy data so are good candidates for encoding Tor payloads. However, a more sophisticated adversary may try to use a PNG or JPEG decoder to check whether the file is valid or not, requiring a more sophisticated encoder to defeat.

#### HTML/Javascript/CSS

It would be possible to encode data as HTML or Javascript, for example using [Format-Transforming Encryption][fte]. Such content would look implausible to the eye but could be made to pass most automated checks.

# Related work

[ScrambleSuit][scramblesuit] implements scanning resistance, packet-size obfuscation, session-key establishment and bulk encryption, so may be a useful basis for further work.

<!-- References -->

[uniformdh]:https://lists.torproject.org/pipermail/tor-dev/2012-December/004245.html

[obfs3]:https://gitweb.torproject.org/pluggable-transports/obfsproxy.git/blob/HEAD:/doc/obfs3/obfs3-protocol-spec.txt

[elligator]:http://eprint.iacr.org/2013/325.pdf

[inertia]:http://dx.doi.org/10.1007/978-3-642-25867-1_28

[linger]:http://blog.netherlabs.nl/articles/2009/01/18/the-ultimate-so_linger-page-or-why-is-my-tcp-not-reliable

[comet]:http://en.wikipedia.org/wiki/Comet_(programming)

[morpher]:https://research.torproject.org/techreports/morpher-2012-03-13.pdf

[scramblesuit]:http://www.cs.kau.se/philwint/scramblesuit/

[fte]:http://eprint.iacr.org/2012/494
