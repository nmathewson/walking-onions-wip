
<!-- Section A.8 --> <a id='SA.8'></a>

## Appendix H: Choosing good clusters of exit policies

With Walking Onions, we cannot easily support all the port
combinations [*] that we currently allow in the "policy summaries"
that we support in microdescriptors.

> [*] How many "short policy summaries" are there? The number would be
> 2^65535, except for the fact today's Tor doesn't permit exit policies to
> get maximally long.

In the Walking Onions whitepaper
(https://crysp.uwaterloo.ca/software/walkingonions/) we noted in
section 6 that we can group exit policies by class, and get down to
around 220 "classes" of port, such that each class was either
completely supported or completely unsupported by every relay.  But
that number is still impractically large: if we need ~11 bytes to
represent a SNIP index range, we would need an extra 2320 bytes per
SNIP, which seems like more overhead than we really want.

We can reduce the number of port classes further, at the cost of
some fidelity.  For example, suppose that the set {https,http} is
supported by relays {A,B,C,D}, and that the set {ssh,irc} is
supported by relays {B,C,D,E}.  We could combine them into a new
port class {https,http,ssh,irc}, supported by relays {B,C,D} -- at
the expense of no longer being able to say that relay A supported
{https,http}, or that relay E supported {ssh,irc}.

This loss would not necessarily be permanent: the operator of relay
A might be willing to add support for {ssh,irc}, and the operator of
relay E might be willing to add support for {https,http}, in order
to become useful as an exit again.

(We might also choose to add a configuration option for relays to
take their exit policies directly from the port classes in the
consensus.)

How might we select our port classes?  Three general categories of
approach seem possible: top-down, bottom-up, and hybrid.

In a top-down approach, we would collaborate with authority and exit
operators to identify _a priori_ reasonable classes of ports, such
as "Web", "Chat", "Miscellaneous internet", "SMTP", and "Everything
else".  Authorities would then base exit indices on these classes.

In a bottom-up approach, we would find an algorithm to run on the
current exit policies in order to find the "best" set of port
classes to capture the policies as they stand with minimal loss.
(Quantifying this loss is nontrivial: do we weight by bandwidth? Do
we weight every port equally, or do we call some more "important"
than others?)

> See exit-analysis for an example tool that runs a greedy algorithm
> to find a "good" partition using an unweighted,
> all-ports-are-equal cost function.  See the files
> "greedy-set-cov-{4,8,16}" for examples of port classes produced
> by this algorithm.

In a hybrid approach, we'd use top-down and bottom-up techniques
together. For example, we could start with an automated bottom-up
approach, and then evaluate it based feedback from operators.  Or we
could start with a handcrafted top-down approach, and then use
bottom-up cost metrics to look for ways to split or combine those
port classes in order to represent existing policies with better
fidelity.

