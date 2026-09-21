# Retro arcade -- a Linux desktop streamed to the browser over WebRTC, used to
# run emulator front-ends and legacy 32-bit Windows applications.
#
# Base is LinuxServer's Selkies image: an Ubuntu 24.04 desktop that serves a
# WebRTC HTML5 client on :3000, with audio muxed into the same stream. That
# matters -- plain VNC has no audio channel at all, and bolting PulseAudio onto
# it via Guacamole gives you two unsynchronised TCP streams.
FROM ghcr.io/linuxserver/baseimage-selkies:ubuntunoble

# winetricks + cabextract: some legacy Windows applications are MFC programs,
# and Wine cannot ship Microsoft's MFC42.DLL for licensing reasons. winetricks
# installs it from Microsoft's Visual C++ 6 redistributable (`winetricks mfc42`);
# cabextract is what winetricks uses to unpack it.
#
# OpenGL matters more than it looks: Wine's Direct3D layer (wined3d) translates
# D3D calls into OpenGL, so with no libGL.so.1 present a DirectX application
# cannot create ANY graphics context and dies with an opaque fatal error. The
# base desktop image ships no GL at all. Mesa's software rasteriser is ample for
# late-1990s software -- and the i386 copies are the ones that count, since
# those applications are 32-bit and would otherwise find only the 64-bit libs.
#
# 32-bit Windows binaries (PE32) need 32-bit Wine, and 32-bit Wine means i386
# multiarch, which Ubuntu 24.04 does not enable by default.
RUN dpkg --add-architecture i386 && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        wine \
        wine32:i386 \
        wine64 \
        winbind \
        scummvm \
        dosbox-x \
        retroarch \
        libretro-core-info \
        libretro-nestopia \
        libretro-gambatte \
        libretro-genesisplusgx \
        p7zip-full \
        unzip \
        mame \
        xterm \
        openbox \
        x11-xserver-utils \
        x11-utils \
        hsetroot \
        winetricks \
        cabextract \
        libgl1 \
        libglx-mesa0 \
        libgl1-mesa-dri \
        libgl1:i386 \
        libglx-mesa0:i386 \
        libgl1-mesa-dri:i386 \
    && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# python3-tk in its own layer, with the import asserted inline.
#
# It was originally in the big apt list above, and a build reused a CACHED
# layer that did not contain it -- the smoke test then failed and the cause was
# not visible from the build output. Whatever the cache did, this makes the
# outcome unambiguous: if tkinter cannot be imported by the SYSTEM python (the
# launcher shebangs /usr/bin/python3, not LinuxServer's /lsiopy/bin/python3),
# the BUILD fails here rather than the launcher silently refusing to start.
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3-tk && \
    /usr/bin/python3 -c "import tkinter; print('tkinter OK', tkinter.TkVersion)" && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Version-pinned libretro MAME cores. A mixed-era arcade library does not run
# under any single MAME version, so several are installed side by side and a
# routing map (romset -> version, produced by an external compatibility sweep
# that is not part of this repo) tells launch-mame which one to use.
ARG LIBRETRO_URL=https://buildbot.libretro.com/nightly/linux/x86_64/latest
RUN mkdir -p /opt/cores && cd /opt/cores && \
    for c in mame2003_plus mame2010 mame2016; do \
        curl -fsSL -o "$c.zip" "${LIBRETRO_URL}/${c}_libretro.so.zip" && \
        unzip -oq "$c.zip" && rm -f "$c.zip"; \
    done && ls -la /opt/cores

# Ubuntu ships emulators in /usr/games, which is NOT on PATH in a bare container
# shell or in the openbox session. The launcher menu calls `scummvm` by bare
# name, so without this the menu entry silently does nothing when clicked --
# the build smoke test caught exactly that.
ENV PATH="/usr/games:${PATH}"

# Openbox root menu is the launcher, and /defaults/autostart is the session hook.
COPY root/ /

# /defaults must be traversable by abc, the uid every service and app runs as.
# The base image keeps the PulseAudio socket at /defaults/native, and connect()
# to a unix socket needs search permission on EVERY directory in its path -- the
# socket's own srwxrwxrwx is not enough. When /defaults is 0700 root:root the
# call fails with EACCES, which libpulse reports as the thoroughly misleading
# "Connection refused", so Selkies' pcmflux capture dies and NOTHING has sound.
# It hides well: a root shell tests fine, because root bypasses the check.
# COPY root/ / can stamp the build agent's umask onto this directory, so set the
# mode explicitly instead of trusting whatever the checkout happened to produce.
RUN chmod 0755 /defaults

# HTTP client. 3001 is the base image's TLS listener; TLS is terminated at a
# reverse proxy in front of this instead, so only 3000 is published.
EXPOSE 3000
