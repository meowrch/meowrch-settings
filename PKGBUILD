# Maintainer: dimflix <dimflix.official@gmail.com>
# Updated with automatic SHA256 hash calculation
pkgname=meowrch-settings
pkgver=3.0.0
pkgrel=1
pkgdesc="System optimization settings for Meowrch"
arch=('any')
url="https://github.com/meowrch/meowrch-settings"
license=('MIT')
depends=('systemd' 'zram-generator' 'pipewire' 'earlyoom' 'pciutils')
optdepends=('nvidia-utils: for NVIDIA GPU optimizations'
            'amd-ucode: for AMD CPU microcode'
            'intel-ucode: for Intel CPU microcode'
            'hdparm: for HDD performance optimizations')
install=meowrch-settings.install
source=("$url/archive/refs/tags/v$pkgver.tar.gz")
sha256sums=('SKIP')  # Автоматическая замена в workflow

package() {
    cd "$srcdir/meowrch-settings-$pkgver"
    
    # Copy organized structure
    cp -r "etc" "$pkgdir/"
    cp -r "usr" "$pkgdir/"
}
