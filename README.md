
# B'SST: Bitcoin-like Script Symbolic Tracer

Copyright (c) 2023 Dmitry Petukhov (https://github.com/dgpv), dp@bsst.dev

Symbolically executes the opcodes, tracks constraints that opcodes impose on
values they operate on, and shows the report with conditions that the script
enforces, possible failures, possible values for data, etc.

Supports Bitcoin script and Elements script.

**IMPORTANT**: This program can help uncover problems with the scripts it analyses,
BUT it cannot guarantee that there are no problems, inconsistenses, bugs,
vulnerabilities, et cetera in the analyzed script. This program itself or the
underlying libraries can also contain bugs or other inconsistencies that could
preclude detection of the problems with the analyzed script. For some type
of problems, the analysis algorithm just cannot detect them at all.

This program should be used as an additional layer of defence in the struggle
to detect defects and unexpected behavior in the scripts, much like other
things like testing or code audit are used for this purpose, simply reducing
the probability of defects being undetected. It can also be used as a tool to
better understand the behavior of analyzed scripts.

[Elements script interpreter](https://github.com/ElementsProject/elements/blob/master/src/script/interpreter.cpp),
which is an extension of Bitcoin script interpreter, was used as reference.
Efforts have been made to match the behavior of reference interpreter closely, but it
will obviously never be a 100%, consensus-correct, match.

## License

Free for non-commercial use. Licensed under Prosperity Public License 3.0.0.
This license allows for thirty-day trial period for commercial purposes. There are also
exemptions for educational institutions, public research organizations, etc.
Please read LICENSE.md for the full text of the license.
For inquiries, please write to license@bsst.dev

Contains portions of the code that were originally released under MIT software
license. These are code of the CSHA256 class (derived from MIT-licensed code,
that was authored by various Bitcoin Core developers) and ripemd160 function
(MIT-licensed code, authored by Pieter Wuille). Please refer to the source code
of bsst python module for more information on these.

## Thoroughness vs speed of analysis

It is highly recommended to have [Z3 theorem prover](https://github.com/Z3Prover/z3) python
package installed (see Optional Dependencies below), and run `bsst-cli` with `--z3-enabled=true`
setting. Without this, a lot of possible issues decectable with help of Z3 will not be detected.

Still, running with `--z3-enabled=false` (the default setting) can be useful in some contexts,
where speed of checking is more important than thoroughness.

## Dependencies

Python 3.10 or later is required

## Optional Dependencies

For B'SST to be able to use Z3 theorem prover, "z3-solver" python package
(https://pypi.org/project/z3-solver/) is needed.

For the analyzer to check validity of statically-known public keys,
secp256k1 C library (https://github.com/bitcoin-core/secp256k1/) is needed.
B'SST will attempt to find it with `ctypes.util.find_library` and thenload it
using `ctypes.cdll.LoadLibrary`.

## Syntax

Syntax parser is non-strict:

* case-insensitive, `OP_ADD` is the same as `ADD`,
* The string `'data'` can be represented as: `'data'`, `x('64617461')`, or `0x64617461`.
* Strings in quotes cannot contain whitespace, and can use only single quotes (`'`)
* LE64 value 555 can be represented as `x('2b02000000000000')`, `0x2b02000000000000`, or `le64(555)`.
* ScriptNum values are represented with normal base10 integers.

Identifiers starting with `$` are recognized as data placeholders: `$some_var`

`//` marks the start of the comment, that spans to end of line.

A special format of comment is recoginzed:

`OP_ADD // =>add_result` will mark the value on the stack after `OP_ADD` with
the identifier `add_result`, and this identifier will be used in the report.
There should be no whitespace between `=>` and the identifier. There may be
whitespace between `//` and `=>`, but nothing other than whitespace. In the case when
different code paths result in different values assigned to the same identifier,
an apostrophe <<'>> will be appended to the identifier with different value.

## Reports

The reports show:

* Valid paths: execution paths (branches with branch condition values) that can result in successful script completion
* Enforced constraints: per-path list of constraints that must be satisfied for the successful script completion.
  If constraint is identical in all sub-paths, it will be moved up one level of path hierarchy.

    - If constraint condition is always true, it will be marked with `<*>`
    - If constraint condition is always true in particular execution path, it will be marked with `{*}`
      (unless `--mark-path-local-always-true-enforcements` is set to `false`)
    - If constraint condition is shown as `BOOL(<condition>)`, that means the condition is passed to `CastToBool()`:
      empty data, arbitrary-length block of zero bytes, as well as arbitrary-length 'negative zero' (zero-bytes block
      ended with byte 0x80) are seen as `false`, while any other value is seen as `true`. If it is obvious that the
      condition is already boolean (like the result of `LESSTHAN`, for example), the condition is not shown wrapped in `BOOL`.

* Model values: possible values for various variables such as witnesses, script result, or transaction fields
  (in Elements mode, where there are transaction introspection opcodes)

    - If there could be more values, the name and the value will be separated with `:`
    - If it is found that only one value is possible, the name and the value will be separated with `=`
    - If the value is totally unknown, it will be `?`

* Warnings: possible issues dected while symbolically executing the script, that do not lead to script failure,
  but it is probably better to examine them and understand if they are problematic or not
* Failures: Information on detected script failures, with opcode that might have caused the vailure, and stack contents
  at the this opcode was executed
* Progress log: if `--log-progress` is `true`, the details of checking of 'always true' conditions for enforced constraints,
  and 'only one possible value' for model values will be reported - namely, the 'probable cause' of why the condition is
  deemed always true or only one possible value is found, will be printed.

Some opcodes are abbreviated: `CHECKLOCKTIMEVERIFY` -> `CLTV`, `CHECKSEQUENCEVERIFY` -> `CSV`.
For Elements mode, `'a' 'b' CAT` will be reported as `'a'.'b'`

NOTE: Model values will satisfy restrictions imposed on them by opcodes, as modelled by B'SST.
Since modelling is not perfect, sometimes incomplete, these values can still be invalid if used for execution
of the script on "real" interpreter. For example, for ECDSA pubkeys only constraints on size and first byte are modelled,
and model value can show arbitrary data for the rest of pubkey.

NOTE: With Z3 enabled, failure report may give several possible causes for the failure. It does not mean that
all of these conditions are a definite cause of this particular failure. Some of them may be false positives,
but this is the nature of Z3 - it gives 'possible causes' for constraint violation, and for the report to give
more concrete place of failure, much more constraints would need to be placed by the code, which can significantly
slow down the solving times, and there's still no guarantees that you would always get just one definitive cause of
constraint violation

### Example report

For this rather complex Elements script:

https://github.com/fuji-money/tapscripts/blob/with-annotations-for-bsst/beta/mint-mint.tapscript

B'SST gives this report:

https://gist.github.com/dgpv/b57ecf4d9e3d0bfdcc2eb9147c9b9abf

## Custom opcodes

Please look at `examples/example_op_plugin.py`. With `--op-plugins=examples/example_op_plugin.py`,
B'SST will recognize `EXAMPLE` custom opcode.

## Assumptions and omissions

B'SST makes certain assumptions, and omits modelling some of the aspects of the script.

Below is (probably incomplete) list of these assumptions and omissions:

`CODESEPARATOR` is not modelled, and treated as NOP

`OP_SUCCESS` and "upgradeable NOPs" are not modelled

For non-tapscript modes, Script size limit is not modelled, but the 'number of opcodes' limit is modelled

The following script flags are assumed to be always set (consensus-enforced at the time of writing, mid-2023):

`SCRIPT_VERIFY_DERSIG`, `SCRIPT_VERIFY_CHECKLOCKTIMEVERIFY`, `SCRIPT_VERIFY_CHECKSEQUENCEVERIFY`
for all modes, and `SCRIPT_SIGHASH_RANGEPROOF` for Elements mode

The following script flags are not modelled, because modelling them is practically infeasible,
only relevant for things outside of script execution itself, or only apply to non-modelled parts
of the script: `SCRIPT_VERIFY_SIGPUSHONLY`, `SCRIPT_VERIFY_CONST_SCRIPTCODE`,
`SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_TAPROOT_VERSION`, `SCRIPT_VERIFY_DISCOURAGE_OP_SUCCESS`,
`SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_NOPS`, `SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_WITNESS_PROGRAM`,
`SCRIPT_VERIFY_P2SH`, `SCRIPT_VERIFY_WITNESS`, `SCRIPT_VERIFY_TAPROOT` for all modes,
and `SCRIPT_NO_SIGHASH_BYTE` for Elements mode.

Script size limit is not modelled (the limit of 10000 bytes that exists for segwit and pre-segwit)

## No API except command line

Currently, the only supported API is the command line.

While it is possible to import `bsst` module in python and call functions
from it, anything inside `bsst` module should be considered arbitrary and subject to change without
notice.

The command line interface may also change, but the changes will be noted in [Release notes](release-notes.md)

## Interrupting the solver

When the solver is working, it can be interrupted by sening it a SIGINT signal (usually done with `^C` on the terminal). After interruption, the solver will retry an attempt at solving (with different random seeds, unless solver randomization is disabled), for the maximum amount of tries set with `--max-solver-tries`. To cancel the analysis altogether and quit the program while the solver works, you will need to send a signal other than SIGINT to the program, for example SIGQUIT (usually done with `^\` on the terminal)

## Installation

Simply clone the repo:

`git clone https://github.com/dgpv/bsst/`

`cd bsst`

`./bsst-cli --help`

The file `bsst/__init__.py` is itself a runnable script without any mandatory dependencies except python standard library.
It is possible to just copy `bsst/__init__.py` into a convenient location under convenient name, and run it directly,
without installing `bsst` python module.

## Usage:

        bsst-cli [options] [settings]

## Available options:

  --help

        Show help on usage

  --license

        Show the software license this program is released under

  --version

        Show version

## Available settings:

  Default value for each setting is shown after the '=' sign

  --input-file='-'

        The file of the script to analyze. The dash "-" means STDIN

  --z3-enabled=false

        If true, Z3 theorem prover (https://github.com/Z3Prover/z3)
        will be employed to track and enforce constraints on values processed
        by the script. This will significantly improve the thoroughness of
        the analysis.
        If false, the analysis will be fast, but not as thorough, much fewer
        issues may be detected

  --z3-debug=false

        Enabling this will set `--all-z3-assertions-are-tracked-assertions`
        to true, and also shows all triggered tracked assertions as possible
        script failures

  --points-of-interest=''

        A set of "points" in the script to report the execution state at,
        in addition to the usual information in the report.
        The "point" can be an integer - that means the program counter position
        in the script, or the string "L<num>" where "<num>" is the line number
        in the text of the script

  --produce-model-values=true

        Produce 'model values' for fields of transaction, witnesses, and
        the value(s) on the stack after execution has finished (if
        `--is-incomplete-script` is true or `--cleanstack-flag` is false,
        there may be more than one value on the stack at the end of successful
        execution).
        
        Model values are the values that, when assigned to said fields, do not
        lead to the script failure. If there is only one such possible value,
        it will be shown with '=' between the name and the value in the report,
        otherwise the separator will be ':'.

  --check-always-true-enforcements=true

        Use Z3 to check enforcements for being 'always true': that is,
        the enforcement condition being false means that no valid execution
        paths exist in the script. Turning this setting off skips that
        detection, which means that the analysis will finish faster.

        When condition is detected as 'always true' it will be marked with
        "<*>" in the report. Always-true conditions may indicate either an
        issue with the script (like, doing `$data DUP EQUALVERIFY` instead of
        `DUP $data EQUALVERIFY`), or an opportunity for optimization, if after
        further analysis it is obvious that other conditions make this
        'always true' enforcement unnecessary. Sometimes the enforcement is
        'always true' only in particular execution path (see
        `--mark-path-local-always-true-enforcements`).

        Sometimes 'always true' condition for enforcements can also be detected
        without use of Z3, this settings will not affect these cases.

  --log-progress=true

        Print progress log as the script is analyzed.
        The progress log lines are sent to STDOUT

  --log-solving-attempts=true

        In addition to progress log, log info about each solving attempt

  --log-solving-attempts-to-stderr=false

        In addition to progress log, log info about each solving attempt
        to STDERR

  --all-z3-assertions-are-tracked-assertions=false

        Set names for all Z3 assertions generated, making them "tracked".
        This will severely slow down the solving speed. But it may sometimes
        help to find more about the probable cause for 'always true'
        enforcement or for 'only one possible model value'. Automatically
        set to true if `--z3-debug` is true

  --use-parallel-solving=true

        Enable running several solvers in parallel.
        if `--parallel-solving-num-processes` is not set, then the number
        of CPUs on the machine will be used. Using parallel solvers is
        likely to speed up the solving. Will be automatically turned off
        if `--use-z3-incremental-mode` is true. Parallel solving is only
        available on the platforms that support 'fork' system call for
        'start method' of python multiprocessing module (that means that
        parallel solving is not supported on Windows or MacOS)

  --parallel-solving-num-processes=0

        Number of solver processes to run in parallel. If zero, then
        number of available CPU will be used

  --solver-timeout-seconds=5

        Timeout in seconds after which the Z3 solving attempt will be
        abandoned, and another attempt will start. Zero means no timeout.
        
        When solver randomization is enabled (`--disable-z3-randomization` is
        false), restarting solver can often help to find solution faster

  --solver-increasing-timeout-max=31536000

        Maximum value for solver timeout when increasing timeout is
        employed (via `--solver-increasing-timeout-multiplier`)

  --solver-increasing-timeout-multiplier=1.5

        Multiplier to increase the solver timeout after each attempt
        For example, if set to 1.5 and `--solver-timeout-seconds` is 10,
        on the second attempt the timeout will be 15 seconds, on third addempt
        22.5 seconds, etc.

  --max-solver-tries=100

        Maximum timer of tries for the solver to get sat or unsat result.
        After this number of tries, the analyzer will exit if
        `--exit-on-solver-result-unknown` is true, or will continue analysis.
        In the later case, the analysis might not be correct, because the
        assertions of the unsolved case will be ignored

  --exit-on-solver-result-unknown=true

        If true, then when the solver did not produce sat or unsat after
        `--max-solver-tries` attempts, stop the analysis and exit

  --use-z3-incremental-mode=false

        Incremental mode will use weaker solvers (and the solver can run
        for a long time for certain scripts). Non-incremental mode resets the
        solver for each branch, and re-adds all constraints tracked from the
        start of the script, so it will re-check all the constraints for each
        branch. But because Z3 will use stronger solvers in non-incremental
        mode, solving times will likely to actually be faster than in
        incremental mode.  In incremental mode, the randomizations of z3 seeds
        or shuffling of assertions will not be performed, and no repeated
        attempts at finding solutions will be performed on 'unsat' from solver.
        Also no attempts to check if enforcements can be 'always true' will
        be peformed

  --disable-error-code-tracking-with-z3=false

        Disable error code tracking in Z3 assertions. Script failures
        will still be detected as with enabled error code tracking, but
        the failure will be reported as "untracked constraint check failed".
        Disabling error code tracking can speed up solving, at the price
        of losing information about concrete error codes

  --is-incomplete-script=false

        If true, final result check will be skipped, and
        `--cleanstack-flag` will be set to false

  --restrict-data-identifier-names=true

        If false, identifiers assigned to values in the script via
        specially-formatted comments will be unrestricted, except that
        apostrophe <<'>> is not allowed. Otherwise, these
        identifiers will be checked to be valid python identifiers

  --assume-no-160bit-hash-collisions=false

        If true, it is assumed that 160-bit hashes will never collide,
        and the expression "For all x, y, hashfun(x) == hashfun(y) <=> x == y"
        can be deemed true. (NOTE: it is always assumed that 256-bit hash
        functions will never collide)

  --skip-immediately-failed-branches-on='VERIFY'

        A script fragment that is expected to fail if top of the stack
        is not True. Will be looked for right after opcodes that leave the
        'success' flag on the stack, like for example ADD64 or MUL64. Any
        enforcement inside that script fragment, that would otherwise be
        registered, will be ignored. Sequences like `ADD64 VERIFY` can
        be viewed as a single opcode that fails on invalid arguments.
        This setting allows the analysis to do just that. If for some reason
        the script uses different sequence of opcodes to detect such failures,
        like for example `1 EQUALVERIFY`, you can set this option with the
        string "1 EQUALVERIFY", or empty string to disable this mechanism.

  --is-miner=false

        If true, the analysis will assume that only consensus rules apply,
        and policy rules are not (as what would be the case when the script is
        executed by the miner). It is a good idea to analyze both with
        `--is-miner=true` and `--is-miner=false`, to see if the script behavior
        can be different for 'policy rules' vs 'consensus rules'

  --is-elements=false

        If true, Elements opcodes and rules will be enabled

  --sigversion=tapscript

        Rules for script execution. One of: base, witness_v0, tapscript

  --dont-use-tracked-assertions-for-error-codes=false

        If true, error code detection will use implication instead
        of tracked assertions

  --disable-z3-randomization=false

        Disable randomization for Z3 solver.
        Will likely make solving slower

  --do-progressive-z3-checks=true

        Perform Z3 check after each opcode is symbolically executed.
        When true, analysis time for the whole script will likely be longer,
        but might some failures might be detected faster. Also might give
        clearer reasons for paricular failure when the failure is detected
        right after the opcode rather than at the end of execution path

  --tag-data-with-position=false

        If true, each value pushed on the stack will be tagged with
        the value of program counter at the time it was pushed. This will
        make the analysis treat such values as unique within each execution
        path, even if values might actually be the same

  --tag-enforcements-with-position=true

        If true, each enforcement will be tagged with the value of program
        counter at the time it was enforced by the secipt. This will make the
        analysis treat such enforcements as unique within each execution path,
        even if the enforced conditions might be the same

  --use-deterministic-arguments-order=true

        If true, the opcodes where the order of arguments is not important
        will have their arguments sorted according to their canonical
        representation. For example, ADD(3, 1) will be analysed and represented
        as ADD(1, 3)

  --mark-path-local-always-true-enforcements=true

        If true, the enforcements that are always true only in certain
        execution path, but not in all valid execution paths, will be
        marked with "{*}"

  --discourage-upgradeable-pubkey-type-flag=true

        SCRIPT_VERIFY_DISCOURAGE_UPGRADEABLE_PUBKEY_TYPE

  --strictenc-flag=true

        SCRIPT_VERIFY_STRICTENC

  --witness-pubkeytype-flag=true

        SCRIPT_VERIFY_WITNESS_PUBKEYTYPE

  --minimalif-flag=true

        SCRIPT_VERIFY_MINIMALIF

  --minimaldata-flag=true

        SCRIPT_VERIFY_MINIMALDATA

        If `--minimaldata-flag-strict` is false, immediate data values
        are not subjected to checks: `0x01 VERIFY` will not fail

  --minimaldata-flag-strict=false

        SCRIPT_VERIFY_MINIMALDATA

        Immediate data values are subjected to checks:
        `0x01 VERIFY` will fail, must use `--OP-1` (or just `--1`) instead

        If true, `--minimaldata-flag` is implied to be true

  --nulldummy-flag=false

        SCRIPT_VERIFY_NULLDUMMY
        If this flag is not set explicitly, it will be false with
        `--sigversion=base`, and false otherwise

  --low-s-flag=true

        SCRIPT_VERIFY_LOW_S

  --nullfail-flag=true

        SCRIPT_VERIFY_NULLFAIL

  --cleanstack-flag=true

        SCRIPT_VERIFY_CLEANSTACK
        Will be false if `--is-incomplete-script` is true

  --max-tx-size=1000000

        Maximum transaction size in bytes (used to limit tx weight as
        max_tx_size*4). Only relevant in Elements mode

  --max-num-inputs=24386

        Max possible number of inputs in transaction.
        
        Default value as per https://bitcoin.stackexchange.com/questions/85752/maximum-number-of-inputs-per-transaction
        
        Only relevant in Elements mode, and in Elements the inputs are larger.
        This does not take into account the length of the examined script
        either. So the default value should actually be lower, but still, this
        is OK as an upper bound for now. Might adjust default value later.

  --max-num-outputs=13157

        Max possible number of outputs in transaction.
        
        Default value is a very rough upper bound based on max possible
        non-seqwit size for transaction and min size of output.
        Might adjust default value later.

  --op-plugins=''

        Set of opcode handling plugins to load (paths to python files
        with names ending in '_op_plugin.py')
