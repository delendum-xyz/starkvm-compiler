# starkvm-compiler
Tools for  MidenVM ISA.

**It is important to note this code is unaudited and is NOT ready for production use.** 

# Build
## Building Felix
* You need Ocaml 14 and Python 3.x and a recent g++ or clang++
* install SDL2, SDL2_image, SDL2_ttf
  * you can use brew on MacOS or apt-get or something on Linux
  * Felix uses SDL2 for graphics

* cd into the repo
* run one of the scripts in buildscript depending on your OS
  * `sh buildscript/linuxsetup.sh`
  * `sh buildscript/macosxsetup.sh`
* let Felix know where it's installed: DO NOT INSTALL IT
  * `mkdir -p $HOME/.felix/config`
  * `echo "FLX_INSTALL_DIR: <location-of-repo>/build/release" > $HOME/.felix/config/felix.fpc`
* build Felix
  * make
* test `flx hello.flx`

* Setup so you can run from anywhere:
  * Add `<repo>/build/release/host/bin` to your PATH
  * Add `<repo>/build/release/host/lib/rtl` to your `DYLD_LIBRARY_PATH` or `LD_LIBRARY_PATH` in your startup script (.bash_profile on MacOS)

That's it. The build proceeds in two phases. 
* First, a Python script will build a bootstrap version of Felix BOOTFLX
* Next, BOOTFLX will rebuild the whole system.
* Finally a set of regression tests run.

Failures in the optional regression tests are OK.
However if you want graphics the SDL tests should work.
The graphics tests have a time limit so you don't need to 
press any keys (but you can).

Felix puts ALL artefacts in $HOME/.felix/cache unless you tell it otherwise.
So you can run a script in a readonly directory from anywhere.

## Example command
`flx mfront.flx examples/basic/in.txt > examples/basic/out.masm`

# Milestones (RFC)
* Control flows and basic variables - if/else, loops, functions, local variables
* Structs, pattern matching, arrays
* Basic integration with core library
* Non-deterministic inputs (hints)
* More to be added...
