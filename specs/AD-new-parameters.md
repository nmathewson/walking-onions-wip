
<!-- Section A.4 --> <a id='SA.4'></a>

## Appendix D: New network parameters.

We introduce these network parameters:

From section 5:

* `create-pad-len` -- Clients SHOULD pad their CREATE cell bodies
  to this size.

* `created-pad-len` -- Relays SHOULD pad their CREATED cell bodies to this
  size.

* `extend-pad-len` -- Clients SHOULD pad their EXTEND cell bodies to this
  size.

* `extended-pad-len` -- Relays SHOULD pad their EXTENDED cell bodies to this
size.

From section 7:

* `hsv2-index-bytes` -- how many bytes to use when sending an hsv2 index
  position to look up a hidden service directory.  Min: 1,
  Max: 40. Default: 4.

* `hsv3-index-bytes` -- how many bytes to use when sending an hsv3 index
  position to look up a hidden service directory.  Min: 1,
  Max: 128. Default: 4.

* `hsv3-intro-legacy-fields` -- include legacy fields in service descriptors.
  Min: 0. Max: 1. Default: 1.

* `hsv3-intro-snip` -- include intro point SNIPs in service descriptors.
  Min: 0. Max: 1. Default: 0.

* `hsv3-rend-service-snip` -- Should services advertise and accept rendezvous
  point SNIPs in INTRODUCE2 cells?    Min: 0. Max: 1. Default: 0.

* `hsv3-rend-client-snip` -- Should clients place rendezvous point SNIPS in
  INTRODUCE2 cells when the service supports it?
  Min: 0. Max: 1. Default: 0.

* `hsv3-tolerate-no-legacy` -- Should clients tolerate v3 service descriptors
  that don't have legacy fields? Min: 0. Max: 1. Default: 0.

From section 8:

* `enforce-endive-dl-delay-after` -- How many seconds after receiving a
  SNIP with some timestamp T does a client wait for rejecting older SNIPs?
  Equivalent to "N" in "limiting ENDIVE variance within the network."
  Min: 0. Max: INT32_MAX. Default: 3600 (1 hour).

* `allow-endive-dl-delay` -- Once a client has received an SNIP with
  timestamp T, it will not accept any SNIP with timestamp earlier than
  "allow-endive-dl-delay" seconds before T.
  Equivalent to "Delta" in "limiting ENDIVE variance within the network."
  Min: 0. Max: 2592000 (30 days). Default: 10800 (3 hours).
