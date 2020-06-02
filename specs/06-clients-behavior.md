
<!-- Section 6 --> <a id='S6'></a>

# Client behavior with walking onions

Today's Tor clients have several behaviors that become somewhat
more difficult to implement with Walking Onions.  Some of these
behaviors are essential and achievable.  Others can be achieved with
some effort, and still others appear to be incompatible with the
Walking Onions design.

<!-- Section 6.1 --> <a id='S6.1'></a>

## Bootstrapping and guard selection

When a client first starts running, it has no guards on the Tor
network, and therefore can't start building circuits immediately.
To produce a list of possible guards, the client begins connecting
to one or more fallback directories on their ORPorts, and building
circuits through them.  These are 3-hop circuits.  The first hop of
each circuit is the fallback directory; the second and third hops
are chosen from the Middle routing index.  At the third hop, the
client then sends an informational request for a guard's SNIP.  This
informational request is an EXTEND2 cell with handshake type NIL,
using a random spot on the Guard routing index.

Each such request yields a single SNIP that the client will store.
These SNIPs, in the order in which they were _requested_, will form the
client's list of "Sampled" guards as described in guard-spec.txt.

Clients SHOULD ensure that their sampled guards are not
linkable to one another.  In particular, clients SHOULD NOT add more
than one guard retrieved from the same third hop on the same
circuit. (If it did, that third hop would realize that some client using
guard A was also using guard B.)

> Future work: Is this threat real?  It seems to me that knowing one or two
> guards at a time in this way is not a big deal, though knowing the whole
> set would sure be bad.  However, we shouldn't optimize this kind of
> defense away until we know that it's actually needless.

If a client's network connection or choice of entry nodes is heavily
restricted, the client MAY request more than one guard at a time, but if
it does so, it SHOULD discard all but one guard retrieved from each set.

After choosing guards, clients will continue to use them even after
their SNIPs expire.  On the first circuit through each guard after
opening a channel, clients should ask that guard for a fresh SNIP for
itself, to ensure that the guard is still listed in the consensus, and
to keep the client's information up-to-date.

<!-- Section 6.2 --> <a id='S6.2'></a>

## Using bridges

As now, clients are configured to use a bridge by using an address and a
public key for the bridge.  Bridges behave like guards, except that they
are not listed in any directory or ENDIVE, and so cannot prove
membership when the client connects to them.

On the first circuit through each channel to a bridge, the client
asks that bridge for a SNIP listing itself in the `Self` routing
index.  The bridge responds with a self-created unsigned SNIP:

     ; This is only valid when received on an authenticated connection
     ; to a bridge.
     UnsignedSNIP = [
        ; There is no signature on this SNIP.
        auth : nil,

        ; Next comes the location of the SNIP within the ENDIVE.  This
        ; SNIPLocation will list only the Self index.
        index : bstr .cbor SNIPLocation,

        ; Finally comes the information about the router.
        router : bstr .cbor SNIPRouterData,
     ]

*Security note*: Clients MUST take care to keep UnsignedSNIPs separated
from signed ones. These are not part of any ENDIVE, and so should not be
used for any purpose other than connecting through the bridge that the
client has received them from.  They should be kept associated with that
bridge, and not used for any other, even if they contain other link
specifiers or keys.  The client MAY use link specifiers from the
UnsignedSNIP on future attempts to connect to the bridge.

<!-- Section 6.3 --> <a id='S6.3'></a>

## Finding relays by exit policy

To find a relay by exit policy, clients might choose the exit
routing index corresponding to the exit port they want to use.  This
has negative privacy implications, however, since the middle node
discovers what kind of exit traffic the client wants to use.
Instead, we support two other options.

First, clients may build anonymous three-hop circuits and then use those
circuits to request the SNIPs that they will use for their exits.  This
may, however, be inefficient.

Second, clients may build anonymous three-hop circuits and then use a
BEGIN cell to try to open the connection when they want.  When they do
so, they may include a new flag in the begin cell, "DVS" to enable
Delegated Verifiable Selection.  As described in the Walking Onions
paper, DVS allows a relay that doesn't support the requested port to
instead send the client the SNIP of a relay that does.  (In the paper,
the relay uses a digest of previous messages to decide which routing
index to use. Instead, we have the client send an index field.)

This requires changes to the BEGIN and END cell formats.  After the
"flags" field in BEGIN cells, we add an extension mechanism:

    struct begin_cell {
        nulterm addr_port;
        u32 flags;
        u8 n_extensions;
        struct extension exts[n_extensions];
    }

We allow the `snip_index_pos` link specifier type to appear as a begin
extension.

END cells will need to have a new format that supports including policy and
SNIP information.  This format is enabled whenever a new `EXTENDED_END_CELL`
flag appears in the begin cell.

    struct end_cell {
        u8 tag IN [ 0xff ]; // indicate that this isn't an old-style end cell.
        u8 reason;
        u8 n_extensions;
        struct extension exts[n_extensions];
    }

We define three END cell extensions.  Two types are for addresses, that
indicate what address was resolved and the associated TTL:

    struct end_ext_ipv4 {
        u32 addr;
        u32 ttl;
    }
    struct end_ext_ipv6 {
        u8 addr[16];
        u32 ttl;
    }

One new END cell extension is used for delegated verifiable selection:

    struct end_ext_alt_snip {
        u16 index_id;
        u8 snip[..];
    }

This design may require END cells to become wider; see
319-wide-everything.md for an example proposal to
supersede proposal 249 and allow more wide cell types.

<!-- Section 6.4 --> <a id='S6.4'></a>

## Universal path restrictions

There are some restrictions on Tor paths that all clients should obey,
unless they are configured not to do so.  Some of these restrictions
(like "start paths with a Guard node" or "don't use an Exit as a middle
when Exit bandwidth is scarce") are captured by the index system. Some
other restrictions are not.  Here we describe how to implement those.

The general approach taken here is "build and discard".  Since most
possible paths will not violate these universal restrictions, we
accept that a fraction of the paths built will not be usable.
Clients tear them down a short time after they are built.

Clients SHOULD discard a circuit if, after it has been built, they
find that it contains the same relay twice, or it contains more than
one relay from the same family or from the same subnet.

Clients MAY remember the SNIPs they have received, and use those
SNIPs to avoid index ranges that they would automatically reject.
Clients SHOULD NOT store any SNIP for longer than it is maximally
recent.

> NOTE: We should continue to monitor the fraction of paths that are
> rejected in this way.  If it grows too high, we either need to amend
> the path selection rules, or change authorities to e.g. forbid more
> than a certain fraction of relay weight in the same family or subnet.

> FUTURE WORK: It might be a good idea, if these restrictions truly are
> 'universal', for relays to have a way to say "You wouldn't want that
> SNIP; I am giving you the next one in sequence" and send back both
> SNIPs.  This would need some signaling in the EXTEND/EXTENDED cells.

<!-- Section 6.5 --> <a id='S6.5'></a>

## Client-configured path restrictions

Sometimes users configure their clients with path restrictions beyond
those that are in ordinary use.  For example, a user might want to enter
only from US relays, but never exit from US.  Or they might be
configured with a short list of vanguards to use in their second
position.

<!-- Section 6.5.1 --> <a id='S6.5.1'></a>

### Handling "light" restrictions

If a restriction only excludes a small number of relays, then clients
can continue to use the "build and discard" methodology described above.

<!-- Section 6.5.2 --> <a id='S6.5.2'></a>

### Handling some "heavy" restrictions

Some restrictions can exclude most relays, and still be reasonably easy
to implement if they only _include_ a small fraction of relays.  For
example, if the user has a EntryNodes restriction that contains only a
small group of relays by exact IP address, the client can connect or
extend to one of those addresses specifically.

If we decide IP ranges are important, that IP addresses without
ports are important, or that key specifications are important, we
can add routing indices that list relays by IP, by RSAId, or by
Ed25519 Id.  Clients could then use those indices to remotely
retrieve SNIPs, and then use those SNIPs to connect to their
selected relays.

> Future work: we need to decide how many of the above functions to actually
> support.

<!-- Section 6.5.3 --> <a id='S6.5.3'></a>

### Recognizing too-heavy restrictions

The above approaches do not handle all possible sets of restrictions. In
particular, they do a bad job for restrictions that ban a large fraction
of paths in a way that is not encodeable in the routing index system.

If there is substantial demand for such a path restriction, implementors
and authority operators should figure out how to implement it in the
index system if possible.

Implementations SHOULD track what fraction of otherwise valid circuits
they are closing because of the user's configuration.  If this fraction
is above a certain threshold, they SHOULD issue a warning; if it is
above some other threshold, they SHOULD refuse to build circuits
entirely.

> Future work: determine which fraction appears in practice, and use that to
> set the appropriate thresholds above.
