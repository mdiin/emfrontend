# em_frontend

A visualisation of the Event Model produced by the emcli.

## Prerequisites

### Fedora

```
sudo dnf install clang cmake ninja-build pkgconf gtk3-devel xz-devel libstdc++-devel
```

### Ubuntu/Debian

```
sudo apt-get install clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev libstdc++-12-dev
```

## Running

```
bb app
```

If the build fails with a linker error mentioning a stale compiler path (e.g. from a Guix profile), delete the cached CMake build and retry:

```
rm -rf build/linux && bb app
```
