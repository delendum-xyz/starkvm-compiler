test:
	# check the macro expansions are recognised as valid miden assembler
	flx mfront.flx t2.mid > t2.masm
	flx mfront.flx t2.masm
	# compile fib.mid to fib.masm
	flx mfront.flx fib.mid >fib.masm
	# verify test masms
	flx mfront.flx examples/basic/collatz.masm
	flx mfront.flx examples/basic/comparison.masm
	flx mfront.flx examples/basic/conditional.masm
	flx mfront.flx examples/basic/fibonacci.masm
	flx mfront.flx examples/basic/nprime.masm
	flx mfront.flx examples/basic/game-of-life-4x4.masm

bb2m:
	cd bb2m; cargo build
	cd bb2m; cargo run "src/main.rs"

.PHONY: bb2m
