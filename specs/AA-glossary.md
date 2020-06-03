<!-- Section A --> <a id='SA'></a>

# Appendices

<!-- Section A.1 --> <a id='SA.1'></a>

## Appendix A: Glossary

I'm going to put a glossary here so I can try to use these terms
consistently.

*SNIP* -- A "Separable Network Index Proof".  Each SNIP contains the
information necessary to use a single Tor relay, and associates the relay
with one or more index ranges. SNIPs are authenticated by the directory
authorities.

*ENDIVE* -- An "Efficient Network Directory with Individually Verifiable
Entries".  An ENDIVE is a collection of SNIPS downloaded by relays,
authenticated by the directory authorities.

*Routing index* -- A routing index is a map from binary strings to relays,
with some given property.  Each relay that is in the routing index is
associated with a single *index range*.

*Index range* -- A range of positions withing a routing index.  Each range
 contains many positions.

*Index position* -- A single value within a routing index.  Every position in
 a routing index corresponds to a single relay.

*ParamDoc* -- A network parameters document, describing settings for the
 whole network.  Clients download this infrequently.

*Index group* -- A collection of routing indices that are encoded in the same
 SNIPs.
