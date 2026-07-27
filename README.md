# Building KiCad 9.0.9 on macOS 10.15.7 (Catalina), Intel, with MacPorts
This procedure has been designed incrementally on 2026-07-24 with help from Claude Fable 5.
It may not be the most efficient, but it produced a usable build.

It is posted here in the hope that it can be useful to somebody attempting a similar build
on another unsupported mac OS version. If you just want to use the app on your older mac 
running Catalina, you can find the disk image as a Release.

**Target result:** a self-contained `KiCad.app` in `~/kicad9` with schematic editor,
PCB editor, 3D viewer (with models), gerbview, calculator, footprint/symbol editors,
Plugin & Content Manager, and the ngspice simulator. Optionaly, a redistributable .dmg
installer can be produced from the build products.

**Not included:** the wxPython scripting console (deliberately disabled — see
Step 6). It is deprecated upstream in favour of the IPC API, which *is* built.

---

## 0. Assumptions and why the obvious routes don't work

| | |
|---|---|
| Machine | Intel Mac, macOS 10.15.7, ≥16 GB RAM, ≥60 GB free |
| Toolchain | Xcode 11.7 / Command Line Tools installed |
| Packages | MacPorts installed and `selfupdate`-ed; no Homebrew required |
| Time | 1 day, mostly unattended (MacPorts deps + ~2000 compile units) |

Three things rule out the easy paths:

- **Official binaries** require macOS ≥ 11.6.
- **`sudo port install kicad`** is stuck at 7.0.11.
- **kicad-mac-builder** (what the KiCad project uses) assumes Homebrew and builds
  its own patched wxWidgets. We use stock MacPorts wx 3.2 instead, which costs us
  one source patch (Step 5) and the scripting console.

Also note **Apple Clang 11.0.3 cannot build KiCad 9** — it is LLVM 9 internally and
KiCad 9 uses C++20 concepts (LLVM 10+). We use MacPorts `clang-17`. This is safe:
clang 17 ships modern libc++ *headers* but links Catalina's ABI-stable libc++
*runtime*, and mixing it with Apple-clang-built MacPorts dylibs is fine.

---

## 1. Install dependencies

```bash
sudo port selfupdate

sudo port install cmake ninja swig swig-python pkgconfig doxygen gettext bison \
  wxWidgets-3.2 glew glm cairo harfbuzz opencascade ngspice curl zstd unixODBC \
  protobuf3-cpp nng libgit2 \
  python312 \
  boost188 \
  clang-17
```

If `boost188` is not found, run `port search --name --glob 'boost1*'` and use the
highest available; note the number, you need it in Step 6.

What the less obvious ones are for:

- `opencascade` — 3D viewer and STEP import/export (largest dependency)
- `protobuf3-cpp` + `nng` — the new v9 IPC plugin API (mandatory even if unused).
  **The 3.21.x series is required**; protobuf ≥ 33 breaks KiCad 9.0.x. MacPorts
  gives 3.21.12, which is correct. Ignore `protobuf-cpp @2.6.1` (the legacy line).
- `libgit2` — v9 project git integration
- `ngspice` — simulator engine (MacPorts splits the dylib into `ngspice-lib`,
  pulled in automatically)
- `harfbuzz` / `cairo` / `glew` / `glm` — text shaping, 2D fallback rendering, GL

`rust`, `cargo` and `mbedtls3` get built from source as `nng` dependencies —
that alone can take a couple of hours.

**wxPython is not needed** with the console disabled. If you ever want it, it fails
to install on Catalina because its build script calls `readlink -f` (a GNU option
BSD readlink lacks); the workaround is `sudo port install coreutils`, then
temporarily `sudo ln -s /opt/local/bin/greadlink /opt/local/bin/readlink`, install,
and remove the shim.

---

## 2. Record the machine-specific paths

MacPorts layouts vary between versions. Collect these now — Step 6 needs them:

```bash
# wx-config: note the directory is called "3.1" even though it holds wx 3.2.x
port contents wxWidgets-3.2 | grep bin/wx-config

# boost prefix (versioned, NOT /opt/local/include)
ls -d /opt/local/libexec/boost/*

# ngspice: confirm the real file and the version digit
ls -la /opt/local/lib/ | grep -i ngspice

# opencascade headers (7.9 ships under the standard prefix)
port contents opencascade | grep Standard_Version.hxx
```

Expected on a 2026-vintage MacPorts tree:

```
/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.1/bin/wx-config
/opt/local/libexec/boost/1.88
/opt/local/lib/libngspice.0.dylib   (+ libngspice.dylib symlink)
/opt/local/include/opencascade/Standard_Version.hxx
```

Substitute anything that differs into the `cmake` invocation below.

---

## 3. Get the source

```bash
mkdir -p ~/projects/kicad_build/src && cd ~/projects/kicad_build/src
git clone --branch 9.0.9 --depth 1 https://gitlab.com/kicad/code/kicad.git
cd kicad
```

---

## 4. Patch the macOS install scripts (three patches)

All three are Catalina/MacPorts adaptations of code that only ever ran under
kicad-mac-builder's controlled Homebrew tree. Run this whole block from the source
root; it backs up each original first.

```bash
cd ~/projects/kicad_build/src/kicad

cp cmake/InstallSteps/InstallMacOS.cmake cmake/InstallSteps/InstallMacOS.cmake.orig
cp cmake/InstallSteps/RefixupMacOS.cmake cmake/InstallSteps/RefixupMacOS.cmake.orig
cp eeschema/CMakeLists.txt eeschema/CMakeLists.txt.orig

python3 - << 'PYEOF'
import sys

# --- Patch 1: dependency scan ------------------------------------------------
# a) Conflicting paths (e.g. /opt/local/lib/libbz2 vs /usr/lib/libbz2) are a hard
#    error upstream; prefer the MacPorts copy.
# b) Catalina still has system dylibs on disk (Big Sur+ hides them in the dyld
#    shared cache), so the scanner bundles libSystem/libobjc/etc. and the refixup
#    then rewrites them to @rpath -- which makes dyld abort at launch with
#    "initializer in image ... that does not link with libSystem.dylib".
p = "cmake/InstallSteps/InstallMacOS.cmake"
s = open(p).read()
old = """    file( GET_RUNTIME_DEPENDENCIES
        LIBRARIES ${libs}
        EXECUTABLES ${exe}
        RESOLVED_DEPENDENCIES_VAR _r_deps
        UNRESOLVED_DEPENDENCIES_VAR _u_deps
        POST_EXCLUDE_FILES Python
    )"""
new = """    file( GET_RUNTIME_DEPENDENCIES
        LIBRARIES ${libs}
        EXECUTABLES ${exe}
        RESOLVED_DEPENDENCIES_VAR _r_deps
        UNRESOLVED_DEPENDENCIES_VAR _u_deps
        CONFLICTING_DEPENDENCIES_PREFIX _c_deps
        POST_EXCLUDE_FILES Python
        POST_EXCLUDE_REGEXES "^/usr/lib/" "^/System/"
    )

    # Prefer the MacPorts copy when a dependency resolves to several paths.
    foreach( _fname ${_c_deps_FILENAMES} )
        set( _picked "" )
        foreach( _cand ${_c_deps_${_fname}} )
            if( _cand MATCHES "^/opt/local/" )
                set( _picked "${_cand}" )
            endif()
        endforeach()
        if( _picked STREQUAL "" )
            list( GET _c_deps_${_fname} 0 _picked )
        endif()
        message( STATUS "Conflict for ${_fname}: using ${_picked}" )
        list( APPEND _r_deps "${_picked}" )
    endforeach()"""
if old not in s:
    sys.exit("Patch 1 anchor not found -- source layout differs, patch by hand")
open(p, "w").write(s.replace(old, new))

# --- Patch 2: refixup --------------------------------------------------------
# a) Files copied from /opt/local are not user-writable; install_name_tool needs
#    write access.
# b) Never rewrite through a symlink -- it writes to the root-owned original in
#    /opt/local and fails.
p = "cmake/InstallSteps/RefixupMacOS.cmake"
s = open(p).read()
old = """    foreach( item ${items} )
        message( "Refixing prereqs for '${item}'" )
        refix_prereqs( ${item} )
    endforeach( )"""
new = """    execute_process( COMMAND chmod -R u+w "${target}" )

    foreach( item ${items} )
        if( IS_SYMLINK "${item}" )
            message( "Skipping symlink '${item}'" )
            continue()
        endif()
        message( "Refixing prereqs for '${item}'" )
        refix_prereqs( ${item} )
    endforeach( )"""
if old not in s:
    sys.exit("Patch 2 anchor not found -- source layout differs, patch by hand")
open(p, "w").write(s.replace(old, new))

# --- Patch 3: ngspice install ------------------------------------------------
# Upstream installs *every* dylib under the directory containing libngspice.
# Under kicad-mac-builder that directory holds only libngspice; here it is
# /opt/local/lib, i.e. the entire MacPorts library tree (incl. root-owned
# symlinks that break install_name_tool). Install just the one resolved file.
p = "eeschema/CMakeLists.txt"
s = open(p).read()
old = """    install( DIRECTORY "${LIBNGSPICE_PATH}/"
            DESTINATION "${OSX_BUNDLE_INSTALL_PLUGIN_DIR}/sim"
            FILES_MATCHING PATTERN "*.dylib")"""
new = """    get_filename_component( NGSPICE_DLL_REALPATH "${NGSPICE_DLL}" REALPATH )
    install( FILES "${NGSPICE_DLL_REALPATH}"
            PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
                        GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
            DESTINATION "${OSX_BUNDLE_INSTALL_PLUGIN_DIR}/sim" )"""
if old not in s:
    sys.exit("Patch 3 anchor not found -- source layout differs, patch by hand")
open(p, "w").write(s.replace(old, new))

print("all three patches applied")
PYEOF
```

The second ngspice rule in `eeschema/CMakeLists.txt` (`install( DIRECTORY
"${LIBNGSPICE_PATH}/ngspice" ...)`) is correct and must stay — those are the
simulator codemodels.

---

## 5. Patch the wxWidgets 3.3 API use

`kicad/project_tree.cpp` calls `wxTreeCtrl::SetStateImages()`, which exists only in
wx 3.3 / KiCad's wxWidgets fork. Replace it with the wx 3.2 equivalent:

```bash
cd ~/projects/kicad_build/src/kicad
cp kicad/project_tree.cpp kicad/project_tree.cpp.orig

python3 - << 'PYEOF'
import sys
p = "kicad/project_tree.cpp"
s = open(p).read()
old = "    SetStateImages( stateImages );"
new = """    {
        // wx 3.2 has no SetStateImages(); build a wxImageList from the bundles
        wxSize iconSize = KiBitmapBundle( BITMAPS::git_good_check ).GetDefaultSize();
        wxImageList* stateImgList = new wxImageList( iconSize.x, iconSize.y, true,
                                                     (int) stateImages.size() );

        for( const wxBitmapBundle& bnd : stateImages )
        {
            wxBitmap bmp = bnd.GetBitmap( iconSize );

            if( !bmp.IsOk() )
                bmp = wxBitmap( iconSize.x, iconSize.y );

            stateImgList->Add( bmp );
        }

        AssignStateImageList( stateImgList );
    }"""
if s.count(old) != 1:
    sys.exit("anchor not unique/found -- patch by hand")
open(p, "w").write(s.replace(old, new))
print("project_tree.cpp patched")
PYEOF
```

`AssignStateImageList` transfers ownership, so nothing leaks. Cosmetic difference:
the "untracked" git state gets a blank icon instead of no overlay.

---

## 6. Configure

Adjust `boost188` → your version and the wx / ngspice paths from Step 2 if they
differ.

```bash
cd ~/projects/kicad_build/src/kicad
mkdir -p build/release && cd build/release

cmake -G Ninja \
  -DCMAKE_C_COMPILER=/opt/local/bin/clang-mp-17 \
  -DCMAKE_CXX_COMPILER=/opt/local/bin/clang++-mp-17 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$HOME/kicad9 \
  -DCMAKE_PREFIX_PATH=/opt/local \
  -DCMAKE_SHARED_LINKER_FLAGS="-L/opt/local/lib -lmbedtls -lmbedx509 -lmbedcrypto" \
  -DCMAKE_EXE_LINKER_FLAGS="-L/opt/local/lib -lmbedtls -lmbedx509 -lmbedcrypto" \
  -DBoost_ROOT=/opt/local/libexec/boost/1.88 \
  -DwxWidgets_CONFIG_EXECUTABLE=/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.1/bin/wx-config \
  -DPYTHON_EXECUTABLE=/opt/local/bin/python3.12 \
  -DPYTHON_LIBRARY=/opt/local/Library/Frameworks/Python.framework/Versions/3.12/lib/libpython3.12.dylib \
  -DPYTHON_INCLUDE_DIR=/opt/local/Library/Frameworks/Python.framework/Versions/3.12/include/python3.12 \
  -DPYTHON_FRAMEWORK=/opt/local/Library/Frameworks/Python.framework \
  -DOCC_INCLUDE_DIR=/opt/local/include/opencascade \
  -DOCC_LIBRARY_DIR=/opt/local/lib \
  -DNGSPICE_INCLUDE_DIR=/opt/local/include \
  -DNGSPICE_LIBRARY=/opt/local/lib/libngspice.dylib \
  -DNGSPICE_DLL=/opt/local/lib/libngspice.0.dylib \
  -DKICAD_SCRIPTING_WXPYTHON=OFF \
  -DKICAD_USE_SENTRY=OFF \
  -DKICAD_BUILD_I18N=ON \
  ../..
```

Why each non-obvious flag is there:

| Flag | Reason |
|---|---|
| `clang-mp-17` | Apple Clang 11 lacks C++20 concepts |
| `*_LINKER_FLAGS` mbedtls | MacPorts `nng` is a **static** lib built with TLS; KiCad's link line omits mbedtls, giving ~40 undefined `mbedtls_*` symbols |
| `Boost_ROOT` | MacPorts keeps versioned boost under `libexec`, invisible to `CMAKE_PREFIX_PATH` |
| `wxWidgets_CONFIG_EXECUTABLE` | wx is a framework outside PATH; the dir is misleadingly named `3.1` |
| `PYTHON_*` | KiCad's old `FindPythonLibs` otherwise picks system Python 2.7 |
| `PYTHON_FRAMEWORK` | empty → `set_target_properties called with incorrect number of arguments` at `kicad/CMakeLists.txt:221` |
| `OCC_*` / `NGSPICE_*` | seed the finders; both search paths that don't match MacPorts |
| `KICAD_SCRIPTING_WXPYTHON=OFF` | see below |
| `KICAD_USE_SENTRY=OFF` | no crash telemetry |
| `KICAD_BUILD_I18N=ON` | build translations (French UI etc.) |

**On the console being off:** with it ON, the app builds and runs, but wxPython
loads its own private wx 3.2.8 into a process already running wx 3.2.10. Result:
duplicate class registries — close buttons stop working in every window, and the
app segfaults on quit. Two wx runtimes in one process is not survivable, and the
console is deprecated upstream anyway. `import pcbnew` from external Python,
headless action plugins, `kicad-cli`, and the IPC API all still work.

Expect the configure to end with `Configuring done` / `Generating done`. A warning
about `FindBoost`/CMP0167 and "Doxygen missing components: dot" are both harmless.

---

## 7. Build and install

```bash
ninja -j4 2>&1 | tee build.log
ninja install 2>&1 | tee install.log
```

`-j4` caps memory use — linking pcbnew peaks around 2–3 GB per job. ~2000 targets;
budget several hours.

`ninja install` ends with two `macOS signing failed ... bundle format unrecognized`
warnings. **These are expected** and are fixed by the first command in Step 8
(codesign rejects the Python framework until it has a `Versions/Current` symlink).
Catalina runs locally built unsigned apps regardless — Gatekeeper only gates
quarantined downloads.

---

## 8. Post-install fixes

**These are wiped by every `ninja install` — re-run the whole block after any
reinstall.**

```bash
APP=~/kicad9/KiCad.app

# 1. Python framework needs Versions/Current (PYTHONHOME points at it; codesign
#    also refuses the bundle without it)
ln -s 3.12 "$APP/Contents/Frameworks/Python.framework/Versions/Current"

# 2. KiCad dlopens the unversioned ngspice name
ln -s libngspice.0.dylib "$APP/Contents/PlugIns/sim/libngspice.dylib"

# 3. Optional: silence a harmless simulator console error. ivlng.vpi is the
#    Icarus Verilog co-simulation bridge and needs vvp symbols it can't get.
rm -f "$APP/Contents/PlugIns/sim/ngspice/ivlng.vpi"

# 4. Optional: satisfy codesign now that the framework is well-formed
codesign --force --deep --sign - "$APP"
```

Optional cleanup: the dependency scanner also drags in MacPorts Python **3.14**
(pulled in as a build dependency of other ports). `rm -rf
"$APP/Contents/Frameworks/Python.framework/Versions/3.14"` reclaims the space;
`Current → 3.12` is what matters.

---

## 9. Libraries

**Pin to the `9.0.9` tag.** `master` has already moved to the v10 file format
(dated 20260206) and KiCad 9 refuses to read it — the failure mode is a wall of
"created with a more recent version" errors in the footprint editor and
"`.kicad_symdir` not found" in the symbol editor.

```bash
mkdir -p ~/kicad9-libs && cd ~/kicad9-libs

for r in kicad-symbols kicad-footprints kicad-templates kicad-packages3D; do
    git clone --depth 1 --branch 9.0.9 \
        https://gitlab.com/kicad/libraries/$r.git
done
```

`kicad-packages3D` is ~10 GB — omit it if you don't need 3D component models
(the board itself still renders). If the tag is rejected, list what exists with
`git ls-remote --tags <url> | grep "9\.0\."` and take the highest 9.0.x.

Link them into the bundle. **Also wiped by `ninja install`:**

```bash
SS=~/kicad9/KiCad.app/Contents/SharedSupport
ln -s ~/kicad9-libs/kicad-symbols    "$SS/symbols"
ln -s ~/kicad9-libs/kicad-footprints "$SS/footprints"
ln -s ~/kicad9-libs/kicad-templates  "$SS/templates"
ln -s ~/kicad9-libs/kicad-packages3D "$SS/3dmodels"
```

Symlinks rather than copies, so a reinstall never destroys the ~13 GB of data —
only the links, which the commands above recreate.

---

## 10. First launch and verification

```bash
~/kicad9/KiCad.app/Contents/MacOS/kicad
```

Running from Terminal the first few times is worth it — dyld and Python errors go
to stderr and are invisible when launching from Finder.

KiCad 9 uses `~/Library/Preferences/kicad/9.0`, entirely separate from KiCad 8, so
an existing 8.x install is unaffected. On first run it offers to import 8.x
settings; either answer is fine.

Checklist:

- [ ] Main launcher window opens
- [ ] Open an existing project
- [ ] Schematic editor: opens, panels populate, selection works
- [ ] PCB editor: opens, pan/zoom work
- [ ] 3D viewer: board renders, components appear (needs `3dmodels`)
- [ ] Symbol editor: library tree populates
- [ ] Footprint editor: library tree populates
- [ ] Gerber viewer, PCB calculator
- [ ] Plugin & Content Manager: lists packages, can install one
- [ ] Tools → Simulator in eeschema: opens, runs an op point on a divider

**If the symbol editor errors** about missing `.kicad_symdir` files, the global
table was copied from an unpinned checkout before Step 9. Replace it:

```bash
cp ~/kicad9-libs/kicad-symbols/sym-lib-table ~/Library/Preferences/kicad/9.0/sym-lib-table
```

(and the same for `fp-lib-table` from `kicad-footprints` if footprints misbehave).

---

## 11. Updating to a later 9.0.x

The patched source tree is reusable:

```bash
cd ~/projects/kicad_build/src/kicad
git stash                       # park the four patches
git fetch --depth 1 origin tag 9.0.10
git checkout 9.0.10
git stash pop                   # reapply; resolve conflicts if upstream moved the code
cd build/release
ninja -j4 && ninja install
```

Then redo **Step 8** and the symlinks in **Step 9** — they are wiped every time.
Bump the library repos to the matching tag as well.

---

## 12. Building a distributable DMG (optional)

The installed bundle is already self-contained and rpath-correct — the only things
tying it to this machine are the library symlinks. Three caveats before you share
it:

- **Gatekeeper.** The ad-hoc signature (`--sign -`) is not a Developer ID.
  A downloaded DMG is quarantined, and recipients get *"KiCad.app is damaged and
  can't be opened"*. They clear it with right-click → Open, or
  `xattr -dr com.apple.quarantine /Applications/KiCad.app`. Signing it properly
  needs a Developer ID ($99/yr) **and** notarization — and notarization is a hard
  blocker on this machine: Apple retired the `altool` path in late 2023, and
  `notarytool` requires Xcode 13+, which does not run on Catalina. You would need a
  newer Mac to notarize, even though the binary is built here.
- **GPLv3.** Distributing binaries obliges you to offer corresponding source,
  including the patches from Steps 4–5. Publishing the patched tree, or this
  document plus a patch set, satisfies that.
- **Compatibility of the result:** x86_64 only (runs under Rosetta 2 on Apple
  Silicon), built against the 10.15 SDK, so macOS 10.15 and later — exactly the
  audience that cannot use the official builds.

```bash
rm -rf /tmp/kicad-dist && mkdir -p /tmp/kicad-dist
cp -R ~/kicad9/KiCad.app /tmp/kicad-dist/

# 1. Replace the library symlinks with real content (drop git history)
SS=/tmp/kicad-dist/KiCad.app/Contents/SharedSupport
rm "$SS/symbols" "$SS/footprints" "$SS/templates" "$SS/3dmodels"
cp -R ~/kicad9-libs/kicad-symbols    "$SS/symbols"
cp -R ~/kicad9-libs/kicad-footprints "$SS/footprints"
cp -R ~/kicad9-libs/kicad-templates  "$SS/templates"
cp -R ~/kicad9-libs/kicad-packages3D "$SS/3dmodels"
find "$SS" -maxdepth 2 -name .git -exec rm -rf {} +

# 2. Remove absolute symlinks -- codesign rejects any symlink in a bundle that
#    resolves outside it. The bundled Python framework carries a wx-config
#    pointing into /opt/local (a wxPython build helper, unused at runtime).
PF=/tmp/kicad-dist/KiCad.app/Contents/Frameworks/Python.framework
find "$PF" -type l -lname '/*' -delete
find "$PF" -type l ! -exec test -e {} \; -delete     # any dangling links too

# 3. Drag-to-install target
ln -s /Applications /tmp/kicad-dist/Applications

# 4. Sign inside-out. `--deep` validates nested-framework seals rather than
#    reliably replacing them, so a stale seal in the bundled Python framework
#    fails verification with "a sealed resource is missing or invalid" --
#    especially after step 2 deleted a file the old seal still lists.
find /tmp/kicad-dist/KiCad.app -name _CodeSignature -exec rm -rf {} + 2>/dev/null
rm -rf "$PF/Versions/3.14"          # unused; multi-version frameworks sign poorly

codesign --force --sign - "$PF/Versions/3.12"
codesign --force --sign - "$PF"
codesign --force --sign - /tmp/kicad-dist/KiCad.app

codesign --verify --strict /tmp/kicad-dist/KiCad.app && echo "signature ok"

# 5. Compressed read-only image
hdiutil create -volname "KiCad 9.0.9" -srcfolder /tmp/kicad-dist \
  -ov -format UDZO ~/KiCad-9.0.9-Catalina-x86_64.dmg
```

Size: roughly 3–5 GB with 3D models, a few hundred MB without. Consider building
both variants — omit the `3dmodels` copy for the small one.

**Verify before sharing.** Confirm nothing still resolves to MacPorts:

```bash
otool -L /tmp/kicad-dist/KiCad.app/Contents/MacOS/kicad | grep /opt/local
otool -L /tmp/kicad-dist/KiCad.app/Contents/PlugIns/_pcbnew.kiface | grep /opt/local
```

Both should print nothing. The real test is mounting the DMG on a machine that has
never had MacPorts installed — a missed dependency only reveals itself there.

---

## Appendix: failure signatures and their causes

Useful if something diverges on the second machine.

| Symptom | Cause / fix |
|---|---|
| `error: expected unqualified-id ... concept FloatingPoint` | Building with Apple Clang. Use `clang-mp-17` (Step 6). |
| ~40 undefined `_mbedtls_*` symbols linking `libkicommon` | Missing mbedtls linker flags (Step 6). |
| `Could NOT find Boost (missing: Boost_INCLUDE_DIR)` | Missing `Boost_ROOT` (Step 2/6). |
| `NGSPICE library missing` at configure | Missing `NGSPICE_LIBRARY` / `NGSPICE_DLL`. |
| `Standard_Version.hxx cannot be read` | Wrong `OCC_INCLUDE_DIR`; use `port contents opencascade`. |
| `set_target_properties called with incorrect number of arguments` | Missing `PYTHON_FRAMEWORK`. |
| `Could NOT find wxWidgets (missing: wxWidgets_LIBRARIES)` | Wrong `wx-config` path — remember the `3.1` directory name. |
| `error: use of undeclared identifier 'SetStateImages'` | Step 5 patch not applied. |
| `Multiple conflicting paths found for libbz2` | Step 4 patch 1 not applied. |
| `install_name_tool: ... Permission denied` during install | Step 4 patch 2 (or patch 3, if the path is under `PlugIns/sim`). |
| `dyld: initializer in image (.../libSystem.B.dylib) that does not link with libSystem.dylib` | System dylibs got bundled — Step 4 patch 1's `POST_EXCLUDE_REGEXES`. |
| `ModuleNotFoundError: No module named 'encodings'` | Running the app from the *build tree*. Only the installed bundle has a populated Python framework. |
| `Library not loaded: @rpath/libwx_..._core-3.2.0.4.1.dylib` | wxPython console enabled; disable it (Step 6). Do **not** symlink the 3.2.8 dylibs into Frameworks — that loads two wx runtimes and breaks every window's close button. |
| `Failed to load shared library 'libngspice.dylib'` | Step 8 symlink 2 missing. |
| Footprint editor: "created with a more recent version ... 20260206" | Libraries on `master` instead of tag 9.0.9 (Step 9). |
| `codesign: invalid destination for symbolic link in bundle` | An absolute or dangling symlink inside the bundle — see Step 12 item 2. Locate them with `find <bundle> -type l -lname '/*'` and `find <bundle> -type l ! -exec test -e {} \; -print`. |
| `codesign: a sealed resource is missing or invalid` | Stale `_CodeSignature` in a nested framework. Delete all seals and sign inside-out — Step 12 item 4. |
| Recipient sees "KiCad.app is damaged and can't be opened" | Quarantine on an ad-hoc-signed download, not corruption. `xattr -dr com.apple.quarantine /Applications/KiCad.app`, or right-click → Open. |

