# Libvirt & QEMU (QCOW2) on macOS with ARM64 M1

> based on: https://www.naut.ca/blog/2021/12/09/arm64-vm-on-macos-with-libvirt-qemu/

## Installation

```bash
# install homebrew package manager for macOS
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# install dependencies
brew install qemu gcc

cat <<EOT >> libvirt.rb
class Libvirt < Formula
  desc "C virtualization API"
  homepage "https://libvirt.org/"
  url "https://libvirt.org/sources/libvirt-7.10.0.tar.xz"
  sha256 "cb318014af097327928c6e3d72922e3be02a3e6401247b2aa52d9ab8e0b480f9"
  license all_of: ["LGPL-2.1-or-later", "GPL-2.0-or-later"]
  head "https://gitlab.com/libvirt/libvirt.git", branch: "master"

  livecheck do
    url "https://libvirt.org/sources/"
    regex(/href=.*?libvirt[._-]v?(\d+(?:\.\d+)+)\.t/i)
  end

  bottle do
    sha256 arm64_monterey: "9a3965e2aeac2e96d1dd8d145d1b816b3b2b58a991e9439a7eedb29b08669598"
    sha256 arm64_big_sur:  "1f0aeb8d5f8290bb9c4f934120c686000f9e4a44e754b4af7a060940c24b23da"
    sha256 monterey:       "69e8451e0d24773bb7212d46a96fa99fdd9ac0fe827c10c87b256adf64ca98a7"
    sha256 big_sur:        "2ff2402e0b97c6db4c18a70bd0afbe464c6d041668c3b43ee24edbc5c4ec3b2d"
    sha256 catalina:       "be50ad0a90b7140d72c6ffc8cad5743a1277c2ceedb68e55b7e661f7a2d771a8"
    sha256 x86_64_linux:   "1d45a7ce93bcb5dac4878e2d7059bbcc0d15a31921f19b69834ec91f68bae098"
  end

  depends_on "docutils" => :build
  depends_on "meson" => :build
  depends_on "ninja" => :build
  depends_on "perl" => :build
  depends_on "pkg-config" => :build
  depends_on "python@3.9" => :build
  depends_on "gettext"
  depends_on "glib"
  depends_on "gnu-sed"
  depends_on "gnutls"
  depends_on "grep"
  depends_on "libgcrypt"
  depends_on "libiscsi"
  depends_on "libssh2"
  depends_on "yajl"

  uses_from_macos "curl"
  uses_from_macos "libxslt"

  on_macos do
    depends_on "rpcgen" => :build
  end

  on_linux do
    depends_on "libtirpc"
  end

  def install
    inreplace "src/qemu/qemu_command.c", "qemuBuildAccelCommandLine(cmd, def);", ""
    mkdir "build" do
      args = %W[S
        --localstatedir=#{var}
        --mandir=#{man}
        --sysconfdir=#{etc}
        -Ddriver_esx=enabled
        -Ddriver_qemu=enabled
        -Ddriver_network=enabled
        -Dinit_script=none
        -Dqemu_datadir=#{Formula["qemu"].opt_pkgshare}
      ]
      system "meson", *std_meson_args, *args, ".."
      system "meson", "compile"
      system "meson", "install"
    end
  end

  service do
    run [opt_sbin/"libvirtd", "-f", etc/"libvirt/libvirtd.conf"]
    keep_alive true
    environment_variables PATH: HOMEBREW_PREFIX/"bin"
  end

  test do
    if build.head?
      output = shell_output("#{bin}/virsh -V")
      assert_match "Compiled with support for:", output
    else
      output = shell_output("#{bin}/virsh -v")
      assert_match version.to_s, output
    end
  end
end
EOT

brew install --build-from-source --debug ./libvirt.rb
rm ./libvirt.rb

# disable qemu's security features (not supportex by macOS)
echo 'security_driver = "none"' >> /opt/homebrew/etc/libvirt/qemu.conf
echo "dynamic_ownership = 0" >> /opt/homebrew/etc/libvirt/qemu.conf
echo "remember_owner = 0" >> /opt/homebrew/etc/libvirt/qemu.conf

# start libvirt (also starts it after boot)
brew services start libvirt
```

## Create VM's

- Download the desired Linux Server `.iso` to `./iso/`.
- Copy the `./xml/fedora35.xml` and adjust it to your needs:
    - set a new `name` on line 2
    - set a unique `uuid` on line 3
    - set the `path` to you disk image on line 21 (will be created below)
    - set the `path` to your iso file on line 27
    - set a `port` (e.g. 5900) to connect to the vm using vnc on line 36
    - set a `port` (e.g. 2222) to connect to the vm using ssh on line 48
    - additionally expose ports (e.g. 3000 for development servers)

```bash
# create a disk image
qemu-img create -f qcow2 qcow/fedora35.qcow2 50g
qemu-img info qcow/fedora35.qcow2
qemu-img check qcow/fedora35.qcow2

# define and start the vm
virsh define xml/fedora35.xml
virsh start fedora

# use vnc to install server (localhost:5900)
```

## Manage VM's

```bash
# list all vm's
virsh list --all

# stop the vm
virsh shutdown fedora

# create a new layer on top of the current layer
mv qcow/fedora35.qcow2 qcow/fedora35_00.qcow2
qemu-img create -f qcow2 -b fedora35_00.qcow2 qcow/fedora35.qcow2 -F qcow2

# restart the vm with the new layer
virsh start fedora
```

## Access Vm

```bash
# connect to vm and forward vm ports to localhost
cat <<EOT >> ./bin/fedora35
#!/bin/bash
ssh -p 2223 -L3000:localhost:3000 alex@localhost
EOT

chmod +x ./bin/fedora35

./bin/fedora35
```

## Misc

```bash
# undefine the vm
virsh undefine fedora --nvram
```
