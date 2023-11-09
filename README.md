# Diskomator

*🦠 NVMe-TCP at your fingertips 🦠*

Diskomator is an OS-in-an-EFI-binary whose only job is to expose all
local disks as NVMe-TCP network block devices, as they appear.

When built, it results in a single UEFI binary, that embeds a full OS
root file system so that it does not need any further disk access
while it runs. The EFI binary can be invoked directly from the UEFI
firmware, for example by placing it an EFI System Partition (ESP). The
OS root file system contains:

1. A Fedora Linux OS tree

2. A current version of systemd (v255-rc2),
including
[systemd-storagetm](https://www.freedesktop.org/software/systemd/man/latest/systemd-storagetm.html)
and
[systemd-networkd](https://www.freedesktop.org/software/systemd/man/latest/systemd-networkd.html).

3. An SSH server (just to make things easy to debug)

4. The Plymouth boot splash tool to make things pretty at boot and during runtime.

The image is built via [`mkosi`](https://github.com/systemd/mkosi).

All this then does is boot up into a minimal mode where
`systemd-storagetm` and `systemd-networkd` are running. The former
exposes all local block devices via NVMe-TCP, the latter configures
all local network devices.

The resulting EFI binary is relatively large (~300M), because it
embeds all kinds of network drivers and graphics devices, plus their
firmware. To keep things simply this stays close to upstream Fedora,
without any attempts to minimize footprint.

## Why Even?

My personal usecase for this goes something like this: I build
immutable OS images for physical systems regularly and try them out. I
could always write them to an USB stick on my development machine and
then unplug it, and plug it into my testing machine. But that's
cumbersome. My way out: just have a way how the test machine's disk
can be written to directly from my development machine. And that's
what Diskomator is.

Other usecases:
* Debugging
* Remote installation of OSes
* Turn your 2000 USD laptop into a very expensive USB stick
* It's a fantastic show-case for UKIs, `mkosi`, Linux and `systemd` I think

## Caveats

* ⚠️ This currently does not enable NVME authentication nor
  encryption. If you boot from this your disk will be readable and
  writable to anyone with access to your local network!

* ⚠️ The `root` user is accessible via SSH with the password *test*,
  again to anyone with access to your local network!

* ⚠️ A debug shell is always available on Alt-F9.

* This requires an EFI system, with a bit of RAM. After all the OS
  is entirely kept in memory.

* The resulting EFI binary is not SecureBoot signed, you thus have to
  disable SecureBoot if you want to use this. (You can easily add that
  though, if you have a suitable key pair. `mkosi` will help with
  that, see documentation.)

## Building

You'll need:

1. A build tree of a current systemd development version, to get the
   newest version of `systemd-repart`. Once distributions started to
   regularly ship v255, this manual step will go away. On a Fedora
   system you'd do it like this:

   ```
   sudo dnf builddep systemd
   git clone https://github.com/systemd/systemd.git
   cd systemd
   meson build
   ninja -C build
   ```

   And with that you should have a version of systemd built in the
   `build` sub-directory. No need to install this, BTW, just keep it
   around, we can use the build tree directly in step 4.

2. v19 of [`mkosi`](https://github.com/systemd/mkosi) or newer. If
   your distribution doesn't have that yet, you can trivially check it
   out too:

   ```
   sudo dnf builddep mkosi
   git clone https://github.com/systemd/mkosi.git
   ```

3. A checkout of `diskomator`:

   ```
   git clone https://github.com/poettering/diskomator.git
   cd diskomator
   git submodule update --init
   ```

4. Now edit `mkosi.conf` in the diskomator directory locally on your
   system, and adjust the (by default commented) `ExtraSearchPaths=`
   line to point to the build tree from step 1.

5. You are now ready to build the image. In the `diskomator` git
   checkout run:

   ```
   sudo ../mkosi/bin/mkosi -T -i -f build
   ```

   (The `-T -i -f` you can theoretically drop BTW, I only specify them
   here since it improves rebuild times in case you hack on this.)

   Adjust the `mkosi` path to match where you placed the checkout tree
   from step 3.

   Once this completes you'll have two things in the `mkosi.output/`
   subdirectory: `diskomator.efi` and `diskomator.raw`. The former is
   the EFI binary that we care about. The latter is a GPT disk image
   with an ESP with that very EFI binary in it (and no other
   partitions). The latter you can directly `dd` to an USB stick if
   you like, to boot another system from.

   You can even let `mkosi` do the `dd`'ing for you. Which is actually
   a good idea, since it will make sure the image is adapted to your
   chosen target device's sector and disk size 🔥🔥🔥:

   ```
   sudo ../mkosi/bin/mkosi burn /dev/disk/by-id/usb-SanDisk_Ultra_Fit_4C530000190505109123-0\:0
   ```

   Replace the last argument in that command line by the path to the
   device node you want to write this to. As you can see I have a
   SanDisk USB stick I am testing this with.

And that's really all.

Once systemd v255 and mkosi v19 entered the various distributions step
1, 2 and 4 will become unnecessary (and the git submodule thing from
step 3). Once that happened you only have to do 3 and 5.

## Future

I'd like to live to see a future where people build appliances like
this for various purposes, not just this specific NVMe one. For
example, a nice thing to have would be an appliance whose only job is
to make all local displays available via Miracast. I hope this
repository is inspiration enough for an interested soul, to get this
off the ground.

Ideally, distributions would build images like this on their
own. Specifically, I'd be delighted if Fedora (for example) would
build an image like this and SecureBoot sign it, within their own
build infrastructure and make that an offering to their users.

In the meantime it might be nice to build diskomator on the usually
available Open Source build infrastructure somewhere, so that people
can just download a `.raw` or `.efi` file, instead of the cumbersome
build steps listed above. Anyone interested in setting this up?

Anyway, I hope this piqued your interest, now run and do with all this
whatever you want!
