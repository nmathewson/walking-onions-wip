
<!-- Section A.3 --> <a id='SA.3'></a>

## Appendix C: new numbers to assign.

Relay commands:

* We need a new relay command for "FRAGMENT" per proposal 319.

CREATE handshake types:

* We need a type for the NIL handshake.

* We need a handshake type for the new ntor handshake variant.

Link specifiers:

* We need a link specifier for extend-by-index.

* We need a link specifier for dirport URL.

Certificate Types and Key Types:

* We need to add the new entries from CertType and KeyUsage to
  cert-spec.txt, and possibly merge the two lists.

Begin cells:

* We need a flag for Delegated Verifiable Selection.

* We need an extension type for extra data, and a value for indices.

End cells:

* We need an extension type for extra data, a value for indices, a
  value for IPv4 addresses, and a value for IPv6 addresses.

Extensions for decrypted INTRODUCE2 cells:

* A SNIP for the rendezvous point.

Onion key types for decrypted INTRODUCE2 cells:

* An "onion key" to indicate that the onion key for the rendezvous point is
  implicit in the SNIP.

New URLs:

* A URL for fetching ENDIVEs.

* A URL for fetching client / relay parameter documents

* A URL for fetching detached SNIP signatures.

Protocol versions:

(In theory we could omit many new protovers here, since being listed
in an ENDIVE implies support for the new protocol variants.  We're
going to use new protovers anyway, however, since doing so keeps our
numbering consistent.)

We need new versions for these subprotocols:

* _Relay_ to denote support for new handshake elements.

* _DirCache_ to denote support for ENDIVEs, paramdocs, binary diffs, etc.

* _Cons_ to denote support for ENDIVEs

