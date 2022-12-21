# starkvm-compiler
Compiler tooling for Stark-based zkVMs, with an initial focus on Miden

**It is important to note this code is unaudited and is NOT ready for production use.** 

# Motivation
The current zkp stack doesn't have excellent open-source tooling to develop general compute applications yet, we are here to change that!
We believe that building an open-source compiler, which is optimized for StarkVM backends, will accelerate the adoption of performant zkp applications.

A few examples:
* std libraries for StarkVMs - As in regular VMs, not all functionality can be part of the core ISA the VM provides. A thriving ecosystem
of apps will need a strong std layer that is maintainable and auditable.
* Recursion - We aim for this compiler to power the first MIT licensed StarkVM that supports recursion. This will dramatically 
reduce the cost of posting zkp proofs on-chain.

# Technical vision
While the exact approach haven't been decided yet, our intent is to target a layer that will enable reuse of as much 
tooling is possible (LLVM / MLIR / WASM are being actively explored). The Design principles we would like to follow are:
* Provide quick and tangible improvement to the community
* Allow for reuse of software components - While there might be certain existing Rust programs that could not be easily 
translated into Miden ISA, we would like to maintain the benefits of allowing developers to use their existing workflows and libraries. 
Therefore, we aim to support the largest possible subset of Rust programs.
* Be open to more frontends / backends - Staying closer to machine code opens the possibility of allowing other teams
to focus on building frontends for languages they would like to use. Furthermore, we would like to design the compiler 
stack in a way that could allow us to target existing and new backends (ISA) in the future. We believe there is 
still a lot of innovation to be explored in designing a zkVMs and we are excited to support it. 

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
