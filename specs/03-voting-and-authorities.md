
<!-- Section 3 --> <a id='S3'></a>

# Directory authority operations

For Walking Onions to work, authorities must begin to generate
ENDIVEs as a new kind of "consensus document".  Since this format is
incompatible with the previous consensus document formats, and is
CBOR-based, a text-based voting protocol is no longer appropriate
for generating it.

We cannot immediately abandon the text-based consensus and
microdescriptor formats, however, but instead will need to keep
generating them for legacy relays and clients.  Ideally, the legacy
consensus should become a byproduct of the same voting process as is
used to produce ENDIVEs, to limit the amount of divergence in
contents.

Further, it would be good for the purposes of this proposal if we
can "inherit" as our existing voting mechanism for legacy purposes.

This section of the proposal will try to solve these goals by defining a
new binary-based voting format, a new set of voting rules for it, and a
series of migration steps.

<!-- Section 3.1 --> <a id='S3.1'></a>

## Overview

Except as described below, we retain from Tor's existing voting
mechanism all notions of how votes are transferred and processed.
Other changes are likely desirable, but they are out of scope for
this proposal.

Notably, we are not changing how the voting schedule works.  Nor are
we changing the property that all authorities must agree on the list
of authorities; the property that that a consensus is computed as a
deterministic function of a set of votes; or the property that if
authorities believe in different sets of votes, they will not reach
the same consensus.

The principal changes in the voting that are relevant for legacy
consensus computation are:

  * The uploading process for votes now supports negotiation, so
    that the receiving authority can tell the uploading authority
    what kind of formats, diffs, and compression it supports.

  * We specify a CBOR-based binary format for votes, with a simple
    embedding method for the legacy text format.  This embedding is
    meant for transitional use only; once all authorities support
    the binary format, the transitional format and its support
    structures can be abandoned.

  * To reduce complexity, the new vote format also includes
    _verbatim_ microdescriptors, whereas previously microdescriptors
    would have been referenced by hash.  (The use of diffs and
    compression should make the bandwidth impact of this addition
    negligible.)

For computing ENDIVEs, the principle changes in voting are:

  * The consensus outputs for most voteable objects are specified in a
    way that does not require the authorities to understand their
    semantics when computing a consensus.  This should make it
    easier to change fields without requiring new consensus methods.

<!-- Section 3.2 --> <a id='S3.2'></a>

## Negotiating vote uploads

Authorities supporting Walking Onions are required to support a new
resource "/tor/auth-vote-opts".  This resource is a text document
containing a list of HTTP-style headers. Recognized headers are
described below; unrecognized headers MUST be ignored.

The *Accept-Encoding* header follows the same as the HTTP header of
the same name; it indicates a list of Content-Encodings that the
authority will accept for uploads.  All authorities SHOULD support
the gzip and identity encodings.  The identity encoding is mandatory.
(Default: "identity")

The *Accept-Vote-Diffs-From* header is a list of digests of previous
votes held by this authority; any new uploaded votes that are given
as diffs from one of these old votes SHOULD be accepted.  The format
is a space-separated list of "digestname:Hexdigest".  (Default: "".)

The *Accept-Vote-Formats* header is a space-separated list of the
vote formats that this router accepts. The recognized vote formats
are "legacy-3" (Tor's current vote format) and "endive-1" (the vote
format described here). Unrecognized vote formats MUST be ignored.
(Default: "legacy-3".)

If requesting "/tor/auth-vote-opts" gives an error, or if one or
more headers missing, the default values SHOULD be used.  These
documents (or their absence) MAY be cached for up to 2 voting
periods.)

Authorities supporting Walking Onions SHOULD also support the
"Connection: keep-alive" and "Keep-Alive" HTTP headers, to avoid
needless reconnections in response to these requests.
Implementors SHOULD be aware of potential denial-of-service
attacks based on open HTTP connections, and mitigate them as
appropriate.

> Note: I thought about using OPTIONS here, but OPTIONS isn't quite
> right for this, since Accept-Vote-Diffs-From does not fit with its
> semantics.

> Note: It might be desirable to support this negotiation for legacy
> votes as well, even before walking onions is implemented.  Doing so
> would allow us to reduce authority bandwidth a little, and possibly
> include microdescriptors in votes for more convenient processing.

<!-- Section 3.3 --> <a id='S3.3'></a>

## A generalized algorithm for voting

Unlike with previous versions of our voting specification, here I'm
going to try to describe pieces the voting algorithm in terms of
simpler voting operations.  Each voting operation will be named and
possibly parameterized, and data will frequently self-describe what
voting operation is to be used on it.

Voting operations may operate over different CBOR types, and are
themselves specified as CBOR objects.

A voting operation takes place over a given "voteable field".  Each
authority that specifies a value for a voteable field MUST specify
which voting operation to use for that field.  Specifying a voteable
field without a voting operation MUST be taken as specifying the
voting operation "None" -- that is, voting against a consensus.

On the other hand, an authority MAY specify a voting operation for
a field without casting any vote for it.  This means that the
authority has an opinion on how to reach a consensus about the
field, without having any preferred value for the field itself.

<!-- Section 3.3.1 --> <a id='S3.3.1'></a>

### Constants used with voting operations

Many voting operations may be parameterized by an unsigned integer.
In some cases the integers are constant, but in

When we encode these constants, we encode them as short strings
rather than as integers.

> I had thought of using negative integers here to encode these
> special constants, but that seems too error-prone.

The following constants are defined:

`N_AUTH` -- the total number of authorities, including those whose
votes are absent.

`N_PRESENT` -- the total number of authorities whose votes are
present for this vote.

`N_FIELD` -- the total number of authorities whose votes for a given
field are present.

Necessarily, `N_FIELD` <= `N_PRESENT` <= `N_AUTH` -- you can't vote
on a field unless you've cast a vote, and you can't cast a vote
unless you're an authority.

In the definitions below, `//` denotes the truncating integer division
operation, as implemented with `/` in C.

`QUORUM_AUTH` -- The lowest integer that is greater than half of
`N_AUTH`.  Equivalent to `N_AUTH // 2 + 1`.

`QUORUM_PRESENT` -- The lowest integer that is greater than half of
`N_PRESENT`.  Equivalent to `N_PRESENT // 2 + 1`.

`QUORUM_FIELD` -- The lowest integer that is greater than half of
`N_FIELD`.  Equivalent to `N_FIELD // 2 + 1`.

We define `SUPERQUORUM_`..., variants of these fields as well, based
on the lowest integer that is greater than 2/3 majority of the
underlying field.  `SUPERQUORUM_x` is thus equivalent to
`(N_x * 2) // 3 + 1`.

    ; We need to encode these arguments; we do so as short strings.
    IntOpArgument = uint / "auth" / "present" / "field" /
         "qauth" / "qpresent" / "qfield" /
         "sqauth" / "sqpresent" / "sqfield"

No IntOpArgument may be greater than AUTH.  If an IntOpArgument is
given as an integer, and that integer is greater than AUTH, then it
is treated as if it were AUTH.

> This rule lets us say things like "at least 3 authorities must
> vote on x...if there are 3 authorities."

<!-- Section 3.3.2 --> <a id='S3.3.2'></a>

### Producing consensus on a field

Each voting operation will either produce a CBOR output, or produce
no consensus.  Unless otherwise stated, all CBOR outputs are to be
given in canonical form.

Below we specify a number of operations, and the parameters that
they take.  We begin with operations that apply to "simple" values
(integers and binary strings), then show how to compose them to
larger values.

All of the descriptions below show how to apply a _single_ voting
operation to a set of votes.  We will later describe how to behave when
the authorities do not agree on which voting operation to use, in our
discussion of the StructJoinOp operation.

Note that while some voting operations take other operations as
parameters, we are _not_ supporting full recursion here: there is a
strict hierarchy of operations, and more complex operations can only
have simpler operations in their parameters.

All voting operations follow this metaformat:

    ; All a generic voting operation has to do is say what kind it is.
    GenericVotingOp = {
        op : tstr,
        * tstr => any,
    }

Note that some voting operations require a sort or comparison
operation over CBOR values.  This operation is defined later in
appendix E; it works only on homogeneous inputs.

<!-- Section 3.3.3 --> <a id='S3.3.3'></a>

### Generic voting operations

<!-- Section 3.3.3.1 --> <a id='S3.3.3.1'></a>

#### None

This voting operation takes no parameters, and always produces "no
consensus".  It is encoded as:

    ; "Don't produce a consensus".
    NoneOp = { op : "None" }

When encounting an unrecognized or nonconforming voting operation,
_or one which is not recognized by the consensus-method in use_, the
authorities proceed as if the operation had been "None".

<!-- Section 3.3.4 --> <a id='S3.3.4'></a>

### Voting operations for simple values

We define a "simple value" according to these cddl rules:

    ; Simple values are primitive types, and tuples of primitive types.
    SimpleVal = BasicVal / SimpleTupleVal
    BasicVal = bool / int / bstr / tstr
    SimpleTupleVal = [ *BasicVal ]

We also need the ability to encode the types for these values:

    ; Encoding a simple type.
    SimpleType = BasicType / SimpleTupleType
    BasicType = "bool" /  "uint" / "sint" / "bstr" / "tstr"
    SimpleTupleType = [ "tuple", *BasicType ]

In other words, a SimpleVal is either an non-compound base value, or is
a tuple of such values.

    ; We encode these operations as:
    SimpleOp = MedianOp / ModeOp / ThresholdOp /
        BitThresholdOp / CborSimpleOp / NoneOp

We define each of these operations in the sections below.

<!-- Section 3.3.4.1 --> <a id='S3.3.4.1'></a>

#### Median

_Parameters_: `MIN_VOTES` (an integer), `BREAK_EVEN_LOW` (a boolean),
`TYPE` (a SimpleType)

    ; Encoding:
    MedianOp = { op : "Median",
                 ? min_vote : IntOpArgument,  ; Default is 1.
                 ? even_low : bool,           ; Default is true.
                 type : SimpleType  }

Discard all votes that are not of the specified `TYPE`. If there are
fewer than `MIN_VOTES` votes remaining, return "no consensus".

Put the votes in ascending sorted order. If the number of votes N
is odd, take the center vote (the one at position (N+1)/2).  If N is
even, take the lower of the two center votes (the one at position
N/2) if `BREAK_EVEN_LOW` is true. Otherwise, take the higher of the
two center votes (the one at position N/2 + 1).

For example, the Median(…, even_low: True, type: "uint") of the votes
["String", 2, 111, 6] is 6. The Median(…, even_low: True, type: "uint")
of the votes ["String", 77, 9, 22, "String", 3] is 9.

<!-- Section 3.3.4.2 --> <a id='S3.3.4.2'></a>

#### Mode

_Parameters_: `MIN_COUNT` (an integer), `BREAK_TIES_LOW` (a boolean),
`TYPE` (a SimpleType)

    ; Encoding:
    ModeOp = { op : "Mode",
               ? min_count : IntOpArgument,   ; Default 1.
               ? tie_low : bool,              ; Default true.
               type : SimpleType
    }

Discard all votes that are not of the specified `TYPE`.  Of the
remaining votes, look for the value that has received the most
votes.  If no value has received at least `MIN_COUNT` votes, then
return "no consensus".

If there is a single value that has received the most votes, return
it. Break ties in favor of lower values if `BREAK_TIES_LOW` is true,
and in favor of higher values of `BREAK_TIES_LOW` is false.
(Perform comparisons in canonical cbor order.)

<!-- Section 3.3.4.3 --> <a id='S3.3.4.3'></a>

#### Threshold

_Parameters_: `MIN_COUNT` (an integer), `BREAK_MULTI_LOW` (a boolean),
`TYPE` (a SimpleType)

    ; Encoding
    ThresholdOp = { op : "Threshold",
                    min_count : IntOpArgument,  ; No default.
                    ? multi_low: bool,          ; Default true.
                    type : SimpleType
    }

Discard all votes that are not of the specified `TYPE`.  Sort in
canonical cbor order.  If `BREAK_MULTI_LOW` is false, reverse the
order of the list.

Return the first element that received at least `MIN_COUNT` votes.
If no value has received at least `MIN_COUNT` votes, then return
"no consensus".

<!-- Section 3.3.4.4 --> <a id='S3.3.4.4'></a>

#### BitThreshold

Parameters: `MIN_COUNT` (an integer >= 1)

    ; Encoding
    BitThresholdOp = { op : "BitThreshold",
                       min_count : IntOpArgument, ; No default.
    }

These are usually not needed, but are quite useful for
building some ProtoVer operations.

Discard all votes that are not of type uint or bstr; construe bstr
inputs as having type "biguint".

The output is a uint or biguint in which the b'th bit is set iff the
b'th bit is set in at least `MIN_COUNT` of the votes.

<!-- Section 3.3.5 --> <a id='S3.3.5'></a>

### Voting operations for lists

These operations work on lists of SimpleVal:

    ; List type definitions
    ListVal = [ * SimpleVal ]

    ListType = [ "list",
                 [ *SimpleType ] / nil ]

They are encoded as:

    ; Only one list operation exists right now.
    ListOp = SetJoinOp

<!-- Section 3.3.5.1 --> <a id='S3.3.5.1'></a>

#### SetJoin

Parameters: `MIN_COUNT` (an integer >= 1).
Optional parameters: `TYPE` (a SimpleType.)

    ; Encoding:
    SetJoinOp = {
       op : "SetJoin",
       min_count : IntOpArgument,
       ? type : SimpleType
    }

Discard all votes that are not lists.  From each vote,
discard all members that are not of type 'TYPE'.

For the consensus, construct a new list containing exactly those
elements that appears in at least `MIN_COUNT` votes.

(Note that the input votes may contain duplicate elements.  These
must be treated as if there were no duplicates: the vote
[1, 1, 1, 1] is the same as the vote [1]. Implementations may want
to preprocess votes by discarding all but one instance of each
member.)

<!-- Section 3.3.6 --> <a id='S3.3.6'></a>

### Voting operations for maps

Map voting operations work over maps from key types to other non-map
types.

    ; Map type definitions.
    MapVal = { * SimpleVal => ItemVal }
    ItemVal = ListVal / SimpleVal

    MapType = [ "map", [ *SimpleType ] / nil, [ *ItemType ] / nil ]
    ItemType = ListType / SimpleType

They are encoded as:

    ; MapOp encodings
    MapOp = MapJoinOp / StructJoinOp

<!-- Section 3.3.6.1 --> <a id='S3.3.6.1'></a>

#### MapJoin

The MapJoin operation combines homogeneous maps (that is, maps from
a single key type to a single value type.)

Parameters:
   `KEY_MIN_COUNT` (an integer >= 1)
   `KEY_TYPE` (a SimpleType type)
   `ITEM_OP` (A non-MapJoin voting operation)

Encoding:

    ; MapJoin operation encoding
    MapJoinOp = {
       op : "MapJoin"
       ? key_min_count : IntOpArgument, ; Default 1.
       key_type : SimpleType,
       item_op : ListOp / SimpleOp
    }

First, discard all votes that are not maps.  Then consider the set
of keys from each vote as if they were a list, and apply
`SetJoin[KEY_MIN_COUNT,KEY_TYPE]` to those lists.  The resulting list
is a set of keys to consider including in the output map.

> We have a separate `key_min_count` field, even if `item_op` has
> its own `min_count` field, because some min_count values (like
> `qfield`) depend on the overall number of votes for the field.
> Having `key_min_count` lets us specify rules like "the FOO of all
> votes on this field, if there are at least 2 such votes."

For each key in the output list, run the sub-voting operation
`ItemOperation` on the values it received in the votes.  Discard all
keys for which the outcome was "no consensus".

The final vote result is a map from the remaining keys to the values
produced by the voting operation.

<!-- Section 3.3.6.2 --> <a id='S3.3.6.2'></a>

#### StructJoin

A StructJoinOp operation describes a way to vote on maps that encode a
structure-like object.

Parameters:
    `KEY_RULES` (a map from int or string to StructItemOp)
    `UNKNOWN_RULE` (An operation to apply to unrecognized keys.)

    ; Encoding
    StructItemOp = ListOp / SimpleOp / MapJoinOp / DerivedItemOp /
        CborDerivedItemOp

    VoteableStructKey = int / tstr

    StructJoinOp = {
        op : "StructJoin",
        key_rules : {
            * VoteableStructKey => StructItemOp,
        }
        ? unknown_rule : StructItemOp
    }

To apply a StructJoinOp to a set of votes, first discard every vote that is
not a map.  Then consider the set of keys from all the votes as a single
list, with duplicates removed.  Also remove all entries that are not integers
or strings from the list of keys.

For each key, then look for that key in the `KEY_RULES` map.  If there is an
entry, then apply the StructItemOp for that entry to the values for that key
in every vote.  Otherwise, apply the `UNKNOWN_RULE` operation to the values
for that key in every vote.  Otherwise, there is no consensus for the values
of this key.  If there _is_ a consensus for the values, then the key should
map to that consensus in the result.

This operation always reaches a consensus, even if it is an empty map.

<!-- Section 3.3.6.3 --> <a id='S3.3.6.3'></a>

#### CborData

A CborData operation wraps another operation, and tells the authorities
that after the operation is completed, its result should be decoded as a
CBOR bytestring.

Parameters: `ITEM_OP` (Any SingleOp that can take a bstr input.)

     ; Encoding
     CborSimpleOp = {
         op: "CborSimple",
         item-op: MedianOp / ModeOp / ThresholdOp / NoneOp
     }
     CborDerivedItemOp = {
         op: "CborDerived",
         item-op: DerivedItemOp,
     }

To apply either of these operations to a set of votes, first apply
`ITEM_OP` to those votes.  After that's done, check whether the
consensus from that operation is a bstr that encodes a single item of
"well-formed" "valid" cbor.  If it is not, this operation gives no
consensus.  Otherwise, the consensus value for this operation is the
decoding of that bstr value.

<!-- Section 3.3.6.4 --> <a id='S3.3.6.4'></a>

#### DerivedFromField

This operation can only occur within a StructJoinOp operation (or a
semantically similar SectionRules). It indicates that one field
should have been derived from another.  It can be used, for example,
to say that a relay's version is "derived from" a relay's descriptor
digest.

Unlike other operations, this one depends on the entire consensus (as
computed so far), and on the entirety of the set of votes.

> This operation might be a mistake, but we need it to continue lots of
> our current behavior.

Parameters:
    `FIELDS` (one or more other locations in the vote)
    `RULE` (the rule used to combine values)

Encoding
    ; This item is "derived from" some other field.
    DerivedItemOp = {
        op : "DerivedFrom",
        fields : [ +SourceField ],
        rule : SimpleOp
    }

    ; A field in the vote.
    SourceField = [ FieldSource, VoteableStructKey ]

    ; A location in the vote.  Each location here can only occur
    ; be referenced from later locations, or from itself.
    FieldSource = "M" ; Meta.
               / "CP" ; ClientParam.
               / "SP" ; ServerParam.
               / "RM" ; Relay-meta
               / "RS" ; Relay-SNIP
               / "RL" ; Relay-legacy

To compute a consensus with this operation, first locate each field described
in the SourceField entry in each VoteDocument (if present), and in the
consensus computed so far.  If there is no such field in the
consensus or if it has not been computed yet, then
this operation produces "no consensus".  Otherwise, discard the VoteDocuments
that do not have the same value for the field as the consensus, and their
corresponding votes for this field.  Do this for every listed field.

At this point, we have a set of votes for this field's value that all come
from VoteDocuments that describe the same value for the source field(s).  Apply
the `RULE` operation to those votes in order to give the result for this
voting operation.

The DerivedFromField members in a SectionRules or a StructJoinOp
should be computed _after_ the other members, so that they can refer
to those members themselves.

<!-- Section 3.3.7 --> <a id='S3.3.7'></a>

### Voting on document sections

Voting on a section of the document is similar to the StructJoin
operation, with some exceptions.  When we vote on a section of the
document, we do *not* apply a single voting rule immediately.
Instead, we first "_merge_" a set of SectionRules together, and then
apply the merged rule to the votes.  This is the only place where we
merge rules like this.

A SectionRules is _not_ a voting operation, so its format is not
tagged with an "op":

    ; Format for section rules.
    SectionRules = {
      * VoteableStructKey => SectionItemOp,
      ? nil => SectionItemOp
    }
    SectionItemOp = StructJoinOp / StructItemOp

To merge a set of SectionRules together, proceed as follows. For each
key, consider whether at least QUORUM_AUTH authorities have voted voted the
same StructItemOp for that key.  If so, that StructItemOp is the
resulting operation for this key.  Otherwise, there is no entry for this key.

Do the same for the "nil" StructItemOp; use the result as the
`UNKNOWN_RULE`.

Note that this merging operation is *not* recursive.

<!-- Section 3.4 --> <a id='S3.4'></a>

## A CBOR-based metaformat for votes.

A vote is a signed document containing a number of sections; each
section corresponds roughly to a section of another document, a
description of how the vote is to be conducted, or both.

    ; VoteDocument is a top-level signed vote.
    VoteDocument = [
        ; Each signature may be produced by a different key, if they
        ; are all held by the same authority.
        sig : [ + SingleSig ],
        lifetime : Lifespan,
        digest-alg : DigestAlgorithm,
        body : bstr .cbor VoteContent
    ]

    VoteContent = {
        ; List of supported consensus methods.
        consensus-methods : [ + uint ],

        ; Text-based legacy vote to be used if the negotiated
        ; consensus method is too old.  It should itself be signed.
        ; It's encoded as a series of text chunks, to help with
        ; cbor-based binary diffs.
        ? legacy-vote : [ * tstr ],

        ; How should the votes within the individual sections be
        ; computed?
        voting-rules : VotingRules,

        ; Information that the authority wants to share about this
        ; vote, which is not itself voted upon.
        notes : NoteSection,

        ; Meta-information that the authorities vote on, which does
        ; not actually appear in the ENDIVE or consensus directory.
        meta : MetaSection .within VoteableSection,

        ; Fields that appear in the client network parameter document.
        client-params : ParamSection .within VoteableSection,
        ; Fields that appear in the server network parameter document.
        server-params : ParamSection .within VoteableSection,

        ; Information about each relay.
        relays : RelaySection,

        ; Information about indices.
        indices : IndexSection,

        * tstr => any
    }

    ; Self-description of a voter.
    VoterSection = {
        ; human-memorable name
        name : tstr,

        ; List of link specifiers to use when uploading to this
        ; authority. (See proposal for dirport link specifier)
        ? ul : [ *LinkSpecifier ],

        ; List of link specifiers to use when downloading from this authority.
        ? dl : [ *LinkSpecifier ],

        ; contact information for this authority.
        ? contact : tstr,

        ; legacy certificate in format given by dir-spec.txt.
        ? legacy-cert : tstr,

        ; for extensions
        * tstr => any,
    }

    ; An indexsection says how we think routing indices should be built.
    IndexSection = {
        * IndexId => bstr .cbor [ IndexGroupId, GenericIndexRule ],
    }
    IndexGroupId = uint
    ; A mechanism for building a single routing index.  Actual values need to
    ; be within RecognizedIndexRule or the authority can't complete the
    ; consensus.
    GenericIndexRule = {
        type: tstr,
        * tstr => any
    }
    RecognizedIndexRule = EdIndex / RSAIndex / BWIndex / WeightedIndex
    ; The values in an RSAIndex are derived from digests of Ed25519 keys.
    EdIndex = {
        type: "ed-id",
        alg : DigestAlgorithm,
        prefix: bstr,
        suffix: bstr
    }
    ; The values in an RSAIndex are derived from RSA keys.
    RSAIndex = {
        type: "rsa-id"
    }
    ; A BWIndex is built by taking some uint-valued field referred to by
    ; SourceField from all the relays that have all of required_flags set.
    BWIndex = {
        type: "bw",
        bwfield: SourceField,
        require_flags: FlagSet,
    }
    ; A flag can be prefixed with "!" to indicate negation.  A flag
    ; with a name like P@X indicates support for port class 'X' in its
    ; exit policy.
    ;
    ; FUTURE WORK: perhaps we should add more structure here and it
    ; should be a matching pattern?
    FlagSet = [ *tstr ]
    ; A WeightedIndex applies a set of weights to a BWIndex based on which
    ; flags the various routers have.  Relays that match a set of flags have
    ; their weights multiplied by the corresponding WeightVal.
    WeightedIndex = {
        type: "weighted",
        source: BwIndex,
        weight: { * FlagSet => WeightVal }
    }
    ; A WeightVal is either an integer to multiply bandwidths by, or a
    ; string from the Wgg, Weg, Wbm, ... set as documented in dir-spec.txt,
    ; or a reference to an earlier field.
    WeightVal = uint / tstr / SourceField
    VoteableValue =  MapVal / ListVal / SimpleVal

    ; A "VoteableSection" is something that we apply part of the
    ; voting rules to.  When we apply voting rules to these sections,
    ; we do so without regards to their semantics.  When we are done,
    ; we use these consensus values to make the final consensus.
    VoteableSection = {
       VoteableStructKey => VoteableValue,
    }

    ; A NoteSection is used to convey information about the voter and
    ; its vote that is not actually voted on.
    NoteSection = {
       ; Information about the voter itself
       voter : VoterSection,
       ; Information that the voter used when assigning flags.
       ? flag-thresholds : { tstr => any },
       ; Headers from the bandwidth file that the
       ? bw-file-headers : {tstr => any },
       ? shared-rand-commit : SRCommit,
       * VoteableStructKey => VoteableValue,
    }

    ; Shared random commitment; fields are as for the current
    ; shared-random-commit fields.
    SRCommit = {
       ver : uint,
       alg : DigestAlgorithm,
       ident : bstr,
       commit : bstr,
       ? reveal : bstr
    }

    ; the meta-section is voted on, but does not appear in the ENDIVE.
    MetaSection = {
       ; Seconds to allocate for voting and distributing signatures
       ; Analagous to the "voting-delay" field in the legacy algorithm.
       voting-delay: [ vote_seconds: uint, dist_seconds: uint ],
       ; Proposed time till next vote.
       voting-interval : uint,
       ; proposed lifetime for the SNIPs and endives
       snip-lifetime: Lifespan,
       ; proposed lifetime for client params document
       c-param-lifetime : Lifespan,
       ; proposed lifetime for server params document
       s-param-lifetime : Lifespan,
       ; signature depth for ENDIVE
       signature-depth : uint,
       ; digest algorithm to use with endive.
       signature-digest-alg : DigestAlgorithm,
       ; Current and previous shared-random values
       ? cur-shared-rand : [ reveals : uint, rand : bstr ],
       ? prev-shared-rand : [ reveals : uint, rand : bstr ],
       ; extensions.
       * VoteableStructKey => VoteableValue,
    };

    ; A ParamSection will be made into a ParamDoc after voting;
    ; the fields are analogous.
    ParamSection = {
       ? certs : [ 1*2 bstr .cbor VoterCert ],
       ? recommend-versions : [ * tstr ],
       ? require-protos : ProtoVersions,
       ? recommend-protos : ProtoVersions,
       ? params : NetParams,
       * VoteableStructKey => VoteableValue,
    }
    RelaySection = {
       ; Mapping from relay identity key (or digest) to relay information.
       * bstr => RelayInfo,
    }

    ; A RelayInfo is a vote about a single relay.
    RelayInfo = {
       meta : RelayMetaInfo .within VoteableSection,
       snip : RelaySNIPInfo .within VoteableSection,
       legacy : RelayLegacyInfo .within VoteableSection,
    }

    ; Information about a relay that doesn't go into a SNIP.
    RelayMetaInfo = {
        ; Tuple of published-time and descriptor digest.
        ? desc : [ uint , bstr ],
        ; What flags are assigned to this relay?  We use a
        ; string->value encoding here so that only the authorities
        ; who have an opinion on the status of a flag for a relay need
        ; to vote yes or no on it.
        ? flags : { *tstr=>bool },
        ; The relay's self-declared bandwidth.
        ? bw : uint,
        ; The relay's measured bandwidth.
        ? mbw : uint,
        ; The fingerprint of the relay's RSA identity key.
        ? rsa-id : RSAIdentityFingerprint
    }
    ; SNIP information can just be voted on directly; the formats
    ; are the same.
    RelaySNIPInfo = SNIPRouterData

    ; Legacy information is used to build legacy consensuses, but
    ; not actually required by walking onions clients.
    RelayLegacyInfo = {
       ; Mapping from consensus version to microdescriptor digests
       ; and microdescriptors.
       ? mds : [ *Microdesc ],
    }

    ; Microdescriptor votes now include the digest AND the
    ; microdescriptor-- see note.
    Microdesc = [
       low : uint,
       high : uint,
       digest : bstr .size 32,
       ; This is encoded in this way so that cbor-based diff tools
       ; can see inside it.  Because of compression and diffs,
       ; including microdesc text verbatim should be comparatively cheap.
       content : encoded-cbor .cbor [ *tstr ],
    ]

    ; ==========

    ; The VotingRules field explains how to vote on the members of
    ; each section
    VotingRules = {
        meta : SectionRules,
        params : SectionRules,
        relay : RelayRules,
        indices : SectionRules,
    }

    ; The RelayRules object explains the rules that apply to each
    ; part of a RelayInfo.  A key will appear in the consensus if it
    ; has been listed by at least key_min_count authorities.
    RelayRules = {
        key_min_count : IntOpArgument,
        meta : SectionRules,
        snip : SectionRules,
        legacy : SectionRules,
    }

<!-- Section 3.5 --> <a id='S3.5'></a>

## Computing a consensus.

To compute a consensus, the authorities first verify that all the votes are
timely and correctly signed by real authorities.  This includes
validating all invariants stated here, and all internal documents.

If they have two votes from an authority, authorities SHOULD issue a
warning, and they should take the one that is published more
recently.

> TODO: Teor suggests that maybe we shouldn't warn about two votes
> from an authority for the same period, and we could instead have a
> more resilient process here, where authorities can update their
> votes at various times over the voting period, up to some point.
>
> I'm not sure whether this helps reliability more or less than it risks it,
> but it worth investigating.

Next, the authorites determine the consensus method as they do today,
using the field "consensus-method".  This can also be expressed as
the voting operation `Threshold[SUPERQUORUM_PRESENT, false, uint]`.

If there is no consensus for the consensus-method, then voting stops
without having produced a consensus.

Note that in contrast with its behavior in the current voting algorithm, the
consensus method does not determine the way to vote on every
individual field: that aspect of voting is controlled by the
voting-rules.  Instead, the consensus-method changes other aspects
of this voting, such as:

    * Adding, removing, or changing the semantics of voting
      operations.
    * Changing the set of documents to which voting operations apply.
    * Otherwise changing the rules that are set out in this
      document.

Once a consensus-method is decided, the next step is to compute the
consensus for other sections in this order: `meta`, `client-params`,
`server-params`, and `indices`.  The consensus for each is calculated
according to the operations given in the corresponding section of
VotingRules.

Next the authorities compute a consensus on the `relays` section,
which is done slightly differently, according to the rules of
RelayRules element of VotingRules.

Finally, the authorities transform the resulting sections into an
ENDIVE and a legacy consensus, as in "Computing an Endive" and
"Computing a legacy consensus" below.

To vote on a single VotingSection, find the corresponding
SectionRules objects in the VotingRules of this votes, and apply it
as described above in "Voting on document sections".

<!-- Section 3.6 --> <a id='S3.6'></a>

## If an older consensus method is negotiated (Transitional)

The `legacy-vote` field in the vote document contains an older (v3,
text-style) consensus vote, and is used when an older consensus
method is negotiated.  The legacy-vote is encoded by splitting it
into pieces, to help with CBOR diff calculation.  Authorities MAY split at
line boundaries, space boundaries, or anywhere that will help with
diffs.   To reconstruct the legacy vote, concatenate the members of
`legacy-vote` in order.  The resulting string MUST validate
according to the rules of the legacy voting algorithm.

If a legacy vote is present, then authorities SHOULD include the
same view of the network in the legacy vote as they included in their
real vote.

If a legacy vote is present, then authorities MUST
give the same list of consensus-methods and the same voting
schedule in both votes.  Authorities MUST reject noncompliant votes.

<!-- Section 3.7 --> <a id='S3.7'></a>

## Computing an ENDIVE.

If a consensus-method is negotiated that is high enough to support
ENDIVEs, then the authorities proceed as follows to transform the consensus
sectoins above into an ENDIVE.

The ParamSections from the consensus are used verbatim as the bodies of the
`client-params` and `relay-params` fields.

The fields that appear in each RelaySNIPInfo determine what goes
into the SNIPRouterData for each relay. To build the relay section,
first decide which relays appear according to the `key_min_count`
field in the RelayRules.  Then collate relays across all the votes
by their keys, and see which ones are listed.  For each key that
appears in at least `key_min_count` votes, apply the RelayRules to
each section of the RelayInfos for that key.

The sig_params section is derived from fields in the meta section.
Fields with identical names are simply copied; Lifespan values are
copied to the corresponding documents (snip-lifetime as the lifespan
for SNIPs and ENDIVEs, and c and s-param-lifetime as the lifespan
for ParamDocs).

To compute the signature nonce, use the signature digest algorithm
to compute the digest of each input vote body, sort those digests
lexicographically, and concatenate and hash those digests again.

Routing indices are built according to named IndexRules, and grouped
according to fields in the meta section.  See "Constructing Indices" below.

> (At this point extra fields may be copied from the Meta section of
> each RelayInfo into the ENDIVERouterData depending on the meta
> document; we do not, however, currently specify any case where this
> is done.)

<!-- Section 3.7.1 --> <a id='S3.7.1'></a>

### Constructing indices

After having build the list of relays, the authorities construct and
encode the indices that appear in the endives.  The voted-upon
GenericIndexRule values in the IndexSection of the consensus say how
to build the indices in the endive, as follows.

An `EdIndex` is built using the IndexType_Ed25519Id value, with the
provided prefix and suffix values.  Authorities don't need to expand
this index in the ENDIVE, since the relays can compute it
deterministically.

An `RSAIndex` is built using the IndexType_RSAId type.  Authorities
don't need to expand this index in the ENDIVE, since the relays can
compute it deterministically.

A `BwIndex` is built using the IndexType_Weighted type. Each relay a
weight equal to some specified bandwidth field in its consensus
RelayInfo.  If a relay is missing any of the `required_flags` in
its meta section, or if it does not have the specified bandwidth
field, that relay's weight becomes 0.

A `WeightedIndex` is built by computing a BwIndex, and then
transforming each relay in the list according to the flags that it
has set.  Relays that match any set of flags in the WeightedIndex
rule get their bandwidths multiplied by _all_ WeightVals that
apply.  Some WeightVals are computed according to special rules,
such as "Wgg", "Weg", and so on.  These are taken from the current
dir-spec.txt.

For both BwIndex and WeightedIndex values, authorities MUST scale
the computed outputs so that no value is greater than UINT32_MAX;
they MUST do by shifting all values right by lowest number of bits
that achieves this.

> We could specify a more precise algorithm, but this is simpler.

Indices with the same IndexGroupId are placed in the same index
group; index groups are ordered numerically.

<!-- Section 3.8 --> <a id='S3.8'></a>

## Computing a legacy consensus.

When using a consensus method that Supports Walking Onions, the
legacy consensus is computed from the same data as the ENDIVE.
Because the legacy consensus format will be frozen once Walking
Onions is finalized, we specify this transformation directly, rather
than in a more extensible way.

The published time and descriptor digest are used directly.
Microdescriptor negotiation proceeds as before.  Bandwidths,
measured bandwidths, descriptor digests, published times, flags, and
rsa-id values are taken from the RelayMetaInfo section.  Addresses,
protovers, versions, and so on are taken from the RelaySNIPInfo. Header
fields are all taken from the corresponding header fields in the
MetaSection or the ClientParamsSection. All parameters are copied
into the net-params field.

<!-- Section 3.9 --> <a id='S3.9'></a>

## Managing indices over time.

> The present voting mechanism does not do a great job of handling
> the authorities

The semantic meaning of most IndexId values, as understood by
clients should remain unchanging; if a client uses index 6 for
middle nodes, 6 should _always_ mean "middle nodes".

If an IndexId is going to change its meaning over time, it should
_not_ be hardcoded by clients; it should instead be listed in the
NetParams document, as the exit indices are in the `port-classes`
field. (See also section 6 and appendix AH.)  If such a field needs
to change, it also needs a migration method that allows clients with
older and newer parameters documents to exist at the same time.
