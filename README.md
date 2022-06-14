# Frequently Asked Questions

As stewards of the official images and maintainers of many images ourselves, we often see a lot of questions that surface repeatedly. This repository is an attempt to gather some of those and provide some answers!

## Table of Contents

<!-- AUTOGENERATED TOC -->

1.	[Frequently Asked Questions](#frequently-asked-questions)
	1.	[Table of Contents](#table-of-contents)
	2.	[General Questions](#general-questions)
		1.	[What do you mean by "Official"?](#what-do-you-mean-by-official)
		2.	[An image's source changed in Git, now what?](#an-images-source-changed-in-git-now-what)
		3.	[How are images built? (especially multiarch)](#how-are-images-built-especially-multiarch)
		4.	[Inclusion Criteria](#inclusion-criteria)
		5.	[What is `bashbrew`? Where can I download it?](#what-is-bashbrew-where-can-i-download-it)
		6.	[What do you mean by "Supported"?](#what-do-you-mean-by-supported)
		7.	[What's the difference between "Shared" and "Simple" tags?](#whats-the-difference-between-shared-and-simple-tags)
	3.	[Image Building](#image-building)
		1.	[Why does my security scanner show that an image has CVEs?](#why-does-my-security-scanner-show-that-an-image-has-cves)
		2.	[Why do so many official images build from source?](#why-do-so-many-official-images-build-from-source)
		3.	[`HEALTHCHECK`](#healthcheck)
		4.	[OpenPGP / GnuPG Keys and Verification](#openpgp--gnupg-keys-and-verification)
		5.	[Multi-stage Builds](#multi-stage-builds)
		6.	[Why isn't there a Windows equivalent of `docker-entrypoint.sh`?](#why-isnt-there-a-windows-equivalent-of-docker-entrypointsh)
		7.	[Can I use a bot to make my image update PRs?](#can-i-use-a-bot-to-make-my-image-update-prs)
	4.	[Image Usage](#image-usage)
		1.	[`--link` is deprecated!](#--link-is-deprecated)

<!-- AUTOGENERATED TOC -->

## General Questions

### What do you mean by "Official"?

The name of this program was chosen in an attempt to reflect our upstream-first focus (although in hindsight, it's clear that it was a choice with some amount of confusion potential for which we're sorry).

See [the readme of the `github.com/docker-library/official-images` repository](https://github.com/docker-library/official-images#what-are-official-images) for a more verbose overview of the program.

### An image's source changed in Git, now what?

Let's walk through the full lifecycle of a change to an image to help explain the process better. We'll use the [`golang`](https://hub.docker.com/_/golang) image as an example to help illustrate each step.

1.	a change gets committed to the relevant image source Git repository (either via direct commit, PR, or [some automated process](https://doi-janky.infosiftr.net/job/update.sh/) -- somehow some change is committed to the Git repository for the image source)

	-	for [`golang`](https://hub.docker.com/_/golang), that would be a commit to [the `github.com/docker-library/golang` repository](https://github.com/docker-library/golang), such as [f12c995 (the update to Go 1.16.3)](https://github.com/docker-library/golang/commit/f12c995e27fef88ccb984605ab4748737ae3a778)

2.	a PR to the relevant `library/xxx` manifest file is created against https://github.com/docker-library/official-images (which is the source-of-truth for the official images program as a whole)

	-	for [f12c995](https://github.com/docker-library/golang/commit/f12c995e27fef88ccb984605ab4748737ae3a778), that PR would be [docker-library/official-images#9904](https://github.com/docker-library/official-images/pull/9904) ([changing `library/golang` within that repository](https://github.com/docker-library/official-images/pull/9904/files) to point to the updated commits from [the `github.com/docker-library/golang` repository](https://github.com/docker-library/golang) and updating `Tags:` references if applicable)

3.	that PR and a full diff of the actual `Dockerfile` and related [build context](https://docs.docker.com/engine/reference/commandline/build/#extended-description) files are then reviewed by [the official images maintainers](https://github.com/docker-library/official-images/blob/master/MAINTAINERS)

	-	for [docker-library/official-images#9904](https://github.com/docker-library/official-images/pull/9904), that can be seen in [the "Diff:" comment](https://github.com/docker-library/official-images/pull/9904#issuecomment-812221459)

4.	a basic build test is produced (by GitHub Actions) on `amd64` (to ensure that it will likely build properly on the real build servers if accepted, and to run [a small series of official images tests](https://github.com/docker-library/official-images/tree/master/test) against the built image)

	-	for [docker-library/official-images#9904](https://github.com/docker-library/official-images/pull/9904), that can be seen in [the "16 checks passed"](https://github.com/docker-library/official-images/pull/9904#event-4543748220)

5.	once merged, [the official images build infrastructure](#how-are-images-built-especially-multiarch) will pick up the changes and build and push to the relevant per-architecture repositories (`amd64/xxx`, `arm64v8/xxx`, etc)

	-	for [`golang`](https://hub.docker.com/_/golang), those per-architecture build jobs can be seen in [the "golang" view in the build infrastructure](https://doi-janky.infosiftr.net/job/multiarch/view/images/view/golang/)

6.	after those jobs push updated artifacts to the architecture-specific repositories ([`amd64/xxx`](https://hub.docker.com/u/amd64), [`arm64v8/xxx`](https://hub.docker.com/u/arm64v8), etc), [a separate job](https://doi-janky.infosiftr.net/job/put-shared/) collects those updates into ["index" objects](https://github.com/opencontainers/image-spec/blob/v1.0.1/image-index.md) (also known as ["manifest lists"](https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list)) under [`library/xxx`](https://hub.docker.com/u/library) (which is [the "default" namespace within Docker](https://github.com/docker/docker-ce/blob/v18.09.6/components/engine/vendor/github.com/docker/distribution/reference/normalize.go))

	-	for [`golang`](https://hub.docker.com/_/golang), that would be [the `put-shared/light/golang` job](https://doi-janky.infosiftr.net/job/put-shared/job/light/job/golang/)

For images [maintained by the docker-library team](https://github.com/docker-library), we typically include a couple useful scripts in the repository itself, like `./update.sh` and `./generate-stackbrew-library.sh`, which help with automating simple version bumps via `Dockerfile` templating, and generating the contents of the `library/xxx` manifest file, respectively. [We also have infrastructure which performs those version bumps along with a build and test](https://doi-janky.infosiftr.net/job/update.sh/) and commits them directly to the relevant image repository (which is exactly how [the illustrative `golang` a9171b commit](https://github.com/docker-library/golang/commit/a9171b851ba926f2979e8c711d25faa025194def) referenced above was created).

### How are images built? (especially multiarch)

Images are built via a [semi-complex Jenkins infrastructure](https://doi-janky.infosiftr.net/), and the sources for much of that can be found in [the `github.com/docker-library/oi-janky-groovy` repository](https://github.com/docker-library/oi-janky-groovy).

The actual infrastructure is a combination of machines provided by our generous donors:

-	`amd64`, `arm64v8`, `i386`, Jenkins nodes: [Docker, Inc.](https://www.docker.com/) and [Oracle Cloud Infrastructure](https://www.oracle.com/cloud/)
-	`arm32vN`: [WorksOnArm](https://github.com/WorksOnArm/cluster/issues/7)
-	`mips64le`: [Loongson](http://www.loongson.cn/)
-	`ppc64le`, `s390x`: [IBM](https://www.ibm.com/)
-	`riscv64`: [Debian](https://www.debian.org/) and [Tianon](https://tianon.xyz)

For a more complete view of the full image change/publishing process, see ["An image's source changed in Git, now what?"](#an-images-source-changed-in-git-now-what) above.

### Inclusion Criteria

See ["What are 'Official Images'?" in the main project repository](https://github.com/docker-library/official-images#what-are-official-images) for a high-level overview of the focus and goals of the project in general.

Per [the "New Image Checklist"](https://github.com/docker-library/official-images/blob/master/NEW-IMAGE-CHECKLIST.md) (which is used to roughly track status during "New Image" reviews), one of the primary determinations we try to make is whether the image being proposed is "generally useful" and whether the software itself is "reasonably popular" and/or "solves a particular use case well" (to help focus our review bandwidth on things that will be helpful to as large a set of users as possible).

There are also specific `Dockerfile` writing guidelines which can be found in [the "Review Guidelines" section of our documentation](https://github.com/docker-library/official-images#review-guidelines), which is used as a basis for a lot of our review process.

### What is `bashbrew`? Where can I download it?

The `bashbrew` tool is one built by the official images team for the purposes of building and pushing the images. At a very high level, it's a wrapper around `git` and `docker build` in order to help us manage the various `library/xxx` files in the main official images repository in a simple and repeatable way (especially focused around using explicit Git commits in order to achieve maximum repeatability and `Dockerfile` source change reviewability).

The source code is in [the `github.com/docker-library/bashbrew` repository](https://github.com/docker-library/bashbrew). Precompiled artifacts (which are used on the official build servers) can be downloaded from [the relevant Jenkins job](https://doi-janky.infosiftr.net/job/bashbrew/job/master/lastSuccessfulBuild/artifact/) or [the GitHub releases](https://github.com/docker-library/bashbrew/releases).

### What do you mean by "Supported"?

On every image description, there is a section entitled "Supported tags and respective `Dockerfile` links" (for example, see [`debian`'s Hub page](https://hub.docker.com/_/debian/)).

Within the Official Images program, we use the word "Supported" to mean something like actively maintained. To put that another way, a particular 2.5.6 software release is considered supported if a severe bug being found would cause a 2.5.7 release (and once 2.5.7 is released, 2.5.6 is no longer considered supported, but the Docker Hub tag is typically left available for pulling -- it will simply never get rebuilt after that point given that it is unsupported).

See [the "Library definition files" section](https://github.com/docker-library/official-images#library-definition-files) of our maintainer documentation for more details.

### What's the difference between "Shared" and "Simple" tags?

Some images have separated "Simple Tags" and "Shared Tags" sections under "Supported tags and respective `Dockerfile` links" (see [the `mongo` image](https://hub.docker.com/_/mongo/) for an example).

"Simple Tags" are instances of a "single" Linux or Windows image. It is often a manifest list that can include the same image built for other architectures; for example, `mongo:4.0-xenial` currently has images for `amd64` and `arm64v8`. The Docker daemon is responsible for picking the appropriate image for the host architecture.

"Shared Tags" are tags that always point to a manifest list which includes some combination of potentially multiple versions of Windows and Linux images across all their respective images' architectures -- in the `mongo` example, the `4.0` tag is a shared tag consisting of (at the time of this writing) all of `4.0-xenial`, `4.0-windowsservercore-ltsc2016`, `4.0-windowsservercore-1709`, and `4.0-windowsservercore-1803`.

The "Simple Tags" enable `docker run mongo:4.0-xenial` to "do the right thing" across architectures on a single platform (Linux in the case of `mongo:4.0-xenial`). The "Shared Tags" enable `docker run mongo:4.0` to roughly work on both Linux and as many of the various versions of Windows that are supported (such as Windows Server Core LTSC 2016, where the Docker daemon is again responsible for determining the appropriate image based on the host platform and version).

## Image Building

<a id="why-does-my-security-scanner-show-that-an-image-has-so-many-cves"></a>

### Why does my security scanner show that an image has CVEs?

Though not every CVE is removed from the images, we take CVEs seriously and try to ensure that images contain the most up-to-date packages available within a reasonable time frame. For many of the Official Images, a security scanner, like [Docker Security Scanning](https://docs.docker.com/docker-hub/official_images/#official-image-vulnerability-scanning) or [Clair](https://github.com/coreos/clair) might show CVEs, which can happen for a variety of reasons:

-	The CVE has not been addressed in that particular image

	-	Upstream maintainers don't consider a particular CVE to be a vulnerability that needs to be fixed and so won't be fixed.
		-	e.g., [CVE-2005-2541](https://nvd.nist.gov/vuln/detail/CVE-2005-2541) is considered a High severity vulnerability, but in [Debian](https://security-tracker.debian.org/tracker/CVE-2005-2541) is considered “intended behavior,” making it a feature, not a bug.
	-	The OS Security team only has so much available time and has to deprioritize some security fixes over others. This could be because the threat is considered low or that it is too intrusive to backport to the version in "stable".

		e.g., [CVE-2017-15804](https://nvd.nist.gov/vuln/detail/CVE-2017-15804) is considered a High severity vulnerability, but in [Debian](https://security-tracker.debian.org/tracker/CVE-2017-15804) it is marked as a "Minor issue" in Stretch and no fix is available.

	-	Vulnerabilities may not have an available patch, and so even though they've been identified, there is no current solution.

-	The listed CVE is a false positive

	-	In order to provide stability, most OS distributions take the fix for a security flaw out of the most recent version of the upstream software package and apply that fix to an older version of the package (known as backporting).

		e.g., [CVE-2020-8169](https://nvd.nist.gov/vuln/detail/CVE-2020-8169) shows that `curl` is flawed in versions `7.62.0` though `7.70.0` and so is fixed in `7.71.0`. The version that has the fix applied in Debian Buster is `7.64.0-4+deb10u2` (see [security-tracker.debian.org](https://security-tracker.debian.org/tracker/CVE-2020-8169) and [DSA-4881-1](https://www.debian.org/security/2021/dsa-4881)).

	The security scanners can't reliably check for CVEs, so it uses heuristics to determine whether an image is vulnerable. Those heuristics fail to take some factors into account:

	-	Is the image affected by the CVE at all? It might not be possible to trigger the vulnerability at all with this image.
	-	If the image is not supported by the security scanner, it uses wrong checks to determine whether a fix is included.
		-	e.g., For RPM-based OS images, the Red Hat package database is used to map CVEs to package versions. This causes severe mismatches on other RPM-based distros.
		-	This also leads to not showing CVEs which actually affect a given image.

We strive to publish updated images at least monthly for Debian and Ubuntu. We also rebuild earlier if there is a critical security need, e.g. [docker-library/official-images#2171](https://github.com/docker-library/official-images/issues/2171). Many Official Images are maintained by the community or their respective upstream projects, like Alpine and Oracle Linux, and are subject to their own maintenance schedule. These refreshed base images also means that any other image in the Official Images program that is `FROM` them will also be rebuilt (as described in [the project `README.md` file](https://github.com/docker-library/official-images#library-definition-files)).

It is up to individual users to determine whether not a CVE applies to how you are running your service and is beyond the scope of the FAQ.

Parts of this FAQ entry are inspired by [a Google Cloud blog post](https://cloud.google.com/blog/products/containers-kubernetes/exploring-container-security-let-google-do-the-patching-with-new-managed-base-images) (specifically their "Working with managed base images" section), which has additional information which may be useful or relevant.

Related issues: [docker-library/buildpack-deps#46](https://github.com/docker-library/buildpack-deps/issues/46#issuecomment-242863442), [docker-library/official-images#2740](https://github.com/docker-library/official-images/issues/2740#issuecomment-286253279)

### Why do so many official images build from source?

The tendency for many official images to build from source is a direct result of trying to closely follow each upstream's official recommendations for how to deploy and consume their product/project.

For example, the PostgreSQL project publishes (and recommends the use of) their own official `.deb` packages, so [the `postgres` image](https://hub.docker.com/_/postgres/) builds directly from those (from http://apt.postgresql.org/).

On the flip side, the PHP project will only officially support users who are using the latest release (https://bugs.php.net/, "Make sure you are using the latest stable version or a build from Git"), which the distributions do not provide. Additionally, their "installation" documentation describes building from source as the officially supported method of consuming PHP.

One common result of this is that Alpine-based images are almost always required to build from source because it is somewhat rare for an upstream to provide "official" binaries, but when they do they're almost always in the form of something linked against glibc and as such it is very rare for Alpine-compatible binaries to be published (hence most Alpine images building from source).

So to summarize, there isn't an "official images" policy one way or the other regarding building from source; we leave it up to each image maintainer to make the appropriate judgement on what's going to be the best representation / most supported solution for the upstream project they're representing.

### `HEALTHCHECK`

Explicit health checks are not added to official images for a number of reasons, some of which include:

-	many users will have their own idea of what "healthy" means and credentials change over time making generic health checks hard to define
-	after upgrading their images, current users will have extra unexpected load on their systems for healthchecks they don't necessarily need/want and may be unaware of
-	Kubernetes does not use Docker's heath checks (opting instead for separate [`liveness` and `readiness` probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/))
-	sometimes things like databases will take too long to initialize, and a defined health check will often cause the orchestration system to prematurely kill the container ([docker-library/mysql#439 for instance](https://github.com/docker-library/mysql/issues/439))

The [docker-library/healthcheck repository](https://github.com/docker-library/healthcheck) is to serve as an example for creating your own image derived from the prototypes present. They serve to showcase the best practices in creating your own healthcheck for your specific task and needs.

### OpenPGP / GnuPG Keys and Verification

Ideally, images that require downloaded artifacts [should use some cryptographic signature to verify that the artifacts are what we expect them to be](https://github.com/docker-library/official-images#image-build) (mostly from a provenance perspective, but also from a network transmission perspective). Many open source projects publish PGP signatures (typically as a "detached" siganture file) which can be used for the purpose of verifying artifact provenance (with the theory being that only the correct publishers of said artifact are in possession of the private key material required to create said signature).

The way we typically recommend image maintainers fetch those public keys to verify said artifacts is via `gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys XXXXX` (where `XXXXX` gets replaced with the *full* key fingerprint, as in `97FC712E4C024BBEA48A61ED3A5CA953F73C700D`). This will use [the keys.openpgp.org service](https://keys.openpgp.org/about), which does require additional verification in order to be used in this way. If that additional verification is not possible/desirable for the keys in question, we recommend using `hkps://keyserver.ubuntu.com` instead (which is a more "classical" key service).

Historically, we recommended the use of [a suitable server pool from `sks-keyservers.net`, but that service has since been shut down](http://web.archive.org/web/20220119094712/https://www.sks-keyservers.net/). See also [github.com/tianon/pgp-happy-eyeballs](https://github.com/tianon/pgp-happy-eyeballs), which is no longer recommended for use (see [tianon/pgp-happy-eyeballs#4](https://github.com/tianon/pgp-happy-eyeballs/issues/4) for more discussion/details).

Another common solution to this problem is to simply check a `KEYS` file into Git that contains the public keys content (see [Apache Ant's `KEYS` file](https://www.apache.org/dist/ant/KEYS) for an example). The primary downsides of this are that it's a pain during the Official Images review process (since every added/removed `KEYS` entry is many lines of what essentially is just noise to the image `diff`) but more importantly that it becomes much more difficult for users to then *verify* that the key being checked is one that upstream officially publishes (it's fairly common for upstreams to officially publish key fingerprints, as seen in [RabbitMQ's "Signatures" page](https://www.rabbitmq.com/signatures.html)).

Additionally, any usage of the GnuPG command-line tool (`gpg`) [should include the `--batch` command-line flag](https://bugs.debian.org/913614#27) (to enable what is essentially GnuPG's "API" mode).

### Multi-stage Builds

Following [docker-library/official-images#5929](https://github.com/docker-library/official-images/pull/5929), multi-stage builds are officially supported by the official-images build tooling, and tentatively approved for use.

The main caveat of that change is outlined in [docker-library/official-images#5929 (comment)](https://github.com/docker-library/official-images/pull/5929#discussion_r285316415), namely that we don't have a clean way to preserve the cache for the intermediate stages of a proper multi-stage image, and as such they should be used sparingly. As such, we've come up with several guidelines to help image maintainers determine whether their use of multi-stage builds is one that's likely to be accepted during image review:

1.	only a single `FROM`, but potentially multiple `COPY --from=xxx:yyy ...` copying from other tagged official images; for example:

	-	a `tomcat` image doing `FROM openjdk:XXX-jre` followed by `COPY --from=tomcat:XXX-jdk /path/to/compiled/tomcat/native ...` to get the compiled "Tomcat Native" artifacts for a JRE-based image out of the JDK-based counterpart

	-	a Windows Nano Server image copying artifacts from the larger Windows Server Core variant to overcome the lack of PowerShell for downloading/installing artifacts

2.	two-stage build where the necessary artifact does not exist and must be built from source and/or the build process is going to be similarly highly deterministic (thus mitigating the cache concern somewhat); for example:

	-	a Go project without official binary releases (although it is highly recommended for something trivial like Go to publish actual official release binaries, especially if the Go version required/supported for building is highly specific [given that Go only supports two major releases at a time](https://golang.org/doc/devel/release.html#policy))

	-	using [`jlink` from a JDK 9+ image](https://docs.oracle.com/javase/9/tools/jlink.htm) to create an image with a minimal JRE that contains only the necessary components for the contained application

It is also worth pointing out [moby/moby#37830 (no sticky bits)](https://github.com/moby/moby/issues/37830), [moby/moby#37123 (no ownership preservation until 19.03+)](https://github.com/moby/moby/issues/37123), and [moby/moby#36759 (no `ADD --from=xxx`)](https://github.com/moby/moby/issues/36759), so multi-stage builds are not currently supported/useful for "base" images like `ubuntu`.

### Why isn't there a Windows equivalent of `docker-entrypoint.sh`?

This is an unfortunate design limitation of Windows. On Linux, we have the `exec` family of system calls (and a Bash built-in by the same name) that allows us to completely replace our current running process with another. This is what allows us to run an `ENTRYPOINT` script which performs some initialization logic, then replaces itself with the actual server/application process directly (so that Docker can track that process properly). On Windows, [that interface doesn't really exist](https://serverfault.com/a/567393/58240) (and [is really difficult to emulate properly](https://stackoverflow.com/q/51185115/433558)), which means that in order to even begin to replicate this behavior, we'd have to implement a process monitor as well to sit between Docker and and the server/application process for the lifetime of the session (all for some simple initialization behavior, which is kind of a heavy toll).

### Can I use a bot to make my image update PRs?

Sure! Just a few simple guidelines/requests:

1.	don't make them "too often"
	-	multiple times per week is definitely "too often"
	-	once every week is a bit on the boundary, but acceptable
	-	serious security issues override this
2.	make sure the PR @-references any/all human maintainers so you see our review comments
3.	only one PR at a time, please (the easiest way to accomplish this is to use the same branch name every time)
4.	base every new PR off the `master` branch from https://github.com/docker-library/official-images, *not* from your fork (especially important if we squash-merge your PRs)

## Image Usage

### `--link` is deprecated!

> The reports of `--link`'s death are greatly exaggerated.
>
> \-*Mark Twain, probably*

The [documentation for "legacy container links" (`--link`)](https://docs.docker.com/network/links/) include a large warning about `--link` potentially going away at some point, but there is no timeline given and this "soft deprecation" has been going strong for a very long time. Their usage is definitely discouraged, but we expect Docker will continue to support them for quite some time.

Many sources of image documentation use `--link` in their examples for simplicity, including not needing to detail Docker network management, and `--link`'s feature of inherently exchanging connection information to the linked containers as environment variables.

Several of the official images were updated in [docker-library/docs#1441](https://github.com/docker-library/docs/pull/1441) with the compromise of using `--network some-network` in an attempt to convey to users that additional effort will be required for them to connect their services successfully (implying that they should go read documentation / learn about Docker's container networking functionality).
