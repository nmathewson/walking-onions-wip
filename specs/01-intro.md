
<!-- Section 1 --> <a id='S1'></a>

# Introduction: A Specification for Walking Onions

In Proposal 300, I introduced Walking Onions, a design for scaling
Tor and simplifying clients, by removing the requirement that every
client know about every relay on the network.

This proposal will elaborate on the original Walking Onions idea,
and should provide enough detail to allow multiple compatible
implementations. In this introduction, I'll start by summarizing the
key ideas of Walking Onions, and then outline how the rest of this
proposal will be structured.

<!-- Section 1.1 --> <a id='S1.1'></a>

## Remind me about Walking Onions again?

With Tor's current design, every client downloads and refreshes a
set of directory documents that describe the directory authorities'
views about every single relay on the Tor network.  This requirement
makes directory bandwidth usage grow quadratically, since the
directory size grows linearly with the number of relays, and it is
downloaded a number of times that grows linearly with the number of
clients.  Additionally, low-bandwidth clients and bootstrapping
clients spend a disproportionate amount of their bandwidth loading
directory information.

With these drawbacks, why does Tor still require clients to
download a directory?  It does so in order to prevent attacks that
would be possible if clients let somebody else choose their
paths through the network, or if each client chose its paths from a
different subset of relays.

Walking Onions is a design that resists these attacks without
requiring clients ever to have a complete view of the network.

You can think of the Walking Onions design like this: Imagine that with
the current Tor design, the client covers a wall with little pieces
of paper, each representing a relay, and then throws a dart at the wall
to pick a relay.  Low-bandwidth relays get small pieces of paper;
high-bandwidth relays get large pieces of paper.  With the Walking
Onions design, however, the client throws its dart at a _blank
wall_, notes the position of the dart, and asks for the relay whose
paper _would be_ at that position on a "standard wall".  These
"standard walls" are mapped out by directory authorities in advance,
and are authenticated in such a way that the client can receive a
proof of a relay's position on the wall without actually having to
know the whole wall.

Because the client itself picks the position on the wall, and
because the authorities must vote together to build a set of
"standard walls", nobody else controls the client's path through the
network, and all clients can choose their paths in the same way.
But since clients only probe one position on the wall at a time,
they don't need to download a complete directory.

(Note that there has to be more than one wall at a time: the client
throws darts at one wall to pick guards, another wall to pick
middle relays, and so on.)

In Walking Onions, we call a collection of standard walls an
"ENDIVE" (Efficient Network Directory with Individually Verifiable
Entries).  We call each of the individual walls a "routing index",
and we call each of the little pieces of paper describing a relay and
its position within the routing index a "SNIP" (Separable Network
Index Proof).

For more details about the key ideas behind Walking Onions, see
proposal 300.  For more detailed analysis and discussion, see
"Walking Onions: Scaling Anonymity Networks while Protecting Users"
by Komlo, Mathewson, and Goldberg.

<!-- Section 1.2 --> <a id='S1.2'></a>

## The rest of this document

This proposal is unusually long, since Walking Onions touches on many
aspects of Tor's functionality.  It requires changes to voting,
directory formats, directory operations, circuit building, path
selection, client operations, and more.  These changes are described in the
sections listed below.

Here in section 1, we briefly reintroduce Walking Onions, and talk
about the rest of this proposal.

Section 2 will describe the formats for ENDIVEs, SNIPs, and related
documents.

Section 3 will describe new behavior for directory authorities as
they vote on and produce ENDIVEs.

Section 4 describes how relays fetch and reconstruct ENDIVEs from
the directory authorities.

Section 5 has the necessary changes to Tor's circuit extension
protocol so that clients can extend to relays by index position.

Section 6 describes new behaviors for clients as they use Walking
Onions, to retain existing Tor functionality for circuit construction.

Section 7 explains how to implement onion services using Walking
Onions.

Section 8 describes small alterations in client and relay behavior
to strengthen clients against some kinds of attacks based on relays
picking among multiple ENDIVEs, while still making the voting
system robust against transient authority failures.

Section 9 closes with a discussion of how to migrate from the
existing Tor design to the new system proposed here.

<!-- Section 1.2.1 --> <a id='S1.2.1'></a>

### Appendices

Additionally, this proposal has several appendices:

Appendix A defines commonly used terms.

Appendix B provides definitions for CDDL grammar productions that
are used elsewhere in the documents.

Appendix C lists the new elements in the protocol that will require
assigned values.

Appendix D lists new network parameters that authorities must vote
on.

Appendix E gives a sorting algorithm for a subset of the CBOR object
representation.

Appendix F gives an example set of possible "voting rules" that
authorities could use to produce an ENDIVE.

Appendix G lists the different routing indices that will be required
in a Walking Onions deployment.

Appendix H discusses partitioning TCP ports into a small number of
subsets, so that relays' exit policies can be represented only as
the group of ports that they support.

Appendix Z closes with acknowledgments.

<!-- Section 1.2.2 --> <a id='S1.2.2'></a>

### Related proposals

The following proposals are not part of the Walking Onions proposal,
but they were written at the same time, and are either helpful or
necessary for its implementation.

318-limit-protovers.md restricts the allowed version numbers for
each subprotocol to the range 0..63.

319-wide-everything.md gives a general mechanism for splitting relay
commands across more than one cell.

320-tap-out-again.md attempts to remove the need for TAP keys in
the HSv2 protocol.

321-happy-families.md lets families be represented with a single
identifier, rather than a long list of keys

322-dirport-linkspec.md allows a directory port to be represented
with a link specifier.
