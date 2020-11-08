# OnionShare Release Process

Unless you're a core OnionShare developer making a release, you'll probably never need to follow it.

## Changelog, version, docs, and signed git tag

Before making a release, you must update the version in these places:

- [ ] `cli/pyproject.toml`
- [ ] `cli/setup.py`
- [ ] `cli/onionshare_cli/resources/version.txt`
- [ ] `desktop/pyproject.toml` (under `version` and the `./onionshare_cli-$VERSION-py3-none-any.whl` dependency)
- [ ] `desktop/src/setup.py`
- [ ] `docs/source/conf.py`

Make sure snapcraft packaging works. In `snap/snapcraft.yaml`:

- [ ] The `tor`, `libevent`, and `obfs4` parts should be updated if necessary
- [ ] All python packages should be updated to match `cli/pyproject.toml` and `desktop/pyproject.toml`
- [ ] Test the snap package, ensure it works

Make sure Flatpak packaging works. In `flatpak/org.onionshare.OnionShare.yaml`:

- [ ] Update `tor`, `libevent`, and `obfs4` dependencies, if necessary
- [ ] Built the latest python dependencies using [this tool](https://github.com/flatpak/flatpak-builder-tools/blob/master/pip/flatpak-pip-generator) (see below)
- [ ] Test the Flatpak package, ensure it works

```
# you may need to install toml
pip3 install --user toml

# clone flatpak-build-tools
git clone https://github.com/flatpak/flatpak-builder-tools.git
cd flatpak-builder-tools/pip

# get onionshare-cli dependencies
./flatpak-pip-generator $(python3 -c 'import toml; print("\n".join([x for x in toml.loads(open("../../onionshare/cli/pyproject.toml").read())["tool"]["poetry"]["dependencies"]]))' |grep -v "^python$" |tr "\n" " ")
mv python3-modules.json onionshare-cli.json

# get onionshare dependencies
./flatpak-pip-generator $(python3 -c 'import toml; print("\n".join(toml.loads(open("../../onionshare/desktop/pyproject.toml").read())["tool"]["briefcase"]["app"]["onionshare"]["requires"]))' |grep -v "./onionshare_cli" |grep -v -i "pyside2" |tr "\n" " ")
mv python3-modules.json onionshare.json

# use something like https://www.json2yaml.com/ to convert to yaml and update the manifest
# add all of the modules in both onionshare-cli and onionshare to the submodules of "onionshare"
# - onionshare-cli.json
# - onionshare.json
```

Update the documentation:

- [ ] Update all of the documentation in `docs` to cover new features, including taking new screenshots if necessary

You also must edit these files:

- [ ] `desktop/install/org.onionshare.OnionShare.appdata.xml` should have the correct release date, and links to correct screenshots
- [ ] `CHANGELOG.md` should be updated to include a list of all major changes since the last release

Finally:

- [ ] There must be a PGP-signed git tag for the version, e.g. for OnionShare 2.1, the tag must be `v2.1`

The first step for the Linux, macOS, and Windows releases is the same.

Verify the release git tag:

```sh
git fetch
git tag -v v$VERSION
```

If the tag verifies successfully, check it out:

```sh
git checkout v$VERSION
```

## PyPi release

The CLI version of OnionShare gets published on PyPi. To make a release:

```sh
cd cli
poetry install
poetry publish --build
```

## Linux Flatpak release

You must have `flatpak` and `flatpak-builder` installed, with flathub remote added (`flatpak remote-add --if-not-exists --user flathub https://flathub.org/repo/flathub.flatpakrepo`).

Build and test the Flatpak package before publishing:

```sh
flatpak-builder build --force-clean --install-deps-from=flathub --install --user flatpak/org.onionshare.OnionShare.yaml
flatpak run org.onionshare.OnionShare
```

## Linux Snapcraft release

You must have `snap` and `snapcraft` (`snap install snapcraft --classic`) installed.

Build and test the snap before publishing:

```sh
snapcraft
snap install --devmode ./onionshare_*.snap
```

Run the OnionShare snap:

```sh
/snap/bin/onionshare     # GUI version
/snap/bin/onionshare.cli # CLI version
```

## Linux AppImage release

_Note: AppImage packages are currently broken due to [this briefcase bug](https://github.com/beeware/briefcase/issues/504). Until it's fixed, OnionShare for Linux will only be available in Flatpak and Snapcraft._

Set up the development environment described in `README.md`.

Make sure your virtual environment is active:

```sh
. venv/bin/activate
```

Run the AppImage build script:

```sh
./package/linux/build-appimage.py
```

### Windows

Set up the development environment described in `README.md`. And install the [Windows 10 SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk) and add `C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x86` to your path.

Make sure your virtual environment is active:

```
venv\Scripts\activate.bat
```

Run the Windows build script:

```
python package\windows\build.py
```

This will create `windows/OnionShare-$VERSION.msi`, signed.

### macOS

Set up the development environment described in `README.md`. And install `create-dmg`:

```sh
brew install create-dmg
```

Make sure your virtual environment is active:

```sh
. venv/bin/activate
```

Run the macOS build script:

```sh
./package/macos/build.py --with-codesign
```

Now, notarize the release. You must have an app-specific Apple ID password saved in the login keychain called `onionshare-notarize`.

- Notarize it: `xcrun altool --notarize-app --primary-bundle-id "com.micahflee.onionshare" -u "micah@micahflee.com" -p "@keychain:onionshare-notarize" --file macOS/OnionShare.dmg`
- Wait for it to get approved, check status with: `xcrun altool --notarization-history 0 -u "micah@micahflee.com" -p "@keychain:onionshare-notarize"`
- After it's approved, staple the ticket: `xcrun stapler staple macOS/OnionShare.dmg`

This will create `macOS/OnionShare.dmg`, signed and notarized.

### Source package

To make a source package, run `./build-source.sh $TAG`, where `$TAG` is the the name of the signed git tag, e.g. `v2.1`.

This will create `dist/onionshare-$VERSION.tar.gz`.

### Publishing the release

To publish the release:

- Create a new release on GitHub, put the changelog in the description of the release, and upload all six files (the macOS installer, the Windows installer, the source package, and their signatures)
- Upload the six release files to https://onionshare.org/dist/$VERSION/
- Copy the six release files into the OnionShare team Keybase filesystem
- Update the [onionshare-website](https://github.com/micahflee/onionshare-website) repo:
  - Edit `latest-version.txt` to match the latest version
  - Update the version number and download links
  - Deploy to https://onionshare.org/
- Email the [onionshare-dev](https://lists.riseup.net/www/subscribe/onionshare-dev) mailing list announcing the release
- Make a PR to [homebrew-cask](https://github.com/homebrew/homebrew-cask) to update the macOS version