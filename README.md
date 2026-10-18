# k3s-root
==========

> [!IMPORTANT]
> This fork addresses an nftables userspace compatibility issue: older `nft` clients, including
> `nft` 1.1.0 from the original `hardened-calico` image, can segfault while reading nftables rules
> created by `nft` 1.1.3 on a Debian 13 "trixie" host. See the
> [nftables userspace compatibility matrix](https://gist.github.com/aojea/c17a8bbf40d98c3f59c8ed2712ec43a0)
> and the related [Istio issue #58492](https://github.com/istio/istio/issues/58492),
> [Debian bug #1129273](https://bugs.debian.org/1129273), and
> [Ubuntu bug #2142552](https://bugs.launchpad.net/bugs/2142552).

`k3s-root` is based on [Buildroot](https://github.com/buildroot/buildroot) and builds the
userspace binaries packaged for [K3s](https://github.com/k3s-io/k3s). This repository is Octivi's
fork of [k3s-io/k3s-root](https://github.com/k3s-io/k3s-root), with updated nftables components
for compatibility with rulesets created on Debian 13 hosts.

The Octivi build carries nftables 1.1.7 and libnftnl 1.3.2. It also supplies the statically linked
`nft` binary used by [octivi/hardened-calico](https://github.com/octivi/hardened-calico) to replace
`/usr/sbin/nft` in Rancher's Calico image. CI verifies bidirectional compatibility between the
built `nft` binary and nftables 1.1.3 from Debian 13: each version must be able to read rulesets
produced by the other.

## Building

`k3s-root` is built in docker, which means you should be able to simply `make` on your machine to compile. By default, `make` will compile for `amd64`. If you want to compile for other architectures, you can run commands like:
```
make ARCH=arm64
```

or alternatively, if you want to compile all architectures at once, you can run:
```
make all-build
```

If it is not the first time you build, delete the directories artifacts/ and dist/ to avoid problems when creating the symbolic link in the package step:
```
sudo rm -r artifacts/ dist/
```

The default way of building this project is through a docker container using `docker buildx`. While you may be able to run the build scripts standalone, we do not test or support doing so.

## Upgrading to new buildroot version

To upgrade to a new buildroot version:

1. Check out a new branch for your work: `git checkout -B bump-buildroot origin/master`
1. Modify the `BUILDROOT_VERSION` in scripts/download
2. Run `make download` to prepare a Docker image for further work
4. For each target architecture: 
   1. Start a shell in the resulting image: `docker run --rm -it -e BUILDARCH=<ARCH> k3s-io/k3s-root:dev /bin/bash`
   2. Run `./scripts/patch`
   3. Run `cd /usr/src/buildroot && make olddefconfig`
   4. Outside the container, split changes to the buildroot `.config` into `buildroot/config` and `buildroot/<ARCH>config`
   5. Run `make BUILDARCH=<ARCH>`
   6. Verify if the upgrade worked correctly by comparing the old tarball and the new one. If the same files are there, then you are set.
      If the build failed, or some files are missing, you may need to remove or adapt patches or config for the new buildroot version.
