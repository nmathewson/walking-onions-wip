
<!-- Section 9 --> <a id='S9'></a>

# Migrating to Walking Onions

This proposal is a major change in the Tor network that will
eventually require the participation of all relays [*], and will make
clients who support it distinguishable from clients that don't.

> [*] Technically, the last relay in the path doesn't need support.

To keep the compatibility issues under control, here is the order in which it
should be deployed on the network.

1. First, authorities should add support for voting on ENDIVEs.

2. Relays may immediately begin trying to download and reconstruct
   ENDIVEs. (Relay versions are public, so they leak nothing by
   doing this.)

3. Once a sufficient number of authorities are voting on ENDIVEs and
   unlikely to downgrade, relays should begin serving parameter documents
   and responding to walking-onion EXTEND and CREATE cells.  (Again,
   relay versions are public, so this doesn't leak.)

4. In parallel with relay support, Tor should also add client
   support for Walking Onions.  This should be disabled by default,
   however, since it will only be usable with the subset of relays
   that support Walking Onions, and since it would make clients
   distinguishable.

5. Once enough of the relays (possibly, all) support Walking Onions,
   the client support can be turned on.  They will not be able to
   use old relays that do not support Walking Onions.

6. Eventually, relays that do not support Walking Onions should not
   be listed in the consensus.

Client support for Walking Onions should be enabled or disabled, at
first, with a configuration option.  Once it seems stable, the
option should have an "auto" setting that looks at a network
parameter. This parameter should NOT be a simple "on" or "off",
however: it should be the minimum client version whose support for
Walking Onions is believed to be correct.

<!-- Section 9.1 --> <a id='S9.1'></a>

## Future work: migrating away from sedentary onions

Once all clients are using Walking Onions, we can take a pass
through the Tor specifications and source code to remove
no-longer-needed code.

Clients should be the first to lose support for old directories,
since nobody but the clients depends on the clients having them.
Only after obsolete clients represent a very small fraction of the
network should relay or authority support be disabled.

Some fields in router descriptors become obsolete with Walking
Onions, and possibly router descriptors themselves should be
replaced with cbor objects of some kind.  This can only happen,
however, after no descriptor users remain.

