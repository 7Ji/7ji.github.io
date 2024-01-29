---
layout: post
title:  "My Arch repo and builder"
date:   2024-01-29 20:50:00 +0800
categories: designdoc
---

Due to the increasing number of [Arch packages I maintain](https://github.com/7Ji-PKGBUILDs), I decided to start running [my own third-party Arch repo](https://github.com/7Ji/archrepo) on July 23, 2023, with Github releases as the repo storage, unlike most of the existing repos. The repo started as simply a mirror of my local repo, but later turned out to be so popular that I started to maintain it dedicatedly.

Until now, the repo has gone through several stages in the sense of build style: manual build, scripted buil, standalone Rust builder. In the past few days I have been rewriting the builder to pay up the tech debt I made in the early prototype stage. 

Since I'm kinda stuck on some components I'm currently rewriting as I have too many things on my todo list, I feel it clears my mind to document how I started the repo, maintained it, and designed the several versions of the builders. I hope I could clear up my mindset during this writing.

## Before the online repo (Early 2023 - July 23, 2023)
Before hosting the repo on Github, I've hosted a local repo that I maintain by myself for all of my Arch / ALARM devices. The reason behind this is simple: all these Amlogic boxes would need either [linux-aarch64-7ji](https://github.com/7Ji-PKGBUILDs/linux-aarch64-7ji) or [linux-aarch64-flippy](https://github.com/7Ji-PKGBUILDs/linux-aarch64-flippy) to get an up-to-date and smooth user experience, while the `linux-aarch64` package from ALARM repo usually lagged behind A LOT and can't run on my Amlogic devices.

Before the local repo was hosted, I usually just `scp` or `nc` the built packages around and install them manually on each of my boxes. This turned out to be more and more tedious as I got more boxes (more than 10 now) and I wanted/needed to keep the kernels up-to-date with kernel.org releases every week, day 0 after tag.

For a long time, in this time period, all the packages were built by myself by hand, at the same time as I update and test them before pushing them to AUR.

Until one day I just got so tired of transfering these built packages around, and decided to host my local repo so each box can simply `pamcan -Syu`. I can't remember which exact day was it, but it should be in early 2023.

Interestingly, although I host and only host kernel packages on the repo, other projects of mine, [amlogic-s9xxx-archlinuxarm](https://github.com/7Ji/amlogic-s9xxx-archlinuxarm) and [orangepi5-archlinuxarm](https://github.com/7Ji/orangepi5-archlinuxarm), still need to build the kernel packages on Github Actions nightly as I didn't see much use for others of my packages. So, for a long time, I maintained a pacman repo, but it's only an offline repo.

## Manual build (July 23, 2023 - Sep 1, 2023)

In July 2023, on the AUR comment sections of one of the packages I maintain, [linux-aarch64-orangepi](https://github.com/7Ji-PKGBUILDs/linux-aarch64-orangepi5), a user requested a `-bin` package with Github Actions artifact of my [orangepi5-archlinuxarm](https://github.com/7Ji/orangepi5-archlinuxarm) project so they can update more easily. 

This got me thinking: why would people and I need to re-build the same packages multiple times (for me, it's a local test/repo build and a Github Actions build), when I can just build it once and share it with every one?

Hmm, but for an online repo, I would need online storage. I do have several servers around the world, which always host some websites. But considering an online pacman repo would drain A LOT of bandwidth, more than what I paid for, and using Cloudflare CDNs upfront is very unfriendly to pacman, I came up with a crazy idea:

What if, I could use Github releases as repo storage? 

I've already used Github releases for several of my projects before that. The benefit is very clear: 

1. Gtihub already has its own CDN, but without an anti-bot filter like Cloudflare, which would make it very ideal for worldwide redistribution, and still friendly to pacman.
2. The space of Github releases is unlimited (or limited, but I didn't bother to check). So I can upload a lot of packages, and won't need to worry about reaching the limit one day.
3. Its plain storage structure is very alike to a pacman repo storage structure. You get the URL of the repo, and filename of the package, and you then automatically get the URL of the package.
4. It's free! Well this is the most important one. Some others would also use Gitub storage as repo storage but instead use git repos directly. Well, good luck for their Github LFS bills then...

So, I quickly created a Github repo, created a PGP signing key, created two dumb tags `aarch64` and `x86_64`, signed all my packages and uploaded my local packages there.

This worked for a long time, as the repo only contained several packages I maintain by myself that I would need to build when bumping anyway. I just sign the packages I build and upload to Github releases manually every week. Maybe a few extra minutes are needed a week, and that's acceptable.

But things never always work the same, do they?

## Scripted build (Sep 1, 2023 - Sep 14, 2023)

In the early August of 2023, I bought a new Orange Pi 5 Plus to replace my Orange Pi 5 bought Jan 2023 to act as my main aarch64 build server. As a result, I could finally use my Orange Pi 5 for desktop use cases. 

As a replace to my current desktop setup, I want a complete Arch Linux ARM + KDE plasma + chromium setup, with smooth hardware-based video decoding experience. There're already a couple of Rockchip MPP related packages on AUR, so I quickly started to create packages to fill the missing gaps: [chromium-mpp](https://github.com/7Ji-PKGBUILDs/chromium-mpp), [libv4l-rkmpp-git](https://github.com/7Ji-PKGBUILDs/libv4l-rkmpp-git), [v4l-utils-mpp](https://github.com/7Ji-PKGBUILDs/v4l-utils-mpp), [mali-valhall-g610-firmware](https://github.com/7Ji-PKGBUILDs/mali-valhall-g610-firmware) and [libmali-valhall-g610](https://github.com/7Ji-PKGBUILDs/libmali-valhall-g610) . 

As I started maintaining these packages on AUR I also staretd commenting on some of the existing mpp packages to send patches to improve them. I couldn't remember the exact day, but I was invited as co-maintainer of `mpp-git` by **Hüseyin BIYIK** a.k.a. hbiyik, a.k.a. boogiepop, after I sent a patch about the installation localtion of the udev rules, who maintain almost all other Rockchip MPP related packages on AUR, and the famous Rockchip MPP enabled [fork of FFmpeg](https://github.com/hbiyik/FFmpeg).

After a week of that, hbiyik opened [an issue](https://github.com/7Ji/archrepo/issues/1) to request some of the packages he maintained to be hosted on my repo, two kodi packages, and one kernel package for radxa, and sure enough I agreed. 

With this request agreed, I started to realize the potential popularity of my repo, and the amount of packages I would host if I keep accepting/adding new packages. If I don't want myself to manually build A LOT of packages every day then I have to write a builder for the work.

Soon I started to write a simple enough fish shell based builder that takes minimum bandwidth usage and fast rebuild into account. It works as the following:

1. Taking a list of PKGBUILDs that's stored in a .yaml file in a `name: url` list (later reformatted to map)
2. Hashing remote git urls into 64-bit xxhash 3, with 16-char hash string as local repo names.
3. Fetching all remote git repos in parallel.
4. Extracting `PKGBUILD` blob from the root tree of repo HEAD/ master branch, all into a temporary folder.
5. Parsing all `PKGBUILD`s to get package metadata, including sources, dependencies, type of `pkgver`, etc
6. Caching all sources in parallel, with files cached with their integrity checksum as key, and git sources cached in a similar way as the above git repos.
7. For those with `pkgver` function, extract them to a state where we're ready to run `pkgver()`
8. Run `pkgver()` on those dynamic packages.
9. Generate a build id for each `PKGBUILD`, with `[name]-[git tree-like id]-[pkgver]-[dep hash]`
10. Extract for packages that need to be built but not extracted yet, and clean up those that do not need to built.
11. Build for needed build ids in parallel.

The [first commit](https://github.com/7Ji/archrepo/commit/022ad2c64817f1dff98bb4fdd57540d9bef85495) happens on Sep 1, 2023, and I quickly finished it up in the same day.

This proves to be a good enough design with minimum maintainer attention, and in the next two weeks I improved/ fixed the builder in various commits.

But it does miss a few points. The most important one is that it does not build in a containerized environment, so package dependencies would leak into host. Another one is that a lot of useful data strructures are missing in a plain shell based environment, and it makes some simple operations harder to be achieved.

So one day I started to rewrite the builder logic in Rust, and quickly finished the prototype.

## Standalone Rust builder (Sep 14, 2023 - Jan 27, 2024)

The Rust builder was named boringly as [arch_repo_builder](https://github.com/7Ji/arch_repo_builder), I would not write the whole thing down as it has its own [document](https://github.com/7Ji/arch_repo_builder/blob/aa318a20a981a7fd2fa84c77d806faa9535b2c8e/README.md), but comparing with the fish shell based builder, it has the following improvements:

1. It supports a more complex YAML config file which is an extension of the old config format and exposes more stuffs to configure.
2. `PKGBUILD`s are dumped to native data structures so everything can be done more efficiently.
2. All hashes are done using native library functions, with optional hardware acceleration.
3. All netfile sources are cached using native `ureq`, and a netfile with multiple hashes would only need to be cached once (previously a hash = a netfile).
4. All caching happens in parallel.
4. All packages would be built in their dedicated chroot that would be erased after the build. All chroots share a common underlying base chroot to save disk space.
5. Bootstrapping of chroots, extraction of sources and building of packages all happen in parallel in the same stage (so bootstrapping of A, extraction of B and building of C could all happen at the same time).
6. Supports more precise proxy and git-mirrorer uses.
7. It uses ALPM to calculate dep hashes with more exact sources (now use `gpgsign` over `sha256sum` over `md5sum` for each dep)
8. It uses libgit to more efficiently handle git repos, especially when extracting `PKGBUILD` from all kinds of different possible trees.
9. It supports `PKGBUILD` from non-default branch in an optional subtree, like the case for `archlinuxcn/repo`.
10. It does most of the things in compiled Rust, and `makepkg` would only need to `extract()` and `build()` in its Bash.

The builder has been used in the past few months and proves to be very stable. It also grants me freedom to expand the package list without worrying about breaking the builder & the board it runs on.

But this still lacks a few points, as:
1. It uses root permission to deploy and enter chroot. Although the main process drops root permission, this is still very vulnerable to potential attacks.
2. It expects built packages to be pushed to repo and used as dependencies in the next build, instead of using it right in the same build. This would result in wasted build, e.g. A and B are both dep of C, and A is dep of C. Then an update of A would trigger a build which rebuilds B and C, then in the next build C would be rebuilt again.
3. It does not maintain local build counts, and built packages always stay the same as how a user would build with the same PKGBUILD. This would result a lot of rebuilds with same versions, and users would need to update manually if deps are really broken (usually, such updates are not needed).


## Rewritten Rust builder (in development)

I'm current rewriting the Rust builder arb from ground up. It shall bring the following changes:

1. It shall act as a multi-call binary, with a parent running as normal user and map childs into namespaces with their psuedo root permissions. This also means I would need a pseudo init implementation to function in child namespaces.
2. It shall maintain an internal database of build counts, and add a incremental number to the version so dep changes always result in users updating the packages.
3. It shall handle internal dependencies properly, and a dep change from the lowest level should result in all possible packages on the dep chain being rebuilt.
4. It shall maintain an internal repo which updates along the build, and syncs to actual repo after the whole build.
5. It shall cache extracted sources and built sources, to avoid clean builds as much as possible.
6. It shall optimize the chroot package installation, to use a dedicated storage to cache all packages and verify them in one go, instead of verifying them for each installation.
7. It shall do as many things in child namespaces as possible and leak as few attack surfaces to `PKGBUILD`s as possible.

It shall have other improvements I haven't come up with, but I'll design and implement those on the way I rewrite the builder.