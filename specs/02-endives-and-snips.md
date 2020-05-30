
<!-- Section 2 --> <a id='S2'></a>

# Document Formats: ENDIVEs and SNIPs

Here we specify a pair of related document formats that we will
use for specifying SNIPs and ENDIVEs.

Recall from proposal 300 that a SNIP is a set of information about
a single relay, plus proof from the directory authorities that the
given relay occupies a given range in a certain routing index.
For example, we can imagine that a SNIP might say:

* Relay X has the following IP, port, and onion key.
* In the routing index Y, it occupies index positions 0x20002
  through 0x23000.
* This SNIP is valid on 2020-12-09 00:00:00, for one hour.
* Here is a signature of all the above text, using a threshold
  signature algorithm.

You can think of a SNIP as a signed combination of a routerstatus and
a microdescriptor... together with a little bit of the randomized
routing table from Tor's current path selection code, all wrapped
in a signature.

Every relay keeps a set of SNIPs, and serves them to clients when
the client is extending by a routing index position.

An ENDIVE is a complete set of SNIPs.  Relays download ENDIVEs, or
diffs between ENDIVEs, once every voting period.  We'll accept some
complexity in order to make these diffs small, even though some of the
information in them (particularly SNIP signatures and index
ranges) will tend to change with every period.

<!-- Section 2.1 --> <a id='S2.1'></a>

## Preliminaries and scope

<!-- Section 2.1.1 --> <a id='S2.1.1'></a>

### Goals for our formats

We want SNIPs to be small, since they need to be sent on the wire
one at a time, and won't get much benefit from compression.  (To
avoid a side-channel, we want CREATED cells to all be the same
size, which means we need to pad up to the largest size possible
for a SNIP.)

We want to place as few requirements on clients as possible, and we want to
preserve forward compatibility.

We want ENDIVEs to be compressible, and small. We want them to benefit from
diffs.

We should preserve (and loosen) our policy of requiring only loose
time synchronization between clients and relays.  Where possible,
we'll make the permitted skew explicit in the protocol: for example,
rather than saying "you can accept a document 10 minutes before it
is valid", we will just make the validity interval start 10 minutes
earlier.

<!-- Section 2.1.2 --> <a id='S2.1.2'></a>

### Notes on Metaformat

In the format descriptions below, we will describe a set of
documents in the CBOR metaformat, as specified in RFC 7049.  If
you're not familiar with CBOR, you can think of it as a simple
binary version of JSON, optimized first for simplicity of
implementation and second for space.

I've chosen CBOR because it's schema-free (you can parse it
without knowing what it is), terse, dumpable as text, extensible,
standardized, and very easy to parse and encode.

We will choose to represent many size-critical types as maps whose
keys are short integers: this is slightly shorter in its encoding
than string-based dictionaries.  In some cases, we make types even
shorter by using arrays rather than maps, but only when we are
confident we will not have to make changes to the number of elements
in the future.

We'll use CDDL (defined in RFC 8610) to describe the data in a way
that can be validated -- and hopefully, in a way that will make it
comprehensible. (The state of CDDL tooling is a bit lacking at the
moment, so my CDDL validation will likely be imperfect.)

We make the following restrictions to CBOR documents that Tor
implementations will _generate_:

   * No floating-point values are permitted.

   * No tags are allowed unless otherwise specified.

   * All items must follow the rules of RFC 7049 section 3.9 for
     canonical encoding, unless otherwise specified.

Implementations SHOULD accept and parse documents that are not
generated according to these rules, for future extensibility.
However, implementations SHOULD reject documents that are not
"well-formed" and "valid" by the definitions of RFC 7049.

<!-- Section 2.1.3 --> <a id='S2.1.3'></a>

### Design overview: signing documents

We try to use a single document-signing approach here, using a hash
function parameterized to accommodate lifespan information and an
optional nonce.

All the signed CBOR data used in this format is represented as a
binary string, so that CBOR-processing tools are less likely to
re-encode or transform it.  We use the CDDL production `bstr .cbor
Object` to represent a binary string that must hold a valid encoding
of a CBOR object whose type is `Object`.

<!-- Section 2.1.4 --> <a id='S2.1.4'></a>

### Design overview: SNIP Authentication

I'm going to specify a flexible authentication format for SNIPs that
can handle threshold signatures, multisignatures, and Merkle trees.
This will give us flexibility in our choice of authentication
mechanism over time.

  * If we use Merkle trees, we can make ENDIVE diffs much much smaller,
    and save a bunch of authority CPU -- at the expense of requiring
    slightly larger SNIPs.

  * If Merkle tree root signatures are in SNIPs, SNIPs get a
    bit larger, but they can be used by clients that do not have the
    latest signed Merkle tree root.

  * If we use threshold signatures, we need to depend on
    not-yet-quite-standardized algorithms.  If we use multisignatures,
    then either SNIPs get bigger, or we need to put the signed Merkle
    tree roots into a consensus document.

Of course, flexibility in signature formats is risky, since the more
code paths there are, the more opportunities there are for nasty bugs.
With this in mind, I'm structuring our authentication so that there
should (to the extent possible) be only a single validation path for
different uses.

With this in mind, our format is structured so that "not using a
Merkle tree" is considered, from the client's point of view, the same as
"using a Merkle of depth 1".

The authentication on a single snip is structured, in the abstract, as:
   - ITEM: The item to be authenticated.
   - PATH: A string of N bits, representing a path through a Merkle tree from
     its root, where 0 indicates a left branch and 1 indicates a right
     branch.  (Note that in a left-leaning tree, the 0th leaf will have
     path 000..0, the 1st leaf will have path 000..1, and so on.)
   - BRANCH: A list of N digests, representing the digests for the
     branches in the Merkle tree that we are _not_ taking.
   - SIG: A generalized signature (either a threshold signature or a
     multisignature) of a top-level digest.
   - NONCE: an optional nonce for use with the hash functions.

Note that PATH here is a bitstring, not an integer! "0001" and "01" are
different paths, and "" is a valid path, indicating the root of the tree.

We assume two hash functions here: `H_leaf()` to be used with leaf
items, and `H_node()` to be used with intermediate nodes.  These functions
are parameterized with a path through the tree, with a nonce, and with
the lifespan of the object to be signed.

To validate the authentication on a SNIP, the client proceeds as follows:

    Algorithm: Validating SNIP authentication

    Let N = the length of PATH, in bits.

    Let H = H_leaf(PATH, LIFESPAN, NONCE, ITEM).

    While N > 0:
       Remove the last bit of PATH; call it P.
       Remove the last digest of BRANCH; call it B.

       If P is zero:
           Let H = H_node(PATH, LIFESPAN, NONCE, H, B)
       else:
           Let H = H_node(PATH, LIFESPAN, NONCE, B, H)

       Let N = N - 1

    Check wither SIG is a correct (multi)signature over H with the
    correct key(s).

Parameterization on this structure is up to the authorities.  If N is
zero, then we are not using a Merkle tree.  The generalize signature
SIG can either be given as part of the SNIP, or as part of a consensus
document.  I expect that in practice, we will converge on a single set of
parameters here quickly (I'm favoring BLS signatures and a Merkle
tree), but using this format will give clients the flexibility to handle
other variations in the future.

For our definition of `H_leaf()` and `H_node()`, see "Digests and
parameters" below.

<!-- Section 2.1.5 --> <a id='S2.1.5'></a>

### Design overview: timestamps and validity.

For future-proofing, SNIPs and ENDIVEs have separate time ranges
indicating when they are valid.  Unlike with current designs, these
validity ranges should take clock skew into account, and should not
require clients or relays to deliberately add extra tolerance to their
processing.  (For example, instead of saying that a document is "fresh"
for three hours and then telling clients to accept documents for 24
hours before they are valid and 24 hours after they are expired, we will
simply make the documents valid for 51 hours.)

We give each lifespan as a (PUBLISHED, PRE, POST) triple, such that
objects are valid from (PUBLISHED - PRE) through (PUBLISHED + POST).
(The "PUBLISHED" time is provided so that we can more reliably tell
which of two objects is more recent.)

Later (see section 08), we'll explain measures to ensure that
hostile relays do not take advantage of multiple overlapping SNIP
lifetimes to attack clients.

<!-- Section 2.1.6 --> <a id='S2.1.6'></a>

### Design overview: how the formats work together

Authorities, as part of their current voting process, will produce an
ENDIVE.

Relays will download this ENDIVE (either directly or as a diff),
validate it, and extract SNIPs from it.  Extracting these SNIPs may be
trivial (if they are signed individually), or more complex (if they are
signed via a merkle tree, and the merkle tree needs to be
reconstructed).  This complexity is acceptable only to the extent that
it reduces compressed diff size.

Once the SNIPs are reconstructed, relays will hold them and serve them
to clients.

<!-- Section 2.1.7 --> <a id='S2.1.7'></a>

### What isn't in this section

This section doesn't tell you what the different routing indices
are or mean.  For now, we can imagine there being one routing index for
guards, one for middles, and one for exits, and one for each hidden
service directory ring. (See section 06 for more on regular indices,
and section 07 for more on onion services.)

This section doesn't give an algorithm for computing ENDIVEs from
votes, and doesn't give an algorithm for extracting SNIPs from an ENDIVE.
Those come later. (See sections 03 and 04 respectively.)

<!-- Section 2.2 --> <a id='S2.2'></a>

## SNIPs

Each SNIP has three pieces: the part of the SNIP that describes the
router, the part of that describes the SNIPs place within an ENDIVE, and
the part that authenticates the whole SNIP.

Why two _separate_ authenticated pieces?  Because one (the router
description) is taken verbatim from the ENDIVE, and the other
(the location within the ENDIVE) is computed from the ENDIVE by the
relays. Separating them like this helps ensure that the part
generated by the relay and the part generated by the authorities
can't interfere with each other.

    ; A SNIP, as it is sent from the relay to the client.  Note that
    ; this is represented as a three-element array.
    SNIP = [
        ; First comes the signature.  This is computed over
        ; the concatenation of the two bstr objects below.
        auth: SNIPSignature,

        ; Next comes the location of the SNIP within the ENDIVE.
        index : bstr .cbor SNIPLocation,

        ; Finally comes the information about the router.
        router : bstr .cbor SNIPRouterData,
    ]

(Computing the signature over a concatenation of objects is safe, since
the objects' content is self-describing CBOR, and isn't vulnerable to
framing issues.)

<!-- Section 2.2.1 --> <a id='S2.2.1'></a>

### SNIPRouterData: information about a single router.

Here we talk about the type that tells a client about a single
router.  For cases where we are just storing information about a
router (for example, when using it as a guard) we can remember
this part, and discard the other pieces.

The only required parts here are those that identify the router
and tell the client how to build a circuit through it.  The others
are all optional.  In practice, I expect they will be encoded in most
cases, but clients MUST behave properly if they are absent.

More than one SNIPRouterData may exist in the same ENDIVE for a
single router.  For example, there might be a longer version to
represent a router to be used as a guard, and another to represent
the same router when used as a hidden service directory.  (This is
not possible in the voting mechanism that I'm working on, but relays
and clients MUST NOT treat this as an error.)

This representation is based on the routerstats and
microdescriptor entries of today, but tries to omit a number of
obsolete fields, including RSA identity fingerprint, TAP key,
published time, etc.

    ; A SNIPRouterData is a map from integer keys to values for
    ; those key.
    SNIPRouterData = {
        ; identity key.
        ? 0 => Ed25519PublicKey,

        ; ntor onion key.
        ? 1 => Curve25519PublicKey,

        ; list of link specifiers other than the identity key.
        ; If a client wants to extend to the same router later on,
        ; they SHOULD include all of these link specifiers verbatim,
        ; whether they recognize them or not.
        ? 2 => [ LinkSpecifier ],

        ; The software that this relay says it is running.
        ? 3 => SoftwareDescription,

        ; protovers.
        ? 4 => ProtoVersions,

        ; Family.  See below for notes on dual encoding.
        ? 5 => [ * FamilyId ],

        ; Country Code
        ? 6 => Country,

        ; Exit policies describing supported port _classes_.  Absent exit
        ; policies are treated as "deny all".
        ? 7 => ExitPolicy,

        ; NOTE: Properly speaking, there should be a CDDL 'cut'
        ; here, to indicate that the rules below sould only match
        ; if one if the previous rules hasn't matched.
        ; Unfortunately, my CDDL tool doesn't seem to support cuts.

        ; For future tor extensions.
        * int => any,

        ; For unofficial and experimental extensions.
        * tstr => any,
    }

    ; For future-proofing, we are allowing multiple ways to encode
    ; families.  One is as a list of other relays that are in your
    ; family.  One is as a list of authority-generated family
    ; identifiers. And one is as a master key for a family (as in
    ; Tor proposal 242).
    ;
    ; A client should consider two routers to be in the same
    ; family if they have at last one FamilyId in common.
    ; Authorities will canonicalize these lists.
    FamilyId = bstr

    ; A country.  These should ordinarily be 2-character strings,
    ; but I don't want to enforce that.
    Country = tstr;

    ; SoftwareDescription replaces our old "version".
    SoftwareDescription = [
      software : tstr,
      version : tstr,
      extra : tstr
    ]

    ; Protocol versions: after a bit of experimentation, I think
    ; the most reasonable representation to use is a map from protocol
    ; ID to a bitmask of the supported versions.
    ProtoVersions = { ProtoId => ProtoBitmask }

    ; Integer protocols are reserved for future version of Tor. tstr ids
    ; are reserved for experimental and non-tor extensions.
    ProtoId = ProtoIdEnum / int / tstr

    ProtoIdEnum = &(
      Link      : 0,
      LinkAuth  : 1,
      Relay     : 2,
      DirCache  : 3,
      HSDir     : 4,
      HSIntro   : 5,
      HSRend    : 6,
      Desc      : 7,
      MicroDesc : 8,
      Cons      : 9,
      Padding   : 10,
      FlowCtrl  : 11,
    )
    ; This type is limited to 64 bits, and that's fine.  If we ever
    ; need a protocol version higher than 63, we should allocate a
    ; new protoid.
    ProtoBitmask = uint

    ; An exit policy may exist in up to two variants.  When port classes
    ; have not changed in a while, only one policy is needed.  If port
    ; classes have changed recently, however, then SNIPs need to include
    ; each relay's position according to both the older and the newer policy
    ; until older network parameter documents become invalid.
    ExitPolicy = SinglePolicy / [ SinglePolicy, SinglePolicy ]

    ; Each single exit policy is a tagged bit array, whose bits
    ; correspond to the members of the list of port classes in the
    ; network parameter document with a corresponding tag.
    SinglePolicy = [
         ; Identifies which group of port classes we're talking about
         tag : unsigned,
         ; Bit-array of which port classes this relay supports.
         policy : bstr
    ]

<!-- Section 2.2.2 --> <a id='S2.2.2'></a>

### SNIPLocation: Locating a SNIP within a routing index.

The SNIPLocation type can encode where a SNIP is located with
respect to one or more routing indices.  Note that a SNIPLocation
does not need to be exhaustive: If a given IndexId is not listed for
a given relay in one SNIP, it might exist in another SNIP. Clients
should not infer that the absence of an IndexId in one SNIPLocation
for a relay means that no SNIPLocation with that IndexId exists for
the relay.

    ; SNIPLocation: we're using a map here because it's natural
    ; to look up indices in maps.
    SNIPLocation = {
        ; A SNIP's location is given as a index ranges in different
        ; indices.
        * IndexId => IndexRange / ExtensionIndex,
    }

    ; We'll define the different index ranges as we go on with
    ; these specifications.
    ;
    ; IndexId values over 65535 are reserved for extensions and
    ; experimentation.
    IndexId = uint32

    ; An index range extends from a minimum to a maximum value.
    ; These ranges are _inclusive_ on both sides.  If 'hi' is less
    ; than 'lo', then this index "wraps around" the end of the ring.
    ; A "nil" value indicates an empty range, which would not
    ; ordinarily be included.
    IndexRange = [ lo: IndexPos,
                   hi: IndexPos ] / nil

    ; An ExtensionIndex is reserved for future use; current clients
    ; will not understand it and current ENDIVEs will not contain it.
    ExtensionIndex = any

    ; For most routing indices, the ranges are encoded as 4-byte integers.
    ; But for hsdir rings, they are binary strings.  (Clients and
    ; relays SHOULD NOT require this.)
    IndexPos = uint / bstr

A bit more on IndexRanges: Every IndexRange actually describes a set of
_prefixes_ for possible index positions.  For example, the IndexRange
`[ h'AB12', h'AB24' ]` includes all the binary strings that start with (hex)
`AB12`, `AB13`, and so on, up through all strings that start with `AB24`.
Alternatively, you can think of a `bstr`-based IndexRange *(lo, hi)* as
covering *lo*`00000...` through *hi*`ff...`.

IndexRanges based on the uint type work the same, except that they always
specify the first 32 bits of a prefix.

<!-- Section 2.2.3 --> <a id='S2.2.3'></a>

### SNIPSignature: How to prove a SNIP is in the ENDIVE.

Here we describe the types for implementing SNIP signatures, to be
validated as described in "Design overview: Authentication" above.

    ; Most elements in a SNIPSignature are positional and fixed
    SNIPSignature = [
        ; The actual signature or signatures.  If this is a single signature,
        ; it's probably a threshold signature.  Otherwise, it's probably
        ; a list containing one signature from each directory authority.
        SingleSig / MultiSig,

        ; algorithm to use for the path through the merkle tree.
        d_alg : DigestAlgorithm,
        ; Path through merkle tree, possibly empty.
        merkle_path : MerklePath,

        ; Lifespan information.  This is included as part of the input
        ; to the hash algorithm for the signature.
        LifespanInfo,

        ; optional nonce for hash algorithm.
        ? nonce : bstr,

        ; extensions for later use. These are not signed.
        ? extensions : { * any => any },
    ]

    ; We use this group to indicate when an object originated, and when
    ; it should be accepted.
    ;
    ; When we are using it as an input to a hash algorithm for computing
    ; signatures, we encode it as an 8-byte number for "published",
    ; followed by two 4-byte numbers for pre-valid and post-valid.
    LifespanInfo = (
        ; Official publication time in seconds since the epoch.  These
        ; MUST be monotonically increasing over time for a given set of
        ; authorities on all SNIPs or ENDIVEs that they generate: a
        ; document with a greater `published` time is always more recent
        ; than one with an earlier `published` time.
        ;
        ; Seeing a publication time "in the future" on a correctly
        ; authenticated documented is a reliable sign that your
        ; clock is set too far in the past.
        published: uint,

        ; Value to subtract from "published" in order to find the first second
        ; at which this object should be accepted.
        pre-valid : uint32,

        ; Value to add to "published" in order to find the last
        ; second at which this object should be accepted.  The
        ; lifetime of an object is therefore equal to "(post-valid +
        ; pre-valid)".
        post-valid : uint32,
    )

    ; A Lifespan is just the fields of LifespanInfo, encoded as a list.
    Lifespan = [ LifespanInfo ]

    ; One signature on a SNIP or ENDIVE.  If the signature is a threshold
    ; signature, or a reference to a signature in another
    ; document, there will probably be just one of these per SNIP.  But if
    ; we're sticking a full multisignature in the document, this
    ; is just one of the signatures on it.
    SingleSig = [
       s_alg: SigningAlgorithm,
       ; One of signature and sig_reference must be present.
       ?signature : bstr,
       ; sig_reference is an identifier for a signature that appears
       ; elsewhere, and can be fetched on request.  It should only be
       ; used with signature types too large to attach to SNIPs on their
       ; own.
       ?sig_reference : bstr,
       ; A prefix of the key or the key's digest, depending on the
       ; algorithm.
       ?keyid : bstr,
    ]

    MultiSig = [ + SingleSig ]

    ; A Merkle path is represented as a sequence of bits to
    ; indicate whether we're going left or right, and a list of
    ; hashes for the parts of the tree that we aren't including.
    ;
    ; (It's safe to use a uint for the number of bits, since it will
    ; never overflow 64 bits -- that would mean a Merkle tree with
    ; too many leaves to actually calculate on.)
    MerklePath = [ uint, *bstr ]

<!-- Section 2.3 --> <a id='S2.3'></a>

## ENDIVEs: sending a bunch of SNIPs efficiently.

ENDIVEs are delivered by the authorities in a compressed format, optimized
for diffs.

Note that if we are using Merkle trees for SNIP authentication, ENDIVEs do
not include the trees at all, since those can be inferred from the leaves of
the tree.  Similarly, the ENDIVEs do not include raw routing indices, but
instead include a set of bandwidths that can be combined into the routing
indices -- these bandwidths change less frequently, and therefore are more
diff-friendly.

Note also that this format has more "wasted bytes" than SNIPs
do. Unlike SNIPs, ENDIVEs are large enough to benefit from
compression with with gzip, lzma2, or so on.

This section does not fully specify how to construct SNIPs from an ENDIVE;
for the full algorithm, see section 04.

    ; ENDIVEs are also sent as CBOR.
    ENDIVE = [
        ; Signature for the ENDIVE, using a simpler format than for the
        ; a SNIP.  Since ENDIVEs are more like a consensus, we don't need
        ; to use threshold signatures or merkle paths here.
        sig: ENDIVESignature,

        ; Contents, as a binary string.
        body: encoded-cbor .cbor ENDIVEContent,
    ]

    ; The set of signatures across an ENDIVE.
    ;
    ; This type doubles as the "detached signature" document used when
    ; collecting signatures for a consensus.
    ENDIVESignature = {
        ; The actual signatures on the endive. A multisignature is the
        ; likeliest format here.
        endive_sig: [ + SingleSig ],

        ; Lifespan information.  As with SNIPs, this is included as part
        ; of the input to the hash algorithm for the signature.
        ; Note that the lifespan of an ENDIVE is likely to be a subset
        ; of the lifespan of its SNIPs.
        endive_lifespan: Lifespan,

        ; Signatures across SNIPs, at some level of the Merkle tree.  Note
        ; that these signatures are not themselves signed -- having them
        ; signed would take another step in the voting algorithm.
        snip_sigs : DetachedSNIPSignatures,

        ; Signatures across the ParamDoc pieces.  Note that as with the
        ; DetachedSNIPSignatures, these signature are not themselves signed.
        param_doc: ParamDocSignature,

        ; extensions for later use. These are not signed.
        * tstr => any,
    }

    ; A list of single signatures or a list of multisignatures. This
    ; list must have 2^signature-depth elements.
    DetachedSNIPSignatures =
          [ *SingleSig ] / [ *MultiSig ]

    ENDIVEContent = {

        ; Describes how to interpret the signatures over the SNIPs in this
        ; ENDIVE. See section 04 for the full algorithm.
        sig_params : {
            ; When should we say that the signatures are valid?
            lifespan : Lifespan,
            ; Nonce to be used with the signing algorithm for the signatures.
            ? signature-nonce : bstr,

            ; At what depth of a Merkle tree to the signatures apply?
            ; (If this value is 0, then only the root of the tree is signed.
            ; If this value is >= ceil(log2(n_leaves)), then every leaf is
            ; signed.).
            signature-depth : uint,

            ; What digest algorithm is used for calculating the signatures?
            signature-digest-alg: DigestAlgorithm,

            ; reserved for future extensions.
            * tstr => any,
        },

        ; Documents for clients/relays to learn about current network
        ; parameters.
        client-param-doc : encoded-cbor .cbor ClientParamDoc,
        relay-param-doc : encoded-cbor .cbor RelayParamDoc,

        ; Definitions for index group.  Each "index group" is all
        ; applied to the same SNIPs.  (If there is one index group,
        ; then every relay is in at most one SNIP, and likely has several
        ; indices.  If there are multiple index groups, then relays
        ; can appear in more than one SNIP.)
        indexgroups : [ *IndexGroup ],

        ; Information on particular relays.
        ;
        ; (The total number of SNIPs identified by an ENDIVE is at most
        ; len(indexgroups) * len(relays).)
        relays : [ * ENDIVERouterData ],

        ; for future exensions
        * tstr => any,
    }

    ; An "index group" lists a bunch of routing indices that apply to the same
    ; SNIPs.  There may be multiple index groups when a relay needs to appear
    ; in different SNIPs with routing indices for some reason.
    IndexGroup = {
        ; A list of all the indices that are built for this index group.
        ; An IndexId may appear in at most one group per ENDIVE.
        indices : [ + IndexId ],
        ; A list of keys to delete from SNIPs to build this index group.
        omit_from_snips : [ *(int/tstr) ],
        ; A list of keys to forward from SNIPs to the next relay in an EXTEND
        ; cell.  This can help the next relay know which keys to use in its
        ; handshake.
        forward_with_extend : [ *(int/tstr) ],

        ; A number of "gaps" to place in the Merkle tree after the SNIPs
        ; in this group.  This can be used together with signature-depth
        ; to give different index-groups independent signatures.
        ? n_padding_entries : uint,

        ; A detailed description of how to build the index.
        + IndexId => IndexSpec,

        ; For experimental and extension use.
        * tstr => any,
    }

    ; Enumeration to identify how to generate an index.
    Indextype_Raw = 0
    Indextype_Weighted = 1
    Indextype_RSAId = 2
    Indextype_Ed25519Id = 3
    Indextype_RawNumeric = 4

    ; An indexspec may given as a raw set of index ranges.  This is a
    ; fallback for cases where we simply can't construct an index any other
    ; way.
    IndexSpec_Raw = {
        type : Indextype_Raw,
        ; This index is constructed by taking relays by their position in the
        ; list from the list of ENDIVERouterData, and placing them at a given
        ; location in the routing index.  Each index range extends up to
        ; right before the next index position.
        index_ranges: [ * [ uint, IndexPos ] ],
    }

    ; An indexspec where we're placing routers from the list of
    ; ENDIVERouterData index, and by their numeric spans on the index.
    IndexSpec_RawNumeric = {
        type: Indextype_RawNumeric,
        first_index_pos : uint,
        ; This index is constructed by taking relays by index from the list
        ; of ENDIVERouterData, and giving them a certain amount of "weight"
        ; in the index.
        index_ranges: [ * [ idx : uint, span : uint ] ],
    }

    ; This index is computed from the weighted bandwidths of all the SNIPs.
    ;
    ; Note that when a single bandwidth changes, it can change _all_ of
    ; the indices in a bandwidth-weighted index, even if no other
    ; bandwidth changes.  That's why we only pack the bandwidths
    ; here, and scale them as part of the reconstruction algorithm.
    IndexSpec_Weighted = {
        type: Indextype_Weighted,
        ; This index is constructed by assigning a weight to each relay,
        ; and then normalizing those weights. See algorithm below in section
        ; 04.
        ; Limiting bandwidth weights to uint32 makes reconstruction algorithms
        ; much easier.
        index_weights: [ * uint32 ],
    }

    ; This index is computed from the RSA identity keys digests of all of the
    ; SNIPs. It is used in the HSv2 directory ring.
    IndexSpec_RSAId = {
        type: Indextype_RSAId,
        ; How many bytes of RSA identity data go into each indexpos entry?
        n_bytes: uint,
        ; Bitmap of which routers should be included.
        members : bstr,
    }

    ; This index is computed from the Ed25519 identity keys of all of the
    ; SNIPs.  It is used in the HSv3 directory ring.
    IndexSpec_Ed25519Id = {
        type : Indextype_Ed25519Id,
        ; How many bytes of digest go into each indexpos entry?
        n_bytes : uint,
        ; What digest do we use for building this ring?
        d_alg : DigestAlgorithm,
        ; What bytes do we give to the hash before the ed25519?
        prefix : bstr,
        ; What bytes do we give to the hash after the ed25519?
        suffix : bstr,
        ; Bitmap of which routers should be included.
        members : bstr,
    }

    IndexSpec = IndexSpec_Raw /
                IndexSpec_RawNumeric /
                IndexSpec_Weighted /
                IndexSpec_RSAId /
                IndexSpec_Ed25519Id

    ; Information about a single router in an ENDIVE.
    ENDIVERouterData = {
        ; The authority-generated SNIPRouterData for this router.
        1 => encoded-cbor .cbor SNIPRouterData,
        ; The RSA identity, or a prefix of it, to use for HSv2 indices.
        ? 2 => RSAIdentityFingerprint,

        * int => any,
        * tstr => any,
    }

    ; encoded-cbor is defined in the CDDL postlude as a bstr that is
    ; tagged as holding verbatim CBOR:
    ;
    ;    encoded-cbor = #6.24(bstr)
    ;
    ; Using a tag like this helps tools that validate the string as
    ; valid CBOR; using a bstr helps indicate that the signed data
    ; should not be interpreted until after the signature is checked.
    ; It also helps diff tools know that they should look inside these
    ; objects.

<!-- Section 2.4 --> <a id='S2.4'></a>

## Network parameter documents

Network parameter documents ("ParamDocs" for short) take the place of the
current consensus and certificates as a small document that clients and
relays need to download periodically and keep up-to-date.  They are generated
as part of the voting process, and contain fields like network parameters,
recommended versions, authority certificates, and so on.

    ; A "parameter document" is like a tiny consensus that relays and clients
    ; can use to get network parameters.
    ParamDoc = [
       sig : ParamDocSignature,
       ; Client-relevant portion of the parameter document. Everybody fetches
       ; this.
       cbody : encoded-cbor .cbor ClientParamDoc,
       ; Relay-relevant portion of the parameter document. Only relays need to
       ; fetch this; the document can be validated without it.
       ? sbody : encoded-cbor .cbor RelayParamDoc,
    ]
    ParamDocSignature = [
       ; Multisignature or threshold signature of the concatenation
       ; of the two digests below.
       SingleSig / MultiSig,

       ; Lifespan information.  As with SNIPs, this is included as part
       ; of the input to the hash algorithm for the signature.
       ; Note that the lifespan of a parameter document is likely to be
       ; very long.
       LifespanInfo,

       ; how are c_digest and s_digest the digest computed?
       d_alg : DigestAlgorithm,
       ; Digest over the cbody field
       c_digest : bstr,
       ; Digest over the sbody field
       s_digest : bstr,
    ]

    ClientParamDoc = {
       params : NetParams,
       ; List of certificates for all the voters.  These
       ; authenticate the keys used to sign SNIPs and ENDIVEs and votes,
       ; using the authorities longest-term identity keys.
       voters : [ + bstr .cbor VoterCert ],

       ; A division of exit ports into "classes" of ports.
       port-classes: PortClasses,

       ; As in client-versions from dir-spec.txt
       ? recommend-versions: [ * tstr ],
       ; As in recommended-client-protocols in dir-spec.txt
       ? recommend-protos: ProtoVersions,
       ; As in required-client-protocols in dir-spec.txt
       ? require-protos: ProtoVersions,

       ; For future extensions.
       * tstr => any,
    }

    RelayParamDoc = {
       params: NetParams,

       ; As in server-versions from dir-spec.txt
       ? recommend-versions: [ * tstr ],
       ; As in recommended-relay-protocols in dir-spec.txt
       ? recommend-protos: ProtoVersions,
       ; As in required-relay-protocols in dir-spec.txt
       ? require-versions: ProtoVersions,

       * tstr => any,
    }

    ; A NetParams encodes information about the Tor network that
    ; clients and relays need in order to participate in it.  The
    ; current list of parameters is described in the "params" field
    ; as specified in dir-spec.txt.
    ;
    ; Note that there are separate client and relay NetParams now.
    ; Relays are expected to first check for a defintion in the
    ; RelayParamDoc, and then in the ClientParamDoc.
    NetParams = { *tstr => int }

    PortClasses = {
        ; identifies which port class grouping this is. Used to migrate
        ; from one group of port classes to another.
        tag : uint,
        ; list of the port classes.
        classes: { * IndexId => PortList },
    }
    PortList = [ *PortOrRange ]
     ; Either a single port or a low-high pair
    PortOrRange = Port / [ Port, Port ]
    Port = 1...65535

<!-- Section 2.5 --> <a id='S2.5'></a>

## Certificates

Voting certificates are used to bind authorities' long-term
identities to shorter-term signing keys.  These have a similar
purpose to the authority certs made for the existing voting
algorithm, but support more key types.

    ; A 'voter certificate' is a statement by an authority binding keys to
    ; each other.
    VoterCert = [

       ; One or more signatures over `content` using the provided lifetime.
       ; Each signature should be treated independently.
       [ + SingleSig ],
       ; A lifetime value, used (as usual ) as an input to the
       ; signature algorithm.
       LifespanInfo,
       ; The keys and other data to be certified.
       content : encoded-cbor .cbor CertContent,
    ]

    ; The contents of the certificate that get signed.
    CertContent = {
       ; What kind of a certificate is this?
       type : CertType,
       ; A list of keys that are being certified in this document
       keys : [ + CertifiedKey ],
       ; A list of other keys that you might need to know about, which
       ; are NOT certififed in this document.
       ? extra : [ + CertifiedKey ],
       * tstr => any,
    }

    CertifiedKey = {
       ; What is the intended usage of this key?
       usage : KeyUsage,
       ; What cryptographic algorithm is this key used for?
       alg : PKAlgorithm,
       ; The actual key being certified.
       data : bstr,
       ; A human readable string.
       ? remarks : tstr,
       * tstr => any,
    }

<!-- Section 2.6 --> <a id='S2.6'></a>

## ENDIVE diffs

Here is a binary format to be used with ENDIVEs, ParamDocs, and any
other similar binary formats.  Authorities and directory caches need to
be able to generate it; clients and non-cache relays only need to be
able to parse and apply it.

    ; Binary diff specification.
    BinaryDiff = {
        ; This is version 1.
        v : 1,
        ; Optionally, a diff can say what different digests
        ; of the document should be before and after it is applied.
        ; If there is more than one entry, parties MAY check one or
        ; all of them.
        ? digest : { * DigestAlgorithm =>
                         [ pre : Digest,
                           post : Digest ]},

        ; Optionally, a diff can give some information to identify
        ; which document it applies to, and what document you get
        ; from applying it.  These might be a tuple of a document type
        ; and a publication type.
        ? ident : [ pre : any, post : any ],

        ; list of commands to apply in order to the original document in
        ; order to get the transformed document
        cmds : [ *DiffCommand ],

        ; for future extension.
        * tstr => any,
    }

    ; There are currently only two diff commands.
    ; One is to copy some bytes from the original.
    CopyDiffCommand = [
        OrigBytesCmdId,
        ; Range of bytes to copy from the original document.
        ; Ranges include their starting byte.  The "offset" is relative to
        ; the end of the _last_ range that was copied.
        offset : int,
        length : uint,
    ]

    ; The other diff comment is to insert some bytes from the diff.
    InsertDiffCommand = [
        InsertBytesCmdId,
        data : bstr,
    ]

    DiffCommand = CopyDiffCommand / InsertDiffCommand

    OrigBytesCmdId = 0
    InsertBytesCmdId = 1

Applying a binary diff is simple:

    Algorithm: applying a binary diff.

    (Given an input bytestring INP and a diff D, produces an output OUT.)

    Initialize OUT to an empty bytestring.

    Set OFFSET to 0.

    For each command C in D.commands, in order:

        If C begins with OrigBytesCmdId:
            Increase "OFFSET" by C.offset
            If OFFSET..OFFSET+C.length is not a valid range in
               INP, abort.
            Append INP[OFFSET .. OFFSET+C.length] to OUT.
            Increase "OFFSET" by C.length

        else: # C begins with InsertBytesCmdId:
            Append C.data to OUT.

Generating a binary diff can be trickier, and is not specified here.
There are several generic algorithms out there for making binary diffs
between arbitrary byte sequences. Since these are complex, I recommend a
chunk-based CBOR-aware algorithm, using each CBOR item in a similar way
to the way in which our current line-oriented code uses lines.  When
encountering a bstr tagged with "encoded-cbor", the diff algorithm
should look inside it to find more cbor chunks. (See
example-code/cbor_diff.py for an example of doing this with Python's
difflib.)

The diff format above should work equally well no matter what
diff algorithm is used, so we have room to move to other algorithms
in the future if needed.

To indicate support for the above diff format in directory requests,
implementations should use an `X-Support-Diff-Formats` header.  The
above format is designated "cbor-bindiff"; our existing format is
called "ed".

<!-- Section 2.7 --> <a id='S2.7'></a>

## Digests and parameters

Here we give definitions for `H_leaf()` and `H_node()`, based on an
underlying digest function H() with a preferred input block size of B.
(B should be chosen as the natural input size of the hash function, to
aid in precomputation.)

We also define `H_sign()`, to be used outside of SNIP authentication
where we aren't using a Merkle tree at all.

PATH must be no more than 64 bits long.  NONCE must be no more than B-33
bytes long.

     H_sign(LIFESPAN, NONCE, ITEM) =
        H( PREFIX(OTHER_C, LIFESPAN, NONCE) || ITEM)

     H_leaf(PATH, LIFESPAN, NONCE, ITEM) =
        H( PREFIX(LEAF_C, LIFESPAN, NONCE) ||
           U64(PATH) ||
           U64(bits(path)) ||
           ITEM )

     H_node(PATH, LIFESPAN, NONCE, ITEM) =
        H( PREFIX(NODE_C, LIFESPAN, NONCE) ||
           U64(PATH) ||
           U64(bits(PATH)) ||
           ITEM )

     PREFIX(leafcode, lifespan, nonce) =
          U64(leafcode) ||
          U64(lifespan.published) ||
          U32(lifespan.pre-valid) ||
          U32(lifespan.post-valid) ||
          U8(len(nonce)) ||
          nonce ||
          Z(B - 33 - len(nonce))

     LEAF_C = 0x8BFF0F687F4DC6A1 ^ NETCONST
     NODE_C = 0xA6F7933D3E6B60DB ^ NETCONST
     OTHER_C = 0x7365706172617465 ^ NETCONST

     # For the live Tor network only.
     NETCONST = 0x0746f72202020202
     # For testing networks, by default.
     NETCONST = 0x74657374696e6720

     U64(n) -- N encoded as a big-endian 64-bit number.
     Z(n) -- N bytes with value zero.
     len(b) -- the number of bytes in a byte-string b.
     bits(b) -- the number of bits in a bit-string b.

