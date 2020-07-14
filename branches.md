# IPFS × Nix Branches and Internal Changes

During the development of the IPFS x Nix integration, we have produced a number of branches containing different sets of features to facilitate the review, discussion, and upstreaming process with the Nix community. The three branches are:

- [`ipfs-binary-cache-develop`](https://github.com/obsidiansystems/nix/tree/ipfs-binary-cache-develop)
- [`git-object-develop`](https://github.com/obsidiansystems/nix/tree/git-objects-develop)
- [`ipfs-develop`](https://github.com/obsidiansystems/nix/tree/ipfs-develop)

We also created [`ipfs-master`](https://github.com/obsidiansystems/nix/tree/ipfs-master) as a release-tracking branch, functionally equivalent to `ipfs-develop`.
This is the branch targeted by the tutorial in this repo.

At a glance, `ipfs-binary-cache-develop` contains IPFS integration with existing data and `git-object-develop` contains the work on git hashing.
Finally, `ipfs-develop` is the combination of the two, and in integrating them, it contains additional refactors.


## Branch `ipfs-binary-cache-develop`

First, we took https://github.com/NixIPFS/nix/tree/ipfs_binary_cache_legacy as our starting point. We did heavily extend it, but still, it was wonderful to start from something that already works. Thanks for kicking things off!

Unlike the other branches, this one has no data model changes.
There are no new hashing schemes or data structures except for within the implementation of the IPFS store, which is opaque to the rest of Nix.

The original version was a subclass `BinaryCacheStore`, locked into the file-based structure, but we changed it to take advantage of the IPLD format to structure data instead of using plain text files.
This almost eliminates the parsing Nix has to do, and allows store path metadata to directly reference the metadata of reference store paths, rather than just including the store path as indirect reference.
This is great for things like recursive pinning and various graph algorithms.

The work done here remains essentially a trust- and coordination-based scheme, in that there's no explicit correspondence between IPFS hashes and Nix store paths (which is the thing that the user would query).

This means all queries need to consult the mapping first, even ones for "fixed outputs" which are content-addressed in principle, so we are taking advantage of IPFS as a means to distribute data, but not as a way operate without trust and authority.
From a UX standpoint, it also means that there is a need for configuration as users must authorize mappings they trust, or at least consent to a default configuration (say with one mapping correseponding to e.g. `cache.nixos.org`).

In other words, IPFS caching with this branch is fundamentally similar to the other forms of caching we have today.
That said, that similarity comes with benefits too.
Because the data model is unchanged, working with existing data/nixpkgs without conversions like `nix make-content-addressable` is guaranteed.
This is certainly a benefit at least in the a short-term, as it means everybody could, in principle, start taking advantage of IPFS with their existing programs/data, both downloading builds and uploading builds and trust mappings.

## Branch `git-objects-develop`

In this branch we added support for hashing and verifying filesystem data as git does with its tree objects and blob objects.
We paved the way for this change by having already upstreamed a change replacing `bool`s to track how to ingest files with a `FileIngestionMethod` enumeration.
We could just add a new `Git` tag, and give it a corresponding "git:" prefix, to extend the existing schema with this new method.

With this change, we now have the tools to indicate *intent* with Nix references to git repos.
Not only can all store types use the git hashing, but store paths can be based off of it.
This means one can, from a git repo, compute the store path Nix *ought* to have, and derivations consuming the store path, all before Nix actually sees the data.
This opens the door to many future experiments, but the one we are chiefly concerned with today is data exchange without trust, indirection, or configuration via IPFS.

## Branch `ipfs-develop`

This branch is a combination of the changes in the two previous, but also contains much additional work to fully utilize the combination of Git tree hashing and IPFS.

The first major change is that we want Nix to directly look up CIDs (namely git tree hashes) in IPFS — CIDs that themselves are the intent hence the word "directly".
As mentioned above, store paths are computed from other metadata in ways that are hard to invert.
But Nix's internal interfaces use the store path, then that underlying metadata is effectively unavaible for the IPFS store.

The solution is actually is independent of our new IPFS and git hash support, and a change that perhaps we should upstream first.
We made a `StorePathDescriptor` data type which contains exactly the more meaningful data needed to compute store paths for content addressed data, and refactored the `Store` interface such that many store path parameters now take either a store path or a store path description.
This conveys the intent all the way to where it is needed - in our case, the IFPS daemon.
We made a `nix ensure-ca` command to demo this functionality to the end user, and we use it in our tutorial, but our end goal is to likewise adjust the CLI so that anywhere a store path is accepted, a (textual representation of a) store path descriptor can be provided instead.

The second major change is that we also wanted to handle data with references, including self-references, trustlessly, with IPFS.
This is the general case of data in Nix: references to other store paths are how dependencies work, and self-references are just an unfortunately artifact of the way most extant software is installed.

Nix already has support for this as demonstrated in the experimental `nix make-content-addresseable` command, so the natural thing to do is combine it with our newly-added support for git tree objects as a foundation for trustlessness.
We did this by putting the relevant metadata:
 - reference to other store paths, required to themselves be this format and represented with a `(Name, CID)` pair
 - a boolean for whether there is a self-reference
 - and our own name, to help identify self-references (the length matters)
in a wrapping IPLD object; the CID of this object is the CID used for the child references.
We should also store the store directory path; we will fix that soon.
Nix already provided a special normalizer to null-out self references, so we used that when making the underlying git objects, both to hash and transfer, with the self-references being restored on the other end.
There are a few improvements we should make, such as:
 - Only storing our own name if there is self-reference
 - Removing the references padded on the end by the Nix self-reference-removing hashes, so in the no-self-references case the git object can be directly unpacked.
And we hope to make those soon.

Lastly, there is one ramification worth pointing out that's different between the way we handle references and the extant methods in Nix, which is very useful for incremental data transfer such as with IPFS.
With the extant methods, the hash of the normalized, padded data and other metadata is concatenated, and then that concatenation is turned into the store path.
With our message, it's the CID of the total metadata that is turned into the store path—there is an extra hashing step.
What that means is store path descriptor for the existing methods can be arbitrary big with references, while ours is just a CID and a name.
This makes it much easier to use as an opaque reference for computers and humans alike.
