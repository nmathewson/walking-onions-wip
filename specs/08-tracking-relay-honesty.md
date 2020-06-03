
<!-- Section 8 --> <a id='S8'></a>

# Tracking Relay honesty

Our design introduces an opportunity for dishonest relay behavior:
since multiple ENDIVEs are valid at the same time, a malicious relay
might choose any of several possible SNIPs in response to a client's
routing index value.

Here we discuss several ways to mitigate this kind of attack.

<!-- Section 8.1 --> <a id='S8.1'></a>

## Defense: index stability

First, the voting process should be designed such that relays do not
needlessly move around the routing index.  For example, it would
_not_ be appropriate to add an index type whose value is computed by
first putting the relays into a pseudorandom order.  Instead, index
voting should be deterministic and tend to give similar outputs for
similar inputs.

This proposal tries to achieve this property in its index voting
algorithms.  We should measure the degree to which we succeed over
time, by looking at all of the ENDIVEs that are valid at any
particular time, and sampling several points for each index to see
how many distinct relays are listed at each point, across all valid
ENDIVEs.

We do not need this stability property for routing indices whose
purpose is nonrandomized relay selection, such as those indices used
for onion service directories.

<!-- Section 8.2 --> <a id='S8.2'></a>

## Defense: enforced monotonicity

Once an honest relay has received an ENDIVE, it has no reason to
keep any previous ENDIVEs or serve SNIPs from them.  Because of
this, relay implementations SHOULD ensure that no data is served
from a new ENDIVE until all the data from an old ENDIVE is
thoroughly discarded.

Clients and relays can use this monotonicity property to keep relays
honest: once a relay has served a SNIP with some timestamp `T`, that
relay should never serve any other SNIP with a timestamp earlier than
`T`.  Clients SHOULD track the most recent SNIP timestamp that they
have received from each of their guards, and MAY track the most
recent SNIP timestamps that they have received from other relays as
well.

<!-- Section 8.3 --> <a id='S8.3'></a>

## Defense: limiting ENDIVE variance within the network.

The primary motivation for allowing long (de facto) lifespans on
today's consensus documents is to keep the network from grinding to
a halt if the authorities fail to reach consensus for a few hours.
But in practice, _if_ there is a consensus, then relays should have
it within an hour or two, so they should not be falling a full day out
of date.

Therefore we can potentially add a client behavior that, within N
minutes after the client has seen any SNIP with timestamp `T`,
the client should not accept any SNIP with timestamp earlier than
`T - Delta`.

Values for N and Delta are controlled by network parameters
(`enforce-endive-dl-delay-after` and `allow-endive-dl-delay`
respectively in appendix C).  N should be about as long as we expect
it to take for a single ENDIVE to propagate to all the relays on the
network; Delta should be about as long as we would like relays to go
between updating ENDIVEs under ideal circumstances.
