
<!-- Section 7 --> <a id='S7'></a>

# Using and providing onion services with Walking Onions

Both live versions of the onion service design rely on a ring of
hidden service directories for use in uploading and downloading
hidden service descriptors.  With Walking Onions, we can use routing
indices based on Ed25519 or RSA identity keys to retrieve this data.

(The RSA identity ring is unchanging, whereas the Ed25519 ring
changes daily based on the shared random value: for this reason, we
have to compute two simultaneous indices for Ed25519 rings: one for
the earlier date that is potentially valid, and one for the later
date that is potentially valid. We call these `hsv3-early` and
`hsv3-late`.)

Beyond the use of these indices, however, there are other steps that
clients and services need to take in order to maintain their privacy.

<!-- Section 7.1 --> <a id='S7.1'></a>

## Finding HSDirs

When a client or service wants to contact an HSDir, it SHOULD do so
anonymously, by building a three-hop anonymous circuit, and then
extending it a further hop using the snip_span link specifier to
upload to any of the first 3 replicas on the ring.  Clients SHOULD
choose an 'nth' at random; services SHOULD upload to each replica.

Using a full 80-bit or 256-bit index position in the link specifier
would leak the chosen service to somebody other than the directory.
Instead, the client or service SHOULD truncate the identifier to a
number of bytes equal to the network parameter `hsv2-index-bytes` or
`hsv3-index-bytes` respectively.  (See Appendix C.)

<!-- Section 7.2 --> <a id='S7.2'></a>

## SNIPs for introduction points

When services select an introduction point, they should include the
SNIP for the introduction point in their hidden service directory
entry, along with the introduction-point fields.  The format for
this entry is:

    "snip" NL snip NL
      [at most once per introduction points]

Clients SHOULD begin treating the link specifier and onion-key
fields of each introduction point as optional when the "snip" field
is present, and when the `hsv3-tolerate-no-legacy` network parameter
is set to 1. If either of these fields _is_ present, and the SNIP is
too, then these fields MUST match those listed in the SNIPs.
Clients SHOULD reject descriptors with mismatched fields, and alert
the user that the service may be trying a partitioning attack.
The "legacy-key" and "legacy-key-cert" fields, if present, should be
checked similarly.

> Using the SNIPs in these ways allows services to prove that their
> introduction points have actually been listed in the consensus
> recently.  It also lets clients use introduction point features
> that the relay might not understand.

Services should include these fields based on a set of network
parameters: `hsv3-intro-snip` and `hsv3-intro-legacy-fields`.
(See appendix C.)

Clients should use these fields only when Walking Onions support is
enabled; see section 09.

<!-- Section 7.3 --> <a id='S7.3'></a>

## SNIPs for rendezvous points

When a client chooses a rendezvous point for a v3 onion service, it
similarly has the opportunity to include the SNIP of its rendezvous
point in the encrypted part of its INTRODUCE cell.  (This may cause
INTRODUCE cells to become fragmented; see proposal about fragmenting
relay cells.)

> Using the SNIPs in these ways allows services to prove that their
> introduction points have actually been listed in the consensus
> recently.  It also lets services use introduction point features
> that the relay might not understand.

To include the SNIP, the client places it in an extension in the
INTRODUCE cell.  The onion key can now be omitted[*], along with
the link specifiers.

> [*] Technically, we use a zero-length onion key, with a new type
> "implicit in SNIP".

To know whether the service can recognize this kind of cell, the
client should look for the presence of a "snips-allowed 1" field in
the encrypted part of the hidden service descriptor.

In order to prevent partitioning, services SHOULD NOT advertise
"snips-allowed 1" unless the network parameter
"hsv3-rend-service-snip" is set to 1.  Clients SHOULD NOT use this
field unless "hsv3-rend-client-snip" is set to 1.

<!-- Section 7.4 --> <a id='S7.4'></a>

## TAP keys and where to find them

If v2 hidden services are still supported when Walking Onions arrives
on the network, we have two choices:  We could migrate them to use
ntor keys instead of TAP, or we could provide a way for TAP keys to
be advertised with Walking Onions.

The first option would appear to be far simpler. See
proposal draft 320-tap-out-again.md.

The latter option would require us to put RSA-1024 keys in SNIPs, or
put a digest of them in SNIPs and give some way to retrieve them
independently.

(Of course, it's possible that we will have v2 onion services
deprecated by the time Walking Onions is implemented.  If so, that
will simplify matters a great deal too.)

