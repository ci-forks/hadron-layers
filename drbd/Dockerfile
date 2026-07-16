# DRBD 9 out-of-tree kernel module + drbd-utils, packaged as a Hadron layer.
#
# DRBD 9 (from LINBIT) is distributed out-of-tree only. Compared to the in-tree
# DRBD 8 the kernel ships, it adds multi-node replication and quorum, which
# LINSTOR/Piraeus require. Because it is an *update* to a module that also
# exists in-tree, it is installed under
#     /lib/modules/$(KERNELRELEASE)/updates/
# so depmod resolves it ahead of the in-tree kernel/ copy.
#
# HADRON_TOOLCHAIN_VERSION and HADRON_VERSION default to a known-good pair but
# are build ARGs on purpose: a kernel module is tied to the exact kernel of the
# base image. To build for a newer Hadron release (which usually ships a newer
# kernel), override *both* to the same version so the module is compiled
# against, and shipped for, the kernel it will run on. In this repository's CI
# both values are supplied centrally from docker-bake.hcl.
ARG HADRON_TOOLCHAIN_VERSION=v0.5.1
ARG HADRON_VERSION=v0.5.1

# ── Kernel module ───────────────────────────────────────────────────────────
# The toolchain image ships the kernel .config, Module.symvers and the release
# strings under /usr/share/kernel-misc, but not the kernel source tree. Fetch
# the matching kernel source and run `modules_prepare` so the out-of-tree DRBD
# module can be compiled against it (the same approach Hadron uses for its own
# out-of-tree module examples).
FROM ghcr.io/kairos-io/hadron-toolchain:${HADRON_TOOLCHAIN_VERSION} AS module-builder
ARG DRBD_VERSION=9.3.3
ARG TARGETARCH
WORKDIR /build

# Kernel ARCH string used by kbuild and by DRBD's Makefile. The toolchain sets a
# global ARCH env for its own userspace builds, so we always pass this value
# explicitly to the kernel/DRBD make invocations below.
RUN case "${TARGETARCH}" in \
      amd64)   echo x86_64 > /karch ;; \
      arm64)   echo arm64  > /karch ;; \
      riscv64) echo riscv  > /karch ;; \
      *) echo "unsupported TARGETARCH=${TARGETARCH}" >&2; exit 1 ;; \
    esac

# Prepare a build tree for the exact kernel shipped by the toolchain/base image.
RUN kver=$(cat /usr/share/kernel-misc/kernel-version) && \
    series="v$(echo "${kver}" | cut -d. -f1).x" && \
    curl -fL "https://cdn.kernel.org/pub/linux/kernel/${series}/linux-${kver}.tar.xz" -o linux.tar.xz && \
    tar -xf linux.tar.xz && rm linux.tar.xz && mv linux-* linux
WORKDIR /build/linux
# The out-of-tree module is unsigned; the base kernel does not enforce module
# signatures (CONFIG_MODULE_SIG_FORCE is off), so it loads (tainted) at runtime.
# Disable signing in the build tree so `modules_prepare` never needs the
# (absent) kernel signing key. Module signing is not part of the module
# vermagic, so this does not affect loadability. LOCALVERSION is kept as-is so
# the vermagic release string matches the running kernel.
RUN cp /usr/share/kernel-misc/kernel-config .config && \
    ./scripts/config --disable MODULE_SIG --disable MODULE_SIG_ALL --disable MODULE_SIG_FORCE && \
    ARCH="$(cat /karch)" make -s olddefconfig && \
    ARCH="$(cat /karch)" make -s -j"$(nproc)" modules_prepare && \
    cp /usr/share/kernel-misc/Module.symvers .

# Build and install the DRBD 9 module from the LINBIT release tarball. The
# release tarball ships the pre-generated compat sources; the GitHub archive
# would additionally require coccinelle (spatch) at build time.
WORKDIR /build
RUN curl -fL "https://pkg.linbit.com/downloads/drbd/9/drbd-${DRBD_VERSION}.tar.gz" -o drbd.tar.gz && \
    tar -xf drbd.tar.gz && rm drbd.tar.gz && mv drbd-*/ drbd
WORKDIR /build/drbd
# kbuild manages its own compiler flags; clear the toolchain's aggressive
# size/LTO host flags so they do not leak into DRBD's build helpers.
ENV CFLAGS="" LDFLAGS=""
RUN kver=$(cat /usr/share/kernel-misc/kernel-release) && \
    ARCH="$(cat /karch)" make -j"$(nproc)" KDIR=/build/linux KVER="${kver}" && \
    ARCH="$(cat /karch)" make install DESTDIR=/dest KDIR=/build/linux KVER="${kver}" && \
    # Guarantee every DRBD .ko lands in updates/ so it overrides the in-tree
    # DRBD 8 module regardless of the tarball's default install directory.
    cd "/dest/lib/modules/${kver}" && mkdir -p updates && \
    find . -path ./updates -prune -o -name '*.ko' -print | \
      while read -r ko; do mv "${ko}" updates/; done

# ── Userspace tools ─────────────────────────────────────────────────────────
# drbdadm / drbdsetup / drbdmeta for managing the DRBD 9 module. The LINBIT
# release tarball ships a pre-generated ./configure.
FROM ghcr.io/kairos-io/hadron-toolchain:${HADRON_TOOLCHAIN_VERSION} AS utils-builder
ARG DRBD_UTILS_VERSION=9.34.0
ENV CC="gcc"
WORKDIR /build
RUN curl -fL "https://pkg.linbit.com/downloads/drbd/utils/drbd-utils-${DRBD_UTILS_VERSION}.tar.gz" -o drbd-utils.tar.gz && \
    tar -xf drbd-utils.tar.gz && rm drbd-utils.tar.gz && mv drbd-utils-* drbd-utils
WORKDIR /build/drbd-utils
# Only the DRBD 9 tooling is needed; drop legacy 8.3/8.4 support, docs and the
# cluster-manager integrations to keep the layer small.
RUN ./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc \
      --without-manual \
      --without-83support --without-84support \
      --with-systemdsystemunitdir=/lib/systemd/system \
      --with-tmpfilesdir=/lib/tmpfiles.d \
      --without-heartbeat --without-pacemaker --without-rgmanager --without-bashcompletion && \
    make -s -j"$(nproc)" && \
    make -s -j"$(nproc)" install DESTDIR=/dest && \
    rm -rf /dest/usr/share/man

# ── Assemble module index ───────────────────────────────────────────────────
# Add the freshly built module to the base image's module tree and re-run depmod
# there, so the shipped modules.dep/alias/symbols indexes list DRBD 9 from
# updates/. Only the new .ko files and the regenerated index files are kept; the
# in-tree modules already live in the base image at the same paths, so shipping
# them again would only bloat the layer.
FROM ghcr.io/kairos-io/hadron:${HADRON_VERSION} AS assemble
COPY --from=module-builder /dest/lib/modules/ /lib/modules/
RUN set -eux; kver=$(ls /lib/modules); \
    depmod "${kver}"; \
    mkdir -p "/out/lib/modules/${kver}"; \
    cp -a "/lib/modules/${kver}/updates" "/out/lib/modules/${kver}/"; \
    cp -a "/lib/modules/${kver}"/modules.* "/out/lib/modules/${kver}/"

# Final layer: the out-of-tree DRBD 9 module (in updates/), the depmod indexes
# that reference it, and the drbd-utils userspace tools. Use as a layer on top
# of Hadron:
#   FROM ghcr.io/kairos-io/hadron:VERSION
#   COPY --from=ghcr.io/kairos-io/hadron-layers/drbd:TAG / /
# Or create a sysextension with the help of Auroraboot (https://github.com/kairos-io/auroraboot)
FROM scratch AS default
COPY --from=assemble /out/ /
COPY --from=utils-builder /dest/ /

# Smoke-test the built layer against a Hadron base image. Any RUN failure here
# fails the build and gates the release step in CI. The kernel module cannot be
# inserted (the build kernel differs from the CI host kernel), so the test stays
# offline: it checks the module is present and correctly indexed under updates/,
# that depmod resolves `drbd` to the out-of-tree copy (winning over the in-tree
# DRBD 8), and that the userspace tools link and run.
# HADRON_VERSION is pinned centrally in docker-bake.hcl and bumped by Renovate
# when a newer semver tag is published to GHCR.
FROM ghcr.io/kairos-io/hadron:${HADRON_VERSION} AS test
COPY --from=default / /
RUN set -eux; kver=$(ls /lib/modules); \
    test -f "/lib/modules/${kver}/updates/drbd.ko"; \
    modinfo "/lib/modules/${kver}/updates/drbd.ko" | grep -qi '^version:'; \
    modprobe -n -v -S "${kver}" drbd 2>&1 | grep -q 'updates/drbd\.ko'; \
    drbdadm --version | grep -qi 'DRBDADM_VERSION='; \
    drbdsetup --version | grep -qi drbd; \
    drbdmeta --version | grep -qi drbd
