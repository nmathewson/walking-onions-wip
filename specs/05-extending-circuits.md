
<!-- Section 5 --> <a id='S5'></a>

# Extending circuits with Walking Onions

When a client wants to extend a circuit, there are several
possibilities.  It might need to extend to an unknown relay with
specific properties.  It might need to extend to a particular relay
from which it has received a SNIP before.  In both cases, there are
changes to be made in the circuit extension process.

Further, there are changes we need to make for the handshake between
the extending relay and the target relay.  The target relay is no
longer told by the client which of its onion keys it should use... so
the extending relay needs to tell the target relay which keys are in
the SNIP that the client is using.

<!-- Section 5.1 --> <a id='S5.1'></a>

## Modifying the EXTEND/CREATE handshake

First, we will require that proposal 249 (or some similar proposal
for wide CREATE and EXTEND cells) is in place, so that we can have
EXTEND cells larger than can fit in a single cell.  (See
319-wide-everything.md for an example proposal to supersede 249.)

We add new fields to the CREATE2 cell so that relays can send each
other more information without interfering with the client's part of
the handshake.

The CREATE2, CREATED2, and EXTENDED2 cells change as follows:

      struct create2_body {
         // old fields
         u16 htype; // client handshake type
         u16 hlen; // client handshake length
         u8 hdata[hlen]; // client handshake data.

         // new fields
         u8 n_extensions;
         struct extension extension[n_extensions];
      }

      struct created2_body {
         // old fields
         u16 hlen;
         u8 hdata[hlen];

         // new fields
         u8 n_extensions;
         struct extension extension[n_extensions];
      }

      struct truncated_body {
         // old fields
         u8 errcode;

         // new fields
         u8 n_extensions;
         struct extension extension[n_extensions];
      }

      // EXTENDED2 cells can now use the same new fields as in the
      // created2 cell.

      struct extension {
         u16 type;
         u16 len;
         u8 body[len];
      }

These extensions are defined by this proposal:

  [01] -- `Partial_SNIPRouterData` -- Sent from an extending relay
          to a target relay. This extension holds one or more fields
          from the SNIPRouterData that the extending relay is using,
          so that the target relay knows (for example) what keys to
          use.  (These fields are determined by the
          "forward_with_extend" field in the ENDIVE.)

  [02] -- Full_SNIP -- an entire SNIP that was used in an attempt to
          extend the circuit.  This must match the client's provided
          index position.

  [03] -- Extra_SNIP -- an entire SNIP that was not used to extend
          the circuit, but which the client requested anyway.  This
          can be sent back from the extending relay when the client
          specifies multiple index positions, or uses a nonzero "nth" value
          in their `snip_index_pos` link specifier.

  [04] -- SNIP_Request -- a 32-bit index position, or a single zero
          byte, sent away from the client.  If the byte is 0, the
          originator does not want a SNIP.  Otherwise, the
          originator does want a SNIP containing the router and the
          specified index.  Other values are unspecified.

By default, EXTENDED2 cells are sent with a SNIP iff the EXTENDED2
cell used a `snip_index_pos` link specifier, and CREATED2 cells are
not sent with a SNIP.

<!-- Section 5.1.1 --> <a id='S5.1.1'></a>

### New link specifiers

We add a new link specifier type for a router index, using the
following coding for its contents:

    /* Using trunnel syntax here. */
    struct snip_index_pos {
        u32 index_id; // which index is it?
        u8 nth; // how many SNIPs should be skipped/included?
        u8 index_pos[]; // extends to the end of the link specifier.
    }

The `index_pos` field can be longer or shorter than the actual width of
the router index.  If it is too long, it is truncated.  If it is too
short, it is extended with zero-valued bytes.

Any number of these link specifiers may appear in an EXTEND cell.
If there is more then one, then they should appear in order of
client preference; the extending relay may extend to any of the
listed routers.

This link specifier SHOULD NOT be used along with IPv4, IPv6, RSA
ID, or Ed25519 ID link specifiers.  Relays receiving such a link
specifier along with a `snip_index_pos` link specifier SHOULD reject
the entire EXTEND request.

If `nth` is nonzero, then link specifier means "the n'th SNIP after
the one defined by the SNIP index position."  A relay MAY reject
this request if `nth` is greater than 4.  If the relay does not
reject this request, then it MUST include all snips between
`index_pos` and the one that was actually used in an Extra_Snip
extension.  (Otherwise, the client would not be able to verify that
it had gotten the correct SNIP.)

> I've avoided use of CBOR for these types, under the assumption that we'd
> like to use CBOR for directory stuff, but no more.  We already have
> trunnel-like objects for this purpose.

<!-- Section 5.2 --> <a id='S5.2'></a>

## Modified ntor handshake

We adapt the ntor handshake from tor-spec.txt for this use, with the
following main changes.

  * The NODEID and KEYID fields are omitted from the input.
    Instead, these fields _may_ appear in a PartialSNIPData extension.

  * The NODEID and KEYID fields appear in the reply.

  * The NODEID field is extended to 32 bytes, and now holds the
    relay's ed25519 identity.

So the client's message is now:

   CLIENT_PK [32 bytes]

And the relay's reply is now:

   NODEID    [32 bytes]
   KEYID     [32 bytes]
   SERVER_PK [32 bytes]
   AUTH      [32 bytes]

otherwise, all fields are computed as described in tor-spec.

When this handshake is in use, the hash function is SHA3-256 and keys
are derived using SHAKE-256, as in rend-spec-v3.txt.

> Future work: We may wish to update this choice of functions
> between now and the implementation date, since SHA3 is a bit
> pricey.  Perhaps one of the BLAKEs would be a better choice.  If
> so, we should use it more generally.  On the other hand, the
> presence of public-key operations in the handshake _probably_
> outweighs the use of SHA3.

We will have to give this version of the handshake a new handshake
type.

<!-- Section 5.3 --> <a id='S5.3'></a>

## New relay behavior on EXTEND and CREATE failure.

If an EXTEND2 cell based on an routing index fails, the relay should
not close the circuit, but should instead send back a TRUNCATED cell
containing the SNIP in an extension.

If a CREATE2 cell fails and a SNIP was requested, then instead of
sending a DESTROY cell, the relay SHOULD respond with a CREATED2
cell containing 0 bytes of handshake data, and the SNIP in an
extension.  Clients MAY re-extend or close the circuit, but should
not leave it dangling.

<!-- Section 5.4 --> <a id='S5.4'></a>

## NIL handshake type

We introduce a new handshake type, "NIL".  The NIL handshake always
fails.  A client's part of the NIL handshake is an empty bytestring;
there is no server response that indicates success.

The NIL handshake can used by the client when it wants to fetch a
SNIP without creating a circuit.

Upon receiving a request to extend with the NIL circuit type, a
relay SHOULD NOT actually open any connection or send any data to
the target relay.  Instead, it should respond with a TRUNCATED cell
with the SNIP(s) that the client requested in one or more Extra_SNIP
extensions.

<!-- Section 5.5 --> <a id='S5.5'></a>

## Padding handshake cells to a uniform size

To avoid leaking information, all CREATE/CREATED/EXTEND/EXTENDED
cells SHOULD be padded to the same sizes.  In all cases, the amount
of padding is controlled by a set of network parameters:
"create-pad-len", "created-pad-len", "extend-pad-len" and
"extended-pad-len".  These parameters determine the minimum length
that the cell body or relay cell bodies should be.

If a cell would be sent whose body is less than the corresponding
parameter value, then the sender SHOULD pad the body by adding
zero-valued bytes to the cell body.  As usual, receivers MUST ignore
extra bytes at the end of cells.

> ALTERNATIVE: We could specify a more complicated padding
> mechanism, eg. 32 bytes of zeros then random bytes.

