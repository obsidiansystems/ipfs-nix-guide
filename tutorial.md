# IPFS × Nix Tutorial

Welcome to our walkthrough of Milestone 1 of our IPFS × Nix integration.

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
```bash
$ cd $(mktemp -d)
```

In case you already have Nix installed, you can build this version of Nix from our fork:

``` console
$ git clone -b ipfs-master https://github.com/obsidiansystems/nix
$ nix-build nix/release.nix -A build.x86_64-linux # or x86_64-darwin
$ nix-shell -p ./result
```

Alternatively, you can obtain a prebuilt binary via IPFS:

``` console
$ mkdir -p $PWD/nix/bin
$ ipfs get /ipfs/QmRBjvUhYMfkKgB9ACWcGpRSsAV3Mi2qwAnN1mFKk1p8wh -o $PWD/nix/bin/nix
$ chmod +x $PWD/nix/bin/nix
$ ln -s ./nix $PWD/nix/bin/nix-store
$ export PATH=$PWD/nix/bin:$PATH
```

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

and silence errors/warnings for using unstable features
```console
$ echo "experimental-features = nix-command flakes ca-references" > "$NIX_CONF_DIR"/nix.conf
```

> In a future release that's closer to being upstreamed, you will also see "ipfs" in that list.
This will allow us to continue iterating upstream.

Also, to allow us to evaluate Nixpkgs, we need some of Nix's `corepkgs`:

```console
$ mkdir -p $PWD/nix/share/nix/corepkgs
$ curl https://raw.githubusercontent.com/obsidiansystems/nix/ipfs-develop/corepkgs/derivation.nix -o $PWD/nix/share/nix/corepkgs/derivation.nix
$ curl https://raw.githubusercontent.com/obsidiansystems/nix/ipfs-develop/corepkgs/fetchurl.nix -o $PWD/nix/share/nix/corepkgs/fetchurl.nix
```
(We're getting them from our branch, but do note we haven't modified them and have no plans to do so)

Now we're ready to dive into the changes in Nix and its new capabilities.

## Milestone 1

### Part 1. Better Git × Nix integration

#### Hash file data like Git

To ensure smooth cooperation between IPFS and Nix, one of our first objectives was to add native support for the Git protocol: in particular we want Nix to be able to hash Git data with the same hashes that Git tree would use.

Let's clone an example repo, go to a commit, and ask Git for the hash:
```bash
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

```bash
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

```bash
$ nix eval "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })"
{
  lastModified = "19700101000000";
  narHash = "sha256-uIr1qotOcueLPXGNOTArfNSABsA3wAXrz4EqAQxJ8lI=";
  outPath = "/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source";
}
```

If you don't know the hash, you can use the `gitIngestion` attribute, to tell Nix to use the tree hash after it retrieves the default branch:

```bash
$ nix eval "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; gitIngestion = true; })"
{
  lastModified = "20200527194432";
  narHash = "sha256-uIr1qotOcueLPXGNOTArfNSABsA3wAXrz4EqAQxJ8lI=";
  outPath = "/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source";
  rev = "14a65f594e922bac9f31b89a4271df73fb4677a4";
  revCount = 380; shortRev = "14a65f5";
}
```
The two results are slightly different, because the second version has also some attributes about the commit (like `revCount`), but the important thing is that the `narHash` and the `outPath` are the same!

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
$ nix eval --store ipfs:// "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })"
```

Alternatively, if you don't know the tree hash ahead of time:
```bash
nix eval --store ipfs:// "(builtins.fetchTree { type = \"git\"; url = \"https://github.com/ipfs/ipfs\"; gitIngestion = true; })"
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
```bash
$ nix-store --delete /nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source
finding garbage collector roots...
deleting '/nix/store/p3mq27al6qgja0m0b97nq1vs55fa34pr-source'
deleting '/nix/store/trash'
deleting unused links...
note: currently hard linking saves -0.00 MiB
1 store paths deleted, 0.22 MiB freed
```

Then we can copy that path back from IPFS, using `ipfs://` as a substituter.
```bash
$ nix eval "(builtins.fetchTree { type = \"git\"; url = \"http://example.com\"; treeHash = \"$(git -C ipfs rev-parse HEAD:)\"; })" --substituters ipfs://
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
```bash
$ nix build -f ./nixpkgs hello
$ nix shell ./result -c hello
Hello, world!
```

> If the "build" looks a bit quick or terse, this is because by default (i.e. if not specified in that `nix.conf` file we made at the beginning) Nix trusts `cache.nixos.org` as a substituter.
> `cache.nixos.org` provides builds produced as part of Nixpkgs' own CI.
> You can pass `--substituters ''` to force a local build instead, if you like.

Let's convert it to a content addressed path, via the `make-content-addressable` command, and take a look at the rewritten paths:
```bash
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

```bash
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
```bash
$ nix build -f ./nixpkgs hello
$ nix shell ./result -c hello
Hello, world!
```

We can put the artifact in IPFS without first transforming it with `nix make-content-addressable`.
Because we don't transform it, we will need to store the mapping from the Nix path to IPFS CID.
We call this the "trust mapping".

> In the common case, the relationship between the store path and the data it contains cannot be verified except by redoing the build, in which case, you may ask why are you looking up a pre-built build anyway! So for regular situations the choices are "trust or ignore", hence our name.

We'll create a new trust mapping by taking the hash of the empty one and adding a new key-value pair.

```bash
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
```bash
$ declare new_store=ipfs://<non-deterministic-store-hash>?allow-modify=true
```

Now, if we like, we can repeat the same command with the new store:
```bash
$ nix copy ./result --to $new_store
```
The update is *ideompotent*, doing nothing the second time, so there's nothing to warn about!


With our new trust mapping, we can fetch Hello via IPFS.
Again, we'll delete the old path and use the IPFS store as a substitute:
```bash
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

This part of the work adds the so called "floating" derivations ("floating" here is used to suggest the opposite of fixed).

These are builds that are deterministic enough that we know they'll always produce the same hash, even if we don't know the hash beforehand.

The idea is that we will build the derivation first, hash the output, and use that result as the hash for the output.

This allows derivation for which the content address is not available in the beginning to be used without an extra step.

In order to use this feature, you can add the `__contentAddressed = true` attribute in an otherwise normal looking derivation. For example, consider:

```nix
root = mkDerivation {
  name = "text-hashed-root";
  buildCommand = ''
    set -x
    echo "Building a CA derivation"
    mkdir -p $out
    echo "Hello World" > $out/hello
  '';
  __contentAddressed = true;
  outputHashMode = "recursive";
  outputHashAlgo = "sha256";
};
```

Put this content inside "floating-derivation.nix". Now you can build it with:
```sh
nix-build --experimental-features ca-derivations ./floating-derivation.nix -A root --no-out-link
```

#### IPLD derivations

Comment about what IPLD is.

Now, since IPLD is meant to be a single namespace for all the hash-inspired contents, we thought a tighter integration between nix and ipld would be in order.

This is obtained with two new commands, that let nix export a derivation directly to IPLD (returning a cid - content address Id) and import a cid from IPLD, realizing the derivations in the local store.

Let's see those commands in depth. First we have to create a derivation:

```nix
with import ./config.nix;

rec {
  root = mkDerivation {
    name = "ipfs-derivation-output";
    buildCommand = ''
      set -x
      echo "Building a CA derivation"
      mkdir -p $out
      echo "Hello World" > $out/hello
    '';
    __contentAddressed = true;
    outputHashMode = "ipfs";
    outputHashAlgo = "sha256";
    args = ["-c" "eval \"\$buildCommand\""];
  };

  dependent = mkDerivation {
    name = "ipfs-derivation-output-2";
    buildCommand = ''
      set -x
      echo "Building a CA derivation"
      mkdir -p $out
      ln -s ${root} $out/ref
      echo "Hello World" > $out/hello
    '';
    __contentAddressed = true;
    outputHashMode = "ipfs";
    outputHashAlgo = "sha256";
    args = ["-c" "eval \"\$buildCommand\""];
  };
}
```

This simple derivation will build two components, called "root" and "dependent" (the names are arbitrary, the dependent one is called that way because it depends on the main one).

You can see both of them use "ipfs" as their hashing mode, which means that they are properly content-addressed. Save that derivation in the file "ipfs-derivation-output.nix".

We want to instantiate the derivation first (which means actually constructing the drv file). We do that with:

```
drv=$(nix-instantiate --experimental-features ca-derivations ./ipfs-derivation-output.nix -A dependent)
```

Now we can upload this derivation to IPLD:
```
nix ipld-drv export --derivation $drv
```
this command will convert the derivation in the format expected by IPLD, and upload it (make sure your daemon is on); it will also return the content address ID under which the derivation has been stored in IPLD

Conversely, if we have a cid for a derivation in `$cid`, we can import it in our local store using:
```
nix ipld-drv import $cid
```

## The end

We hope that this brief introduction shows you the potential in integrating these two wonderful technologies, and sparks some excitement for the changes to come!
