# moz-gc-experiments

Documentation etc for ongoing wasm gc experiments at Mozilla

[Versions 2 and 3](version2.md) (in progress) add the reftypes proposal, including multiple tables and table manipulation instructions but excluding funcref, to the MVA for GC types.  They are not backward compatible with version 1 due to changes in instruction encoding.

[Version 1](version1.md) (obsolete) is the "minimal viable alpha" (MVA) for GC types - the smallest system that is useful and safe, including a subset of the reftypes proposal.  As of November 26 2018, Firefox Nightly no longer recognizes Version 1.
