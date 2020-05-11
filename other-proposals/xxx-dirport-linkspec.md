
# Extending link specifiers to include the directory port

## Motivation

Authorities still expose directory ports, and encourage relays to
use them preferentially for uploading and downloading.  With Walking
Onions, we try to make every kind of "address" a link specifier --
but there is currently no such way to specify directory ports.  We
want authorities to be able to specify a list of link specifiers
that can be used to contact them for uploads and downloads.

## Proposal

We reserve a new link specifier type "dir-url", for use with the
directory system.  This is a variable-length link specifier, containing
a URL prefix.  The only currently supported URL schema is "http://".
Implementations SHOULD ignore unrecognized schemas.  IPv4 and IPv6
addresses MAY be used directory; hostnames are also allowed.
Implementations MAY ignore hostnames and only use raw addresses.

The URL prefix includes everything through the string "tor" in the
directory hierarchy.

A dir-url link specifier SHOULD NOT appear in an EXTEND cell;
implementations SHOULD reject them if they do appear.



