
<!-- Section A.2 --> <a id='SA.2'></a>

## Appendix B: More cddl definions

    ; These definitions are used throughout the rest of the
    ; proposal

    ; Ed25519 keys are 32 bytes, and that isn't changing.
    Ed25519PublicKey = bstr .size 32

    ; Curve25519 keys are 32 bytes, and that isn't changing.
    Curve25519PublicKey = bstr .size 32

    ; 20 bytes or fewer: legacy RSA SHA1 identity fingerprint.
    RSAIdentityFingerprint = bstr

    ; A 4-byte integer -- or to be cddl-pedantic, one that is
    ; between 0 and UINT32_MAX.
    uint32 = uint .size 4

    ; Enumeration to define integer equivalents for all the digest algorithms
    ; that Tor uses anywhere.  Note that some of these are not used in
    ; this spec, but are included so that we can use this production
    ; whenever we need to refer to a hash function.
    DigestAlgorithm = &(
        NoDigest: 0,
        SHA1    : 1,     ; deprecated.
        SHA2-256: 2,
        SHA2-512: 3,
        SHA3-256: 4,
        SHA3-512: 5,
        Kangaroo12-256: 6,
        Kangaroo12-512: 7,
    )

    ; A digest is represented as a binary blob.
    Digest = bstr

    ; Enumeration for different signing algorithms.
    SigningAlgorithm = &(
       RSA-OAEP-SHA1  : 1,     ; deprecated.
       RSA-OAEP-SHA256: 2,     ; deprecated.
       Ed25519        : 3,
       Ed448          : 4,
       BLS            : 5,     ; Not yet standardized.
    )

    PKAlgorithm = &(
       SigningAlgorithm,

       Curve25519: 100,
       Curve448  : 101
    )

    KeyUsage = &(
       ; A master unchangeable identity key for this authority.  May be
       ; any signing key type.  Distinct from the authority's identity as a
       ; relay.
       AuthorityIdentity: 0x10,
       ; A medium-term key used for signing SNIPs, votes, and ENDIVEs.
       SNIPSigning: 0x11,

       ; These are designed not to collide with the "list of certificate
       ; types" or "list of key types" in cert-spec.txt
    )

    CertType = &(
       VotingCert: 0x12,
       ; These are designed not to collide with the "list of certificate
       ; types" in cert-spec.txt.
    )

    LinkSpecifier = bstr
