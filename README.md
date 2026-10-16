# zisk-patch-slop-koala-bear

A patched [`slop-koala-bear`](https://crates.io/crates/slop-koala-bear) 6.2.2,
the KoalaBear field and Poseidon2 configuration of [SP1](https://github.com/succinctlabs/sp1),
that runs the Poseidon2 permutation on the [ZisK](https://github.com/0xPolygonHermez/zisk)
`poseidon2_koalabear` precompile when compiled for a ZisK guest.

It lets a ZisK guest that verifies SP1 proofs (with `sp1-verifier`)
hash its Merkle trees and drive its Fiat-Shamir challenger on the precompile instead of in software.
Outside a ZisK guest, the crate behaves exactly as the original.

## Usage

Patch the crate in the guest's workspace only, pinned by revision:

```toml
[patch.crates-io]
slop-koala-bear = { git = "https://github.com/jestape/zisk-patch-slop-koala-bear.git", rev = "<commit>" }
```

The patch only applies when the version matches the `slop-koala-bear` version the guest resolves (6.2.2),
otherwise Cargo ignores it with a warning and the guest silently runs Poseidon2 in software.
Check that the guest's `Cargo.lock` resolves `slop-koala-bear` from this repository.

The precompile needs a ZisK version that provides it:
the `feature/poseidon2-non-native` branch of ZisK, on top of `1.4.0-alpha`, at the time of writing.

## What changes

Only `src/koala_bear_poseidon2.rs`; the first commit of this repository is the unmodified crate,
so its second commit shows the whole patch.

- `KoalaPerm` becomes a wrapper around the software permutation (now `KoalaPoseidon2`).
  Inside a ZisK guest (`target_os = "zkvm"`, `target_vendor = "zisk"`), its `permute_mut` calls
  `syscall_poseidon2_koalabear`, exported by `ziskos`, on the state as is:
  sixteen KoalaBear elements in Montgomery form, packed two per little-endian `u64`.
  Everywhere else it runs the software permutation.
  There it takes every state the software permutation takes, such as the prover's packed fields;
  inside ZisK, which has no SIMD packing, only `[KoalaBear; 16]`.
- `KoalaBearDegree4Duplex`, the hasher, compressor and challenger configuration of the SP1 verifier,
  builds its permutations as `KoalaPerm`.
- `my_kb_16_perm` still returns the software permutation, so its other users,
  such as `sp1_primitives::poseidon2_hash`, keep running in software.
  In the SP1 compressed verifier they account for about 1.5% of the cost.

The precompile computes the width-16 Poseidon2 permutation with SP1's parameters (`my_kb_16_perm`),
so the results are the same as the software permutation.
Users should still test the two against each other, as a difference would change which proofs verify.

## Upstream

Packaged from [succinctlabs/sp1](https://github.com/succinctlabs/sp1) at commit
`150e6294959f40dbc3ba42eb21c8eccc14c95bc5` (tag `v6.2.2`), directory `slop/crates/koala-bear`.

To move to another SP1 version, import that version's crate unmodified, as the first commit does,
then reapply the patch on top.

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT license](LICENSE-MIT),
at your option, like the original crate.
