# TODO

## Warnings to fix

- [ ] **Fix `bashio::addon.port` deprecation in `nginx.sh`**
  Replace `bashio::addon.port` with `bashio::app.port`. Triggers two warnings on every
  startup since the API was renamed in the newer base image.

- [ ] **Replace `mysql` command with `mariadb` in init scripts**
  `mysql.sh` and `traccar.sh` both call the `mysql` binary which MariaDB deprecates
  in favour of `/usr/bin/mariadb`. Low urgency — still works, but will break in a
  future MariaDB release.

## Pending branch

- [ ] **Merge `bump-base-image-v21` into main**
  Bumps base image from v19 (Alpine 3.20) to v21 (Alpine 3.24). CI is passing.
  Also upgrades `openjdk21`, `mariadb-client`, `nginx`, and `nss` to Alpine 3.24 versions.
