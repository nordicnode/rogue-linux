# Maintainer: local packaging for this Rogue source tree
#
# This PKGBUILD packages the checkout it lives in: build() and package() run
# in $startdir, so run makepkg from the repository root.
#
# The scoreboard is shared through the "games" group, which Arch's filesystem
# package creates (see /usr/lib/sysusers.d/arch.conf) together with /var/games.
# The installed binary is setgid games so that any local user can post scores
# while the scoreboard stays read-only for everyone else.

pkgname=rogue
pkgver=5.4.4
pkgrel=1
pkgdesc='The original graphical adventure game that spawned an entire genre'
arch=('x86_64' 'aarch64')
url='https://github.com/Davidslv/rogue'
license=('BSD')
depends=('ncurses')
makedepends=('gcc')
backup=('var/games/roguelike/rogue.scr')
source=()

build() {
  cd "$startdir"

  ./configure \
    --prefix=/usr \
    --docdir="/usr/share/doc/$pkgname" \
    --enable-setgid=games \
    --enable-scorefile=/var/games/roguelike/rogue.scr \
    --enable-lockfile=/var/games/roguelike/rogue.lck

  make
}

package() {
  cd "$startdir"

  make DESTDIR="$pkgdir" install

  install -Dm644 rogue.desktop "$pkgdir/usr/share/applications/rogue.desktop"
  install -Dm644 rogue.png "$pkgdir/usr/share/icons/hicolor/32x32/apps/rogue.png"
  install -Dm644 LICENSE.TXT "$pkgdir/usr/share/licenses/$pkgname/LICENSE.TXT"
}
