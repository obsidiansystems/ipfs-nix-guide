# IPFS × Nix Tutorial

Welcome to our walkthrough of both Milestones 1 and 2 of our IPFS × Nix integration.

As you proceed, keep in mind that while parts of this work have already been upstreamed, we plan to incorporate these ideas organically and progressively in Nix, as we continue to optimize how the two ecosystems fit together.

## Setup

Before beginning our tour of new features, we should ensure:

* We have a working installation of IPFS
* We build Nix from the branch in which this integration is being done
* We create separate temporary locations for Nix's state (store, db, etc) to avoid corrupting our normal one with values that only the IPFS × Nix version of Nix will currently understand.


### Make sure IPFS is installed

If you are using NixOS, you can activate IPFS by editing your `/etc/nix/configuration.nix` with the following addition:
```nix
services.ipfs = {
  enable = true;
};
```
If you're using another platform, please install IPFS by following the instructions at https://docs.ipfs.io/install/.
Ensure the IPFS daemon is running.

### Get a copy of our version of Nix

Let's setup a working directory:
```console
$ cd $(mktemp -d)
```

In case you already have Nix installed, you can build this version of Nix from our fork:

***TODO***: Update ipfs-master for milestone 2
```console
$ git clone -b ipfs-master https://github.com/obsidiansystems/nix
$ nix-build -A defaultPackage.x86_64-linux # or x86_64-darwin
$ nix-shell -p ./result
```

Alternatively, you can obtain a prebuilt binary via IPFS:

```console
$ mkdir -p $PWD/nix/bin
$ ipfs get /ipfs/QmP3dcaeV1i6i3nFrdhBmqgTgiC9aRJ7Kd51dA1vmPChTT -o $PWD/nix/bin/nix
$ chmod +x $PWD/nix/bin/nix
$ ln -s ./nix $PWD/nix/bin/nix-store
$ ln -s ./nix $PWD/nix/bin/nix-build
$ ln -s ./nix $PWD/nix/bin/nix-instantiate
$ export PATH=$PWD/nix/bin:$PATH
```

> N.B. you might want to try connecting to Obsidian's IPFS node with
> ```console
> ipfs swarm connect '/dns4/obsidian.webhop.org/tcp/4001/ipfs/QmSdseKbhoRJiswUj9R9NytA1aC1tVSqq1LAw3fGLvSn91'
> ```

### Creating an alternative store for the sake of this tutorial

The last step of the preparation is creating a temporary Nix store.
This is primarily required for two reasons:

* We'll use a version of Nix - different from any released version you might have installed - that supports new data formats.
  Sharing the the same store with a released version of Nix will corrupt the store from the perspective of the released Nix which doesn't know about the new formats.

* The normal location, being in `/nix`, probably requires elevated permissions on your machine to create or modify.
  We don't want to ask you for elevated permissions for a demo, so we'll avoid any filesystem location that might need them.
  (Of course, if you are a Nix expert and want to set up e.g. a trial deployment in multiple VMs, you can still use whatever configuration and directories you'd like)

So, let's create a local directory to work as our temporary store:

```console
$ mkdir -p $PWD/nix/etc
```

We'll set up the variables that make Nix use our newly defined store:
```console
$ export NIX_CONF_DIR=$PWD/nix/etc
$ export NIX_REMOTE=local?real=$PWD/nix/store\&store=/nix/store
$ export NIX_STATE_DIR=$PWD/nix/var
$ export NIX_LOG_DIR=$PWD/nix/var/log
$ export NIX_DATA_DIR=$PWD/nix/share
```
And also to avoid the other configurations/caches in case the format changed
```
$ unset XDG_CONFIG_DIRS
$ export XDG_CACHE_HOME=$PWD/.cache
$ export XDG_CONFIG_HOME=$PWD/.config
$ export XDG_CACHE_HOME=$PWD/.local/share
```

and silence errors/warnings for using unstable features
```console
$ echo "experimental-features = nix-command flakes ca-references ca-derivations" > "$NIX_CONF_DIR"/nix.conf
```

> In a future release that's closer to being upstreamed, you will also see "ipfs" in that list.
This will allow us to continue iterating upstream.

Also, to allow us to evaluate Nixpkgs, we need some of Nix's `corepkgs`:

```console
$ mkdir -p $PWD/nix/share/nix/corepkgs
$ curl https://raw.githubusercontent.com/obsidiansystems/nix/ipfs-master/corepkgs/derivation.nix -o $PWD/nix/share/nix/corepkgs/derivation.nix
$ curl https://raw.githubusercontent.com/obsidiansystems/nix/ipfs-master/corepkgs/fetchurl.nix -o $PWD/nix/share/nix/corepkgs/fetchurl.nix
```
(We're getting them from our branch, but do note we haven't modified them and have no plans to do so)

Now we're ready to dive into the changes in Nix and its new capabilities.

## Milestone 1

### Part 1. Better Git × Nix integration

#### Hash file data like Git

To ensure smooth cooperation between IPFS and Nix, one of our first objectives was to add native support for the Git protocol: in particular we want Nix to be able to hash Git data with the same hashes that Git tree would use.

Let's clone an example repo, go to a commit, and ask Git for the hash:
```console
$ git clone https://github.com/ipfs/ipfs
Cloning into 'ipfs'...
$ git -C ipfs checkout 14a65f594e922bac9f31b89a4271df73fb4677a4
...
HEAD is now at 14a65f5 Merge pull request #459 from johnnymatthews/master
$ git -C ipfs rev-parse HEAD:
fd296e9dc29fc3257aae13cee9bac1bfdc14e7bf
```
> Note that we are asking the hash of `HEAD:` instead of just `HEAD`, because we are interested in the hash of the tree pointed to by the commit, not the hash of the commit itself.
It is crucial that Nix can validate any hash it uses internally, and since it is only storing the contents of one commit and not the whole history, it can verify the tree hash from the contents of the newly created store path.

Let's take note of that hash: we'll now demonstrate how we can store the contents of that directory in the Nix store under a path derived from that hash.

Since the `.git` directory isn't part of the content proper, we have to remove it before inserting the outer directory into the Nix store if we want to end up with the same hash.
In this case we'll make a copy because we'll reuse the original folder later.

```
$ cp -r ipfs/ ipfs-no-git/
$ rm -rf ipfs-no-git/.git
```

Now, we can add those files to the store, but using Git-style hashing (via the new `--git` flag).
Note that we have to specify the name separately:

```console
$ nix add-to-store --git ./ipfs-no-git --name source
/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source
```
> We pass `--name source` so that the store path will exactly match some other paths we get later in the tutorial.
> If we didn't pass `--name`, Nix would choose it based off the path we are adding from.

You might be wondering why the store path doesn't look like the same hash; that is because the store path in Nix, beside being truncated, depends on many other pieces of information other than the hash.
We can nonetheless find the original hash using the command `nix path-info` and checking the `ca` field:

```console
$ nix path-info /nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source --json | jq
[
  {
    "path": "/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source",
    "narHash": "sha256-uIr1qotOcueLPXGNOTArfNSABsA3wAXrz4EqAQxJ8lI=",
    "narSize": 237552,
    "references": [],
    "ca": "fixed:git:sha1:pzki9p5zq6xfkkhkmrx2bhwzqafnwagx",
    "registrationTime": 1594048212
  }
]
```

They're still not equal, but that's just a matter of how it's encoded to text.
If we change it to Base16:
```
$ nix to-base16 'sha1:pzki9p5zq6xfkkhkmrx2bhwzqafnwagx'
fd296e9dc29fc3257aae13cee9bac1bfdc14e7bf
```
it's the same hash we started from!

#### Fetch a repo and store it in Nix according to its tree hash

Nix has a builtin function, `builtin.fetchGit`, that is used to reference a Git repo in a Nix expression.
There's also `builtin.fetchTree`, which is slightly more general, and let's you specify the type of archive to retrieve (Git, Mercurial, etc).
We made this family of functions support Git tree hashes too, so as to keep everything easy about handling git repos already.

To include a git repo in your Nix code, you could use `builtins.fetchTree` specifying `type = "git"` and the desired hash:

```console
$ nix eval --expr "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })"
{
  outPath = "/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source";
  shortTreeHash = "fd296e9";
  submodules = false;
  treeHash = "fd296e9dc29fc3257aae13cee9bac1bfdc14e7bf";
}
```

If you don't know the hash, you can use the `gitIngestion` attribute, to tell Nix to use the tree hash after it retrieves the default branch:


```console
$ nix eval --impure --expr "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; gitIngestion = true; })"
{
  lastModified = 1595975530;
  lastModifiedDate = "20200728223210";
  narHash = "sha256-iLWDyG6ZMQ8+Vl824adTSFyVxVeCPVz/ADG0ndw5+KU=";
  outPath = "/nix/store/f0359vrawiwlrida4yrci4gr4rpmrk8d-source";
  rev = "80d32a72aa00c01f6536d4745e89afa9407868a7";
  revCount = 382;
  shortRev = "80d32a7";
  shortTreeHash = "987ee17";
  submodules = false;
  treeHash = "987ee17d4096620f5e32faf1e28630b67a9e966b"
}
```
The two results are different, because the default branch now points to a newer commit, but we could do the same procedure as before to see that the `treeHash` and `outPath` match.

### Part 2. IPFS × Nix integration

### Export from Nix store to IPFS

Now, let's put data in IPFS, using the Git hashes to get a CID without extra state, configuration, or trust.

> The lack of these 3 potential stumbling blocks will help users of IPFS and Nix immediately start sharing data with the IPFS network.
> And the use of non-Nix-specific CIDs will allow them to share to and from non-Nix users interested in the same data for other reasons.
> We hope that this ease of use and leveraging of network effects spanning multiple communities will make our work easier to adopt.

Copying data into IPFS is as simple as passing the `--to` flag like in the following example:

``` bash
$ nix copy /nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source --to ipfs://
```

> Note that Nix already knows this path's IPFS hash, so there's no other information to add after the `//`.

We can verify the successful upload via:
```
$ ipfs dag get f01781114$(git -C ipfs rev-parse HEAD:) | jq
{
  ".github": {
    "hash": {
      "/": "baf4bcffskskovpj2z3gkxkilpxtiufwsr66cpqa"
    },
    "mode": "40000"
  },
  "LICENSE": {
    "hash": {
      "/": "baf4bcfbm6da4fjvbbwv344tiwfpoqg6cyyjm2sq"
    },
    "mode": "100644"
  },
  "README.md": {
    "hash": {
      "/": "baf4bcfedtbce2try2jrnn4njxw7ci2s7uphblfq"
    },
    "mode": "100644"
  },
  "papers": {
    "hash": {
      "/": "baf4bcfaahjflujisowyymoxgcnvpyelc6a3b67a"
    },
    "mode": "40000"
  }
}
```

> Notice that the string `f01781114` in the previous command is the prefix which turns a SHA1, base16 Git hash into a valid CID, according to the [CID (Content IDentifier) Specification](https://github.com/multiformats/cid).

> Nix has a concept of temporary GC roots.
(GC roots are Nix store paths that are protected from garbage collection and likewise protect anything they reference, transitively.)
> IPFS has a similar concept called pins, which are files that are guaranteed to remain in the user machine.
> They are automatically mapped from one to the other — when uploaded, a Nix temporary GC root is marked as an IPFS pin.

#### Directly fetch a repo and put it in IPFS
We can now combine the two previous ideas to fetch the content of a repo and store them directly in IPFS:

```
$ nix eval --store ipfs:// --expr "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })"
```

Alternatively, if you don't know the tree hash ahead of time:

```console
nix eval --store ipfs:// --impure --expr "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; gitIngestion = true; })"
```

#### Examine git data in IPLD

We can inspect the Git repo directly with the command `ipfs dag get`, which gives us a JSON object that looks just like our directory:

```
$ ipfs dag get f01781114$(git -C ipfs rev-parse HEAD:) | jq
```

```json
{
  ".github": {
    "hash": {
      "/": "baf4bcffskskovpj2z3gkxkilpxtiufwsr66cpqa"
    },
    "mode": "40000"
  },
  "LICENSE": {
    "hash": {
      "/": "baf4bcfbm6da4fjvbbwv344tiwfpoqg6cyyjm2sq"
    },
    "mode": "100644"
  },
  "README.md": {
    "hash": {
      "/": "baf4bcfedtbce2try2jrnn4njxw7ci2s7uphblfq"
    },
    "mode": "100644"
  },
  "papers": {
    "hash": {
      "/": "baf4bcfaahjflujisowyymoxgcnvpyelc6a3b67a"
    },
    "mode": "40000"
  }
}
```

#### Import from IPFS in Nix

Now, we can try to get a Git repo directly from IPFS.

First, we need to delete the old path, to make sure that the data isn't cached locally.
```console
$ nix-store --delete /nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source
finding garbage collector roots...
deleting '/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source'
deleting '/nix/store/trash'
deleting unused links...
note: currently hard linking saves -0.00 MiB
1 store paths deleted, 0.22 MiB freed
```

Then we can copy that path back from IPFS, using `ipfs://` as a substituter.
```console
$ nix eval --expr "(builtins.fetchTree { type = \"git\"; url = \"http://example.com\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })" --substituters ipfs://
```

> `--substituters` and `--store` seem really similar right?
> Well they are!
> `--store` specifies the "working" store (as in "working directory") where we will end up putting everything we get.
> `--substituters` specifies other stores (single CLI arg, space separated list) which we can ask to see if they can provide the path we want.

#### Store items with references

In general, an artifact that is stored in the Nix store contains references to other artifacts.
We need to support this mechanism in IPFS too!
We can take whatever artifact we want, convert it in a IPFS-compatible form, and then copy it to IPFS.
This will create a set of rewrites that maps each component referenced in the Nix build to the corresponding address in IPFS.

As an example, let's build the [GNU Hello](https://www.gnu.org/software/hello/) program, which is packaged in `nixpkgs`.

First, we'll manually clone just one commit of Nixpkgs, because it is an extremely large repo.
```console
$ mkdir nixpkgs
$ git -C nixpkgs init
Initialized empty Git repository in /tmp/tmp.ozeGNsjrHZ/nixpkgs/.git/
$ git -C nixpkgs remote add origin https://github.com/NixOS/nixpkgs
$ git -C nixpkgs fetch origin 43b58960cf0b8b5f891cc3e045ca7cf7cf79c91c --depth=1
...
 * branch              43b58960cf0b8b5f891cc3e045ca7cf7cf79c91c -> FETCH_HEAD
$ git -C nixpkgs checkout FETCH_HEAD
Updating files: 100% (22064/22064), done.
...
HEAD is now at 43b58960 Merge pull request #92568 from r-ryantm/auto-update/shortwave
```

Now, let's "build" and run it.
```console
$ nix build -f ./nixpkgs hello
$ nix shell ./result -c hello
Hello, world!
```

> If the "build" looks a bit quick or terse, this is because by default (i.e. if not specified in that `nix.conf` file we made at the beginning) Nix trusts `cache.nixos.org` as a substituter.
> `cache.nixos.org` provides builds produced as part of Nixpkgs' own CI.
> You can pass `--substituters ''` to force a local build instead, if you like.

Let's convert it to a content addressed path, via the `make-content-addressable` command, and take a look at the rewritten paths:
```console
$ nix make-content-addressable --ipfs -r ./result --json | jq
{
  "rewrites": {
    "/nix/store/y8n2b9nwjrgfx3kvi3vywvfib2cw5xa6-libunistring-0.9.10": "/nix/store/1ffwrp4k258jjj8xkhqj4v0zikmxzps0-libunistring-0.9.10",
    "/nix/store/fhg84pzckx2igmcsvg92x1wpvl1dmybf-libidn2-2.3.0": "/nix/store/cmjfcwkvjfbnxwls1mrf22g0zj6zanlf-libidn2-2.3.0",
    "/nix/store/bqbg6hb2jsl3kvf6jgmgfdqy06fpjrrn-glibc-2.30": "/nix/store/hd1glw944xdb49929h57ix8fg7x1drcf-glibc-2.30",
    "/nix/store/90lq1fg09l7qaav1yllb0y8kga09q8ji-hello-2.10": "/nix/store/ilzfb1jrrg6nf8s09k3ljq7ksqx31qi9-hello-2.10"
  }
}
```

This command takes a built Nix path and rewrites it so that it is addressed based on the contents of the path.
The `--ipfs` flag tells it to do so in a way that works well with IPFS.

Once we have this, it can be copied directly to IPFS:

```console
$ result=/nix/store/ilzfb1jrrg6nf8s09k3ljq7ksqx31qi9-hello-2.10
$ nix copy $result --to ipfs://
```

You can see what metadata Nix stores about this path:

```console
$ nix path-info $result --json | jq
```
```json
[
  {
    "path": "/nix/store/ilzfb1jrrg6nf8s09k3ljq7ksqx31qi9-hello-2.10",
    "narHash": "sha256-EAZULx5G0isAhLzMo6oBO/y8NWRGlLgNiIbZMb/oUY8=",
    "narSize": 205968,
    "references": [
      "/nix/store/hd1glw944xdb49929h57ix8fg7x1drcf-glibc-2.30"
    ],
    "ca": "ipfs:f0171122025ddb2f2ff6a6bbfe57a1137ad15d3e0ad068cb137e1570501c7ddcc24c19ebc",
    "registrationTime": 1594393175
  }
]
```

The `ca` field is just a CID that is put directly into IPFS.
You can verify this through IPFS commands:

```console
$ cid=f0171122025ddb2f2ff6a6bbfe57a1137ad15d3e0ad068cb137e1570501c7ddcc24c19ebc
$ ipfs dag get $cid | jq
```

```json
{
  "cid": {
    "/": "baf4bcfgi2up65zpzhg2fmyi52kecqfhsaevbaha"
  },
  "name": "hello-2.10",
  "qtype": "ipfs",
  "references": {
    "references": [
      {
        "cid": {
          "/": "bafyreigpjalsm7jiyevkroghpnsmkh3lx3apymlxrdunc4fssmdv2patwy"
        },
        "name": "glibc-2.30"
      }
    ],
    "zhasSelfReference": true
  }
}
```
The main CID is a git reference, like we've done before, but with special additionally for self-references.
The references are CIDs to other objects in this same format: a git tree hash with references.

Now, let's verify we can fetch what we've added.

We again delete our path so we will have something we need to download later:

```command
$ nix-store --delete $result
```

and refetch it, using the `nix ensure-ca` command to ensure that we have the build at that address:

```command
$ nix ensure-ca hello-2.10:ipfs:f0171122025ddb2f2ff6a6bbfe57a1137ad15d3e0ad068cb137e1570501c7ddcc24c19ebc --substituters ipfs://
$ nix shell $result -c hello
Hello, world!
```

#### Integration of traditional artifacts

We also improved one the old Nix x IPFS experiments with a trust/translation mapping and integrated it into our prototype.

> While this isn't an ideal experience for sharing sources, as it requires careful configuration, it does have the advantage of working with all Nix code in the wild.
> We recognize that that all the known source hashes in major repos like Nixpkgs won't be converted to our new formats overnight.
> It's good to have an additional way to use IPFS immediately, which in turn will help motivate the community to make the transition to the new hashing schemes we still maintain are the right choice long-term.

Let's start with building the `hello` package again:
```console
$ nix build -f ./nixpkgs hello
$ nix shell ./result -c hello
Hello, world!
```

We can put the artifact in IPFS without first transforming it with `nix make-content-addressable`.
Because we don't transform it, we will need to store the mapping from the Nix path to IPFS CID.
We call this the "trust mapping".

> In the common case, the relationship between the store path and the data it contains cannot be verified except by redoing the build, in which case, you may ask why are you looking up a pre-built build anyway! So for regular situations the choices are "trust or ignore", hence our name.

We'll create a new trust mapping by taking the hash of the empty one and adding a new key-value pair.

```console
$ nix copy ./result --to ipfs://$(echo {} | ipfs dag put)?allow-modify=true
warning: created new store at 'ipfs://<non-deterministic-store-hash>'.
         The old store at ipfs://bafyreigbtj4x7ip5legnfznufuopl4sg4knzc2cof6duas4b3q2fy6swua is immutable, so we can't update it
```

> The `allow-modify=true` tells Nix to allow store-modifying commands for immutable stores, and print out the new hash created by the "functional update".
> The default is to just be "read-only" and not create new stores.
> Nix users are accustomed to stores being mutable, so the warning may come as a surprise to them.
> IPFS users however are familiar with commands like `ipfs object patch` that both take and return CIDs, so this should be less surprising to them.

> The new trust mapping CID is not deterministic like most of the other hashes because it contains timestamps and signatures.
> We could have removed the timestamps, but the still leaves the signatures.
> Also, perhaps it's a good thing that trust maps are less easy to reproduce: it could make it harder to accidentally confuse the provenance of a build.

Let's write down that new store:
```console
$ declare new_store=ipfs://<non-deterministic-store-hash>?allow-modify=true
```

Now, if we like, we can repeat the same command with the new store:
```console
$ nix copy ./result --to $new_store
```
The update is *ideompotent*, doing nothing the second time, so there's nothing to warn about!


With our new trust mapping, we can fetch Hello via IPFS.
Again, we'll delete the old path and use the IPFS store as a substitute:
```console
$ result=$(readlink ./result); rm ./result && nix-store --delete $result; unset result
$ nix build -f ./nixpkgs hello --substituters $new_store -j0
$ nix shell ./result -c hello
Hello, world!
```

#### Using IPNS

We can also use IPNS following the pattern above.
This is nice because with IPNS we have actual mutation, and can "read-modify-write" the IPNS value.
However, the commands might be slow.

Let's use the original store again:
```console
$ nix build -f ./nixpkgs hello
```

Before uploading to IPNS, we have to get the associated primary key hash.
To get that, we'll just try to upload the empty hash again, and parse the correct ID from the response:
```console
$ EMPTY_HASH=$(echo {} | ipfs dag put)
$ IPNS_ID=$(ipfs name publish $EMPTY_HASH --allow-offline | awk '{print substr($3,1,length($3)-1)}')
```

Now we can just upload our artifact like before:
```console
$ nix copy ./result --to ipns://$IPNS_ID
```

And, as before, we can delete the local version, and download from IPFS — this time with the trust mapping identified via IPNS:
```
$ result=$(readlink ./result); rm ./result && nix-store --delete $result; unset result
$ nix build -f ./nixpkgs hello --substituters ipns://$IPNS_ID -j0
$ nix shell ./result -c hello
Hello, world!
```

#### Using DNSLink

You can also use addresses of the form `ipns://domain`, where `domain` is a [DNSLink domain](https://docs.ipfs.io/concepts/dnslink/).

> We didn't include a walkthrough in this doc because of the need to put DNS in a sandbox, and the lack of a general programatic way to update DNS records suitable for inclusion in Nix.
> But do note that `DNSLink` is substantially more performant than IPNS in practice.
> We suspect that production systems will use this; Nix will again print out the new IPFS hash so one can update their DNS record in a manner of their choosing.


## Milestone 2

### Part 3. Floating content-addressed derivations

#### Basic usage

This part will deal with the newly introduced "floating content-addressed" derivations ("floating" here is used to suggest the opposite of "fixed").

Let's create, in the file `content-addressed-0.nix`, a floating content-addressed derivation:
```nix
with import ./nixpkgs {};

stdenv.mkDerivation {
  name = "simple-content-addressed";
  buildCommand = ''
    echo "Building a CA derivation"
    mkdir -p $out
    echo "Hello World" > $out/hello
  '';
  __contentAddressed = true;
  outputHashMode = "recursive";
  outputHashAlgo = "sha256";
}
```

What's going on here?
> If you are familiar with Nix, you might notice these derivations have the `outputHashMode` and `outputHashAlgo` from today's "fixed-output derivations", but no `outputHash`, and also a new `__contentAddressed = true;`.

To understand them, let's review how Nix worked traditionally; users were presented with two choices for derivations:

  - Input-addressed derivations, which aren't addressed by their output, but by the data in the derivation itself. This is currently the most commonly used derivation type in nixpkgs.

  - Fixed content-addressed derivations, which were addressed by their output, but could only work if you specified the content address in the derivation itself before the build. This type of derivation is currently mainly used to fetch source code.

Floating content-addressed derivations get us the best of the two approaches:
We have the comfort of not having to commit to a hash ahead of time when writing the derivation for a deterministic build, _and_ we get content-addressed output, which is more principled, easier to work with, and integrates easily with other systems like IPFS.

We can build this derivation via
```
# out=$(nix-build ./content-addressed-0.nix --no-out-link)
warning: do not know how to query for unknown floating content-addressed derivation output yet
Resolved derivation: '/nix/store/l15yvnask1cmhy56asick2ighwmlbihq-simple-content-addressed.drv' -> '/nix/store/gbn8mhdy8qfq50i74p62g02ygd11h7b6-simple-content-addressed.drv'...
warning: do not know how to query for unknown floating content-addressed derivation output yet
building '/nix/store/gbn8mhdy8qfq50i74p62g02ygd11h7b6-simple-content-addressed.drv'...
Building a CA derivation
```

Now that we have the result of the derivation, let's examine the associated metadata:
```
$ nix path-info --json $out | jq
[
  {
    "path": "/nix/store/31jl3ynipmm7z3pnf42lcghani93r4yq-simple-content-addressed",
    "narHash": "sha256-JjC2jNaxvLUxX7dT5u0TWUe90qX2koSJeJQ85c7oWyU=",
    "narSize": 296,
    "references": [],
    "ca": "fixed:r:sha256:09avx37fag4lg24q94pnlp9bsisr2gnyclxpbwqvbg5iss6bcc16",
    "deriver": "/nix/store/nc3l21r2bynifbryb4ppj65sycni3h72-simple-content-addressed.drv",
    "registrationTime": 1599248702,
    "ultimate": true
  }
]
```
As we can see from the `ca` field, we really have a content-addressed output path here.

> If you are confused why this says "fixed", remember we are looking at the output path not the derivation itself.
> Long ago, Nix decided to use "fixed" in this CA field, perhaps because fixed-output derivations.
> We don't want otherwise content-addressed outputs to betray their origins by using a different scheme, so floating CA derivations' outputs use exactly the same sort of scheme.



#### Dependencies

What about dependencies?
Let's try a more complex example, in the file `content-addressed-1.nix`

```nix
with import ./nixpkgs {};

{ seed ? 0 }:

rec {
  root = stdenv.mkDerivation {
    name = "root";
    buildCommand = ''
      echo "Building a CA derivation"
      echo "The seed is ${toString seed}"
      mkdir -p $out
      echo "Hello World" > $out/hello
    '';
    __contentAddressed = true;
    outputHashMode = "recursive";
    outputHashAlgo = "sha256";
  };
  dependent = stdenv.mkDerivation {
    name = "dependent";
    buildCommand = ''
      echo "building a dependent derivation"
      mkdir -p $out
      echo ${root}/hello > $out/dep
    '';
    __contentAddressed = true;
    outputHashMode = "recursive";
    outputHashAlgo = "sha256";
  };
}
```

Now, we've created two derivations here to show you that a floating content-addressed derivation can depend on other content-addressed derivations.

> N.B Input-addressed derivations can't yet depend on content-addresses ones.
> There should be no reason at this poitn to use input-addressed derivations, so we aren't too concerned, but to make migrations as easy as possible we will see to it being eventually added.

Let's build both:
```console
$ nix-build ./content-addressed-1.nix -A dependent --no-out-link
these 2 derivations will be built:
  /nix/store/wh95jh5p2iwfxcwhgxnjx107l7znfalc-root.drv
  /nix/store/lxn37cp4vkq5w4fccni5kz6w207m448z-dependent.drv
nix-build ./content-addressed-1.nix -A dependent --no-out-link --arg seed 5
  ...
...
warning: do not know how to query for unknown floating content-addressed derivation output yet
warning: do not know how to query for unknown floating content-addressed derivation output yet
Resolved derivation: '/nix/store/wh95jh5p2iwfxcwhgxnjx107l7znfalc-root.drv' -> '/nix/store/c1mvla9ird50x20h8jx0zmrv548hf6fn-root.drv'...
warning: do not know how to query for unknown floating content-addressed derivation output yet
building '/nix/store/c1mvla9ird50x20h8jx0zmrv548hf6fn-root.drv'...
Building a CA derivation
The seed is 0
Resolved derivation: '/nix/store/lxn37cp4vkq5w4fccni5kz6w207m448z-dependent.drv' -> '/nix/store/ka90wa7mbpz68zfkixcgb4xk5w8pcmrp-dependent.drv'...
warning: do not know how to query for unknown floating content-addressed derivation output yet
building '/nix/store/ka90wa7mbpz68zfkixcgb4xk5w8pcmrp-dependent.drv'...
building a dependent derivation
/nix/store/wgq7pqh2h5pkc4lzhgygqbf448ysjfmx-dependent
```

See the `Resolved derivation: '*' -> '*'...` lines?
These indicate that before Nix builds a CA derivation, nix will replace any input derivations with the the paths they produced, giving us a more  compact and normalized derivation to actually build.

#### Different derivations same output

We also parametrized these derivation on a "seed" argument. Let's try constructing the derivation for various values of seed:

```console
$ nix-build ./content-addressed-1.nix -A dependent --no-out-link --arg seed 5
these 2 derivations will be built:
  /nix/store/qksxbzk77zpqzgg5ss439ym10f08igzr-root.drv
  /nix/store/bdrz2ncx991n1d8jqgyrzkpfpl4famjz-dependent.drv
warning: do not know how to query for unknown floating content-addressed derivation output yet
warning: do not know how to query for unknown floating content-addressed derivation output yet
Resolved derivation: '/nix/store/qksxbzk77zpqzgg5ss439ym10f08igzr-root.drv' -> '/nix/store/71jjxk63p9mhag0d5x6bbgkiz7ggv5dh-root.drv'...
warning: do not know how to query for unknown floating content-addressed derivation output yet
building '/nix/store/71jjxk63p9mhag0d5x6bbgkiz7ggv5dh-root.drv'...
Building a CA derivation
The seed is 5
Resolved derivation: '/nix/store/bdrz2ncx991n1d8jqgyrzkpfpl4famjz-dependent.drv' -> '/nix/store/ka90wa7mbpz68zfkixcgb4xk5w8pcmrp-dependent.drv'...
/nix/store/wgq7pqh2h5pkc4lzhgygqbf448ysjfmx-dependent
```
Notice how `dependent` was only built once?
This is because with the normalizing we do, the second derivation in fact doesn't end up varying with the seed.
Since "root" with each seed produced the same store path, that means that the two "downstream" derivations, once normalized to refer to the content-addressed outout of "root" rather than "root" itself, become exactly the same.
Nix never needs to build any derivation more than once, and so the resolved "dependent" isn't built a second it.


#### Derivations producing IPLD-compatible outputs

So how are these content-addressed derivations good for IPFS?
Recall befere we had to use `nix make-content-addressable` to convert
let's remember that IPLD is meant to be a common namespace for all the hash-inspired contents, and so it would be nice to be able to use the IPLD format:

Let's create, in the file `ipfs-derivation-output.nix` two derivations:

```nix
let
  system = builtins.currentSystem;
  builder = builtins.ipfsPath {
    name = "dash-dir";
    cid = "f01711220a1032a51527eeab22081ab5a23406c1d6ea504f22218e98784fd9097160d5b14";
  } + "/dash";
  miniMkdir = builtins.ipfsPath {
    name = "miniMkdir-dir";
    cid = "f01711220d03ed5f1534f08528a13fa0b2deebcfe73ea755a35a56ca1c28fab9e080214f5";
  } + "/miniMkdir";
  args = ["-c" "eval \"\$buildCommand\""];

in
  rec {
    root = builtins.derivation {
      name = "root";
      buildCommand = ''
        set -ex
        echo "Building a CA derivation"
        echo $miniMkdir*
        $miniMkdir $out
        echo "Hello World" > $out/hello
      '';
      __contentAddressed = true;
      outputHashMode = "ipfs";
      outputHashAlgo = "sha256";
      inherit system builder args miniMkdir;
    };

    dependent = builtins.derivation  {
      name = "dependent";
      buildCommand = ''
        set -ex
        echo "Building a CA derivation"
        $miniMkdir $out
        echo ${root} > $out/root
        echo "Hello World" > $out/hello
      '';
      __contentAddressed = true;
      outputHashMode = "ipfs";
      outputHashAlgo = "sha256";
      inherit system builder args miniMkdir;
    };
}
```

You can see that, in addition to having the `__contentAddressed` flag we talked about before, both of them use "ipfs" for `outputHashMode`, which means that they follow the IPLD schema.

Now we can instantiate the derivation first (which means actually constructing the drv file - it will return the derivation path), so that we can show the derivation. We do that with:

```console
$ drv0=$(nix-instantiate ./ipfs-derivation-output.nix -A root --substituters ipfs://)
$ drv1=$(nix-instantiate ./ipfs-derivation-output.nix -A dependent --substituters ipfs://)
$ nix show-derivation --derivation "$drv0"
```
```json
{
  "/nix/store/q79x77al9dvbdyv2j2cian8s2dijxpfn-root.drv": {
    "outputs": {
      "out": {
        "hashAlgo": "ipfs:sha256"
      }
    },
    "inputSrcs": [
      "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir",
      "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir"
    ],
    "inputDrvs": {},
    "platform": "x86_64-linux",
    "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
    "args": [
      "-c",
      "eval \"$buildCommand\""
    ],
    "env": {
      "buildCommand": "set -ex\necho \"Building a CA derivation\"\necho $miniMkdir*\n$miniMkdir $out\necho \"Hello World\" > $out/hello\n",
      "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
      "miniMkdir": "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir",
      "name": "root",
      "out": "/1rz4g4znpzjwh1xymhjpm42vipw92pr73vdgl6xs1hycac8kf2n9",
      "outputHashAlgo": "sha256",
      "outputHashMode": "ipfs",
      "system": "x86_64-linux"
    }
  }
}
```

```console
$ nix show-derivation --derivation "$drv1"
```
```json
{
  "/nix/store/vzdwqsfpj6apk2kgfk8bn09py8h865va-dependent.drv": {
    "outputs": {
      "out": {
        "hashAlgo": "ipfs:sha256"
      }
    },
    "inputSrcs": [
      "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir",
      "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir"
    ],
    "inputDrvs": {
      "/nix/store/q79x77al9dvbdyv2j2cian8s2dijxpfn-root.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
    "args": [
      "-c",
      "eval \"$buildCommand\""
    ],
    "env": {
      "buildCommand": "set -ex\necho \"Building a CA derivation\"\n$miniMkdir $out\necho /16xx03jc550003x886r6h4rhrnqlkfpkh7cg2qyi2a6f61dk61qv > $out/root\necho \"Hello World\" > $out/hello\n",
      "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
      "miniMkdir": "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir",
      "name": "dependent",
      "out": "/1rz4g4znpzjwh1xymhjpm42vipw92pr73vdgl6xs1hycac8kf2n9",
      "outputHashAlgo": "sha256",
      "outputHashMode": "ipfs",
      "system": "x86_64-linux"
    }
  }
}
```

Let's build the whole graph:
```console
$ nix-build ./ipfs-derivation-output.nix -A dependent --no-out-link
these 2 derivations will be built:
  /nix/store/q79x77al9dvbdyv2j2cian8s2dijxpfn-root.drv
  /nix/store/vzdwqsfpj6apk2kgfk8bn09py8h865va-dependent.drv
warning: do not know how to query for unknown floating content-addressed derivation output yet
warning: do not know how to query for unknown floating content-addressed derivation output yet
building '/nix/store/q79x77al9dvbdyv2j2cian8s2dijxpfn-root.drv'...
+ echo Building a CA derivation
Building a CA derivation
+ echo /nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir
/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir
+ /nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir /nix/store/vscggi356kladi3kg5y6qkkwcng5cfyd-root
+ echo Hello World
Resolved derivation: '/nix/store/vzdwqsfpj6apk2kgfk8bn09py8h865va-dependent.drv' -> '/nix/store/g1y1v8vm8jfbn02agpw6qw9d3s07sn4p-dependent.drv'...
/nix/store/2dmmdpamd31if2kmpkqgjn1fj4hv5khq-dependent
```

### IPLD derivations

#### Export
Having seen how we can build derivations that are compatible with IPLD format, we now want a good UI to import and export derivations directly from IPLD.

This is obtained the new command `nix ipld-drv`.

Let's use the derivation from the previous heading, whose path is stored in $drv

We can upload this derivation to IPLD:
```console
$ cid=$(nix ipld-drv export --derivation $drv1)
$ echo $cid
f0171122001172d1b2de78bf64d23ca30b0e41cdb56a60a0030f6ea58ca80084e374d94e7
```
this will convert the derivation in the format expected by IPLD, and upload it (make sure your daemon is on).
It will also return the IPLD content address ID under which the derivation has been stored.

We can look at  the IPLD represention, which is very similiar to the JSON representation `nix show-derivation` already has:
```console
# dependent
ipfs dag get f017112201fb9b75473bb2c80744a1c472e6df89aef924c0d024d1a4b3ed0fb18fc26088a | jq
```
```json
{
  "args": [
    "-c",
    "eval \"$buildCommand\""
  ],
  "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
  "env": {
    "buildCommand": "set -ex\necho \"Building a CA derivation\"\n$miniMkdir $out\necho /16xx03jc550003x886r6h4rhrnqlkfpkh7cg2qyi2a6f61dk61qv > $out/root\necho \"Hello World\" > $out/hello\n",
    "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
    "miniMkdir": "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir",
    "name": "dependent",
    "out": "/1rz4g4znpzjwh1xymhjpm42vipw92pr73vdgl6xs1hycac8kf2n9",
    "outputHashAlgo": "sha256",
    "outputHashMode": "ipfs",
    "system": "x86_64-linux"
  },
  "inputDrvs": [
    [
      {
        "cid": {
          "/": "bafyreifdx57lk6pxmpp27hkgiakazx5cab2xcgxe2mse42ql3bx4thk6lm"
        },
        "name": "root"
      },
      [
        "out"
      ]
    ]
  ],
  "inputSrcs": [
    {
      "cid": {
        "/": "bafyreifbamvfcut65kzcbanllirua3a5n2sqj4rcdduypbh5sclrmdk3cq"
      },
      "name": "dash-dir"
    },
    {
      "cid": {
        "/": "bafyreigqh3k7cu2pbbjiue72bmw65ph6opvhkwrvuvwkdqupvopaqaqu6u"
      },
      "name": "miniMkdir-dir"
    }
  ],
  "name": "dependent",
  "outputs": [
    "out"
  ],
  "platform": "x86_64-linux"
}
```

We can follow the link in the inputDrvs map to get the IPLD representation of "root":

```console
# root
ipfs dag get bafyreifdx57lk6pxmpp27hkgiakazx5cab2xcgxe2mse42ql3bx4thk6lm | jq
```
```json
{
  "args": [
    "-c",
    "eval \"$buildCommand\""
  ],
  "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
  "env": {
    "buildCommand": "set -ex\necho \"Building a CA derivation\"\necho $miniMkdir*\n$miniMkdir $out\necho \"Hello World\" > $out/hello\n",
    "builder": "/nix/store/v0qd217nvhgws2w3id7xp6444lg3yf6d-dash-dir/dash",
    "miniMkdir": "/nix/store/h42sq2q51mkh4mh0q4fimfdn7xzzn6a7-miniMkdir-dir/miniMkdir",
    "name": "root",
    "out": "/1rz4g4znpzjwh1xymhjpm42vipw92pr73vdgl6xs1hycac8kf2n9",
    "outputHashAlgo": "sha256",
    "outputHashMode": "ipfs",
    "system": "x86_64-linux"
  },
  "inputDrvs": [],
  "inputSrcs": [
    {
      "cid": {
        "/": "bafyreifbamvfcut65kzcbanllirua3a5n2sqj4rcdduypbh5sclrmdk3cq"
      },
      "name": "dash-dir"
    },
    {
      "cid": {
        "/": "bafyreigqh3k7cu2pbbjiue72bmw65ph6opvhkwrvuvwkdqupvopaqaqu6u"
      },
      "name": "miniMkdir-dir"
    }
  ],
  "name": "root",
  "outputs": [
    "out"
  ],
  "platform": "x86_64-linux"
}
```

The idea is that that this should be a very natural build interface for IPFS users to use, even without the Nix expression language.

#### import

Conversely, if we have a content address ID for a derivation in `$cid`, we can import it in our local store using:
```console
$ nix ipld-drv import $cid
...
```
This will download the derivation, convert it in the format expected by nix, and realize it in the local store.

## The end

We hope that this brief introduction shows you the potential in integrating these two wonderful technologies, and sparks some excitement for the changes to come!
