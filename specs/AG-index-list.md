
## Appendix G: A list of routing indices

Middle -- general purpose index for use when picking middle hops in
circuits.  Bandwidth-weighted for use as middle relays.  May exclude
guards and/or exits depending on overall balance of resources on the
network.

Guard -- index for choosing guard relays. This index is not used
directly when extending, but instead only for _picking_ guard relays
that the client will later connect to directly.  Bandwidth-weighted
for use as guard relays. May exclude guard+exit relays depending on
resource balance.

HSDirV2 -- index for finding spots on the hsv2 directory ring.

HSDirV3 -- index for finding spots on the hsv3 directory ring.

Self -- A virtual index that never appears in an ENDIVE.  SNIPs with
this index are unsigned, and occupy the entire index range.  This
index is used with bridges to represent each bridge's uniqueness.

> XXXX we also need an index for finding spots on the hsv3 directory
> ring from tomorrow/yesterday.  Not sure if this means three
> indices or two.

### Indices I am not sure we need

> XXXX this section should be empty or rewritten by the time I'm
> done

ipv4 index

ipv6 index

RSA-identity index

Ed25519-identity index


### Things that are not indices but from which relays are selected nontheless.

> XXXX remove this section or flesh it out.

A list of fallback directories.

A list of bridges
