Bootstrapping OpenMandriva's RISC-V port with an Aarch64 host
=============================================================

Let's start out with some good news: The new OpenMandriva RISC-V port is bootstrapped, and
booting in qemu and on initial hardware.

And some even better news: It has been done the right way, which should make ports to new
architectures much easier in the future.

And to top it off, it was bootstrapped with our favorite hardware, an Aarch64 box with an
Ampere Altra CPU, with no need for boring x86 bits and pieces.


How it works
============

OpenMandriva has been making an effort to make the distribution more portable across hardware
platforms for a few years, getting rid of x86 specific bits, adapting packages to handle other
architectures, building https://github.com/OpenMandrivaSoftware/os-image-builder as a replacement
for the old image builder that could handle only x86 bootloaders and ISO9660 files, and building
crosscompiling toolchains into the system from the ground up.

We use the LLVM/Clang toolchain (which has builtin crosscompilers for everything) by default,
and our packages for the traditional toolchain are set up to build crosscompilers for all
relevant architectures (including some we aren't targeting at this time) from the same source
as the native toolchains. These crosscompiling toolchains by nature never lag behind the
native toolchains.

With that done, rpm macros had to be rewritten to allow for crosscompiling more easily.
Fortunately, these days just about everything is built with cmake, meson or autoconf, all of
which support crosscompiling if set up correctly. Also fortunately, virtually all rpm packages
these days call those build tools through macros rather than hardcoding the calls - so after
updating the rpm macros invoking these tools to detect crosscompiling and pass the right
options (including cmake and meson toolchain files) automatically, a lot of packages
crosscompile correctly without any changes.

With those changes, `rpmbuild -ba --target riscv64-linux something.spec` on an ARM box is
not a problem, nor is `rpmbuild -ba --target aarch64-linux whatever.spec` on an x86 box.

The last missing piece for bootstrapping the OS to a new architecture is building the core
packages in the right order -- sometimes multiple times with different options. For example,
for full functionality, harfbuzz needs to be built with freetype support (requiring freetype
to be built first), yet at the same time, for full functionality, freetype needs to be built
with harfbuzz support (requiring harfbuzz to be built first).

This is something that traditionally only full build-from-source distributions like Yocto can
do easily - but after tweaking packaging - primarily using `%bcond_with/%bcond_without`
conditionals, rpm is quite usable for building the same package with different feature sets
without having to maintain several copies of the package.

The last missing bit was a script that gets the needed package and the build order, along with
the right options to activate the `%bcond_with/%bcond_without` options for the early, feature
reduced, builds -- which now exists in the form of
https://github.com/OpenMandrivaSoftware/crossbuild

You tell it what you want to target (e.g. riscv64-linux) and give it some time, and (with any
luck) it produces a set of packages sufficient to get a basic OpenMandriva system up and
running, including native compilers and build tools that can be used to build extra packages
(such as the rest of the OpenMandriva repositories) natively on the target.


Virtual RISC-V boxes on aarch64
===============================

Many RISC-V devices available today are good enough for some serious work, but don't have the
performance needed to build giant packages like chromium or libreoffice natively without an
extremely patient user. (And unfortunately some of those giant packages are also the ones
resisting any attempt at crosscompiling them much more stubbornly than the core packages).

So while our fingers are itching to finally see OpenMandriva booting on the PineTab-V or the
BeagleV and VisionFive 2 boards, the first target is running in qemu on aarch64 -- allowing to
build some of the nastier packages at a reasonable speed.

Simply using qemu-static integrated with binfmt is sufficient for most things, such as running
the `%check` autotests in many packages - and this works out of the box, by just installing
the `qemu-riscv64-static` package on an OpenMandriva system.

For more serious use (including testing RISC-V kernels), an image for use with `qemu-system` is
the next (easy to reach, now that the base packages are there) target.


Next steps
==========

The next immediate step will, obviously, be integrating RISC-V builders into our build system,
and then, when more non-core packages have been built, releasing those RISC-V builds we've
been waiting for.

There's also other architectures that will be easy to port to with the new tools - not just
hardware architectures, but also operating system variants using a different libc (such as
musl in place of glibc). All tools have been built with the flexibility to allow for that
(specifying e.g. `riscv64-linuxmusl` as the target instead of `riscv64-linux` for the
normal glibc target). It will be interesting to see what effects different libcs have on
performance and memory use, especially on hardware architectures that include very low end
devices.
