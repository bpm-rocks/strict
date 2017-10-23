BPM Library: Strict
===================

Enable a "strict mode" for Bash. This also sets up an `ERR` trap to display a stack trace so you know exactly where your Bash script terminated.


Strict Mode
===========

There is a tip on [another website](http://redsymbol.net/articles/unofficial-bash-strict-mode/) for how to enable a stricter environment for Bash. It is a combination of flags and settings that will help point out problematic code.

    set -eEu -o pipefail
    shopt -s extdebug
    IFS=$'\n\t'
    trap 'strict::failure $?' ERR

A brief summary of what each option does:

* `set -e`: Exit immediately if a command exits with a non-zero status, unless that command is part a test condition. On failure this triggers the `ERR` trap. **There are some contexts that will disable this setting!**
* `set -E`: The `ERR` trap is inherited by shell functions, command substitutions and commands in subshells. This helps us use `strict::failure` wherever `set -e` is enabled.
* `set -u`: Exit and trigger the `ERR` trap when accessing an unset variable. This helps catch typos in variable names.
* `set -o pipefail`: The return value of a pipeline is the value of the last (rightmost) command to exit with a non-zero status. So, `a | b | c` will return `a`'s status if it fails, `b`'s status if `b` fails and `a` passes, or `c`'s status if the other two worked. It is easier to catch problems during the middle of processing a pipeline this way.
* `shopt -s extdebug`: Enable extended debugging. Bash will track the parameters to all of the functions in the call stack, allowing the stack trace to also display the parameters that were used.
* `IFS=$'\n\t'`: Set the "internal field separator", which is a list of characters use for word splitting after expansion and to split lines into words with the `read` builtin command. Normally this is `$' \t\n'` and we're removing the space. This helps us catch other issues when we may rely on `$IFS` or accidentally use it incorrectly.
* `trap 'strict::failure $?' ERR`: The ERR trap is triggered when a script catches an error. `wickStrictModeFail` attempts to produce a stack trace to aid in debugging. We pass `$?` as the first argument so we have access to the return code of the failed command.


Why Use Strict Mode?
====================

This sure sounds like it causes many more problems than it solves. People will say that their scripts complain loudly and now break often. The normal way of dealing with variables now causes fatal errors in scripts. Variables that had entire commands now do not execute properly and non-zero status codes are now being reported where they used to just work.

So why use this?

For the same reason that you use `jslint` for testing if JavaScript is good enough or `-Wall` when compiling C code: you want to know when there are problems and you want those problems exposed very early in the process. Also, if a calling program runs your code in strict mode, you want to make sure it still works, right?

For an example of what the stack trace looks like, here is one from a test program I used:

    Error detected - status code 1
    Command:  false
    Location:  ./in-functions, line 13
    Stack Trace:
        [1] two(): ./in-functions, line 13 -> two 2 22 Two
        [2] one(): ./in-functions, line 8 -> one Three\ words\ together second-argument
        [3] main(): ./in-functions, line 17 -> main

You can see the file's name, line number and the name of the function that was called. The exact command is also preserved and is shown after the arrow on the right. At the top you can see that there is a status code displayed and the command which actually failed (it was executing `false`). If this was in a pipeline you would see the pipe status for each command in the pipeline.


Common Problems and Solutions
=============================

When you enable strict mode, it will gladly show you many errors and happily kill your program. Bash doesn't apparently have any feelings and doesn't care about a programmer's emotional state, but we do. Here's many common problems, why it's bad, and ways to correct them.


Early Termination
-----------------

Whenever any command fails and isn't caught, the script will terminate. For instance, you could have this command to remove lines saying "REMOVE".

    grep -v "REMOVE" old-file.txt > new-file.txt

If there are no lines with "REMOVE", then `grep` will return a non-zero status code, causing an abrupt termination of the program. To solve this we have several techniques.

    # Option 1 - use it as a condition
    # Try to use it in a condition
    if grep -v "REMOVE" old-file.txt > new-file.txt; then
        log::debug "One or more lines were removed"
    else
        log::debug "No lines were found"
    fi


    # Option 2 - ignore failures for one command
    grep -v "REMOVE" old-file.txt > new-file.txt || :

The `|| :` at the end means it should run `grep` and if that fails, run `:`.  Calling `:` is the same as calling `true` and is the more traditional method.


Failures in Pipes
-----------------

You may send data through some sort of formatting utility, similar to this:

    myProgram | formatter

The `formatter` command is pretty simple and will always return 0 (success). Your command seems to fail even though formatter always succeeds and that's because the return code from `myProgram` is not zero.

To counter this issue, you really want to ignore the return value from `myProgram`.

    # You can ignore just the return code for myProgram
    (myProgram || :) | formatter

    # You can ignore the return code for the whole line
    myProgram | formatter || :

The better option is to ignore only the return code for `myProgram`. That way errors from other parts of the line can still be caught. Also, be wary of using Bash functions in pipes like this because errors can be disabled in some contexts.


`Unbound variable $1` and Optional Parameters
---------------------------------------------

With `set -u` enabled, you are unable to use positional parameters that are not defined. Let's say you have a function that will `echo` the first parameter.

    # This is the old function
    echoFirst() {
        echo "$1"
    }

When you run `echoFirst` with no arguments, the above will fail. To adapt this you should do one of two things.

    # You can default to an empty value, which is similar to how Bash
    # would interpret this without set -u
    echoFirst() {
        echo "${1-}"
    }

    # Alternately you can test for the number of arguments
    echoFirst() {
        if [[ $# -gt 0 ]]; then
            echo "$1"
        fi
    }

Using `$@` will never give an error. It is because that magical construct will expand to nothing when nothing is passed, or expand to each passed argument, and have them all quoted.

    # This works but would also send $2, $3, etc.
    echoFirst() {
        echo "$@"
    }


`Unbound variable ${ARRAY[@]}`
------------------------------

When working with arrays, Bash exhibits some odd behavior and it varies based on what version of Bash is being used. Let's start with this example:

    ARR=()
    echo "All elements in the array:" "${ARR[@]}"

That code will break in strict mode even though `$ARR` is defined. Bash treats empty arrays as though they are unset variables.

You could wrap it in a conditional, which adds more code to the project.

    if [[ ${#ARR[@]} -gt 0 ]]; then
        echo "All elements in the array:" "${ARR[@]}"
    fi

You could substitute an empty value. This would end up sending an empty string as a second argument to `echo`, which could potentially cause problems with other functions.

    # Option 2 - Substitue an empty value
    echo "All elements in the array:" "${ARR[@]-}"

The best option is to use some interesting Bash syntax. It will use the values of the expanded array if the array is set. Please be very careful with the quotes; adding more quotes will cause this syntax to break.

    echo "All elements in the array:" ${ARR[@]+"${ARR[@]}"}


Appending to an Array
---------------------

There are several ways to append values to an array.

    # Define an array.
    ARR=()

    # Does not work when ARR is empty.
    ARR=("${ARR[@]}" "another value")

    # Bash 4.3 and newer.
    ARR+=("another value")

    # Works in Bash 3 and newer.
    ARR[${#ARR[@]}]="another value"


Getting the Return Code
-----------------------

We used to be able to run a command and catch its return code.

    # Will fail with strict mode
    someCommandThatMayFail

    if [[ $? -ne 0 ]]; then
        echo "There was a failure that will need to get handled"
    fi

With the strict mode in place it is much harder to do that. Instead of fiddling around with temporarily disabling strict mode, use the helper function.

    strict::run RESULT someCommandThatMayFail

    if [[ $RESULT -ne 0 ]]; then
        echo "There was a failure that will need to get handled"
    fi


### Running a Command in a Variable

Many Bash scripts will create a command in a variable and execute it. Because `$IFS` changed, the code no longer works. This is a good thing! It is difficult to get the escaping correct in case paths or arguments contain spaces or other characters. Let's explore the problem.

    # Secretly this command relies on IFS.
    CMD="echo one two three"  # Initial command
    CMD="$CMD four"  # Append an argument
    $CMD  # Run the command

The correct thing to do is to use an array and add all arguments to the array.

    # This command does not rely on IFS
    # Use this instead.
    CMD=(echo one two three)  # Initial command
    CMD+=(four)  # Append an argument
    "${CMD[@]}"  # Run the command

Let's take another example to show why you should not use the first method.

    # Intentionally do not use the built-in echo.
    CMD="$(which echo) \"I'm here!\""
    $CMD

That could fail, especially if someone has updated their `$PATH` to use newer versions of tools and if the command now has a space in it's file path. For instance, my `which echo` could easily report `/home/user/testing programs/bin/echo` and you can see the command that gets executed would look like this.

    /home/user/testing programs/bin/echo "I'm here!"
    ^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^
    command            argument 1        argument 2

What you really want is for the spaces in the command to be kept. Your two options are detailed here.

    # Add more quoting and escaping, which doesn't work with some files.
    CMD="\"$(which echo)\" \"I'm here!\""
    $CMD

    # Switch to using an array
    CMD=("$(which echo)" "I\'m here!")
    "${CMD[@]}"


Slicing an Array
----------------

The syntax `("${array[@]:1}")` is supposed to return a copy of `$array` without the first element. Until Bash 4.0-rc1, this does not work when `$IFS` is set to a non-standard value. Unsetting `$IFS` or setting it to a space works. So, here's the best way to slice the array:

    local oldIfs

    oldIfs=$IFS
    IFS=
    arrayCopy=("${array[@]:1}")
    IFS=$oldIfs




Contexts That Disable Exit On Error
===================================

When running in strict mode (specifically `set -e`), Bash should exit when any command returns a non-zero error code. For our explanations, let's assume this file is called `./error-context-setup`, and is also available in the [examples](example/).

    # example/error-context-setup
    set -e

    errorCommand() {
        false
        echo "Should never get here with 'set -e' enabled."
    }

We can use the error code to abort the program, like the following. `errorCommand` will return a non-zero status code because of `false`, the `echo` in `errorCommand` will not be executed, it will come back to this script and also terminate. The script will give an error code of 1.

    # example/error-context-simple
    . ./error-context-setup
    errorCommand
    echo "Should not see this"

Strange things happen when you start to use any conditional, like this typical example.

    # example/error-context-problem
    . ./error-context-setup
    if ! errorCommand; then
        echo "Error command failed, as expected."
    else
        echo "Weird, this should not happen."
    fi

We previously established that `errorCommand` will terminate and return a non-zero status. However, when the above script is executed, we get the following and the script terminates successfully.

    Should never get here with 'set -e' enabled.
    Weird, this should not happen.

Without `set -e`, the function `errorCommand` would even though the `false` exited with a non-zero value. Also, `errorCommand` would exit with 0 because `echo` completed successfully. This is expected behavior without `set -e`.

Technically `set -e` is still there, but the context changes when you put the function call inside of any conditional. It will no longer abort the code nor trigger errors. There's no way to enable the abort behavior once inside of this context. This would apply even if you are running a subshell instead of a function.

This can happen in many contexts.  The description from [a bug logged](http://austingroupbugs.net/view.php?id=52) to fix the documentation says it can happen for any context including `||`, `&&`, `while`, `until`, `if`, `elif` or `!`.  Many examples are in [`error-context-test`](example/error-context-test).


How to Fix
----------

The best option is to ensure all Bash functions are written to operate in this special mode. You do that by capturing every possible command that could have stopped execution and force an end of the function.

    # Old code
    result=$(ls some-file)
    list=( "some" "words" $(runSomething) )
    grep -q "words" in-this-file
    wickGetIfaceIp ipAddress tun0

    # New code
    result=$(ls some-file) || return $?
    list=( "some" "words" $(runSomething) ) || return $?
    grep -q "words" in-this-file || return $?
    wickGetIfaceIp ipAddress tun0 || return $?

When you ensure that all functions are written this way, then they will always exit early regardless of strict mode and regardless of this error exit context. Consistent, predictable results from your code makes it far more reliable.


Installation
============

Add to your `bpm.ini` file the following dependency.

    [dependencies]
    0=strict

Run `bpm install` to add the library. Finally, use it in your scripts.

    #!/usr/bin/env bash
    . bpm
    bpm::include strict

    strict::mode

There are a few additional functions that allow you to run commands or turn off strict mode as well.


API
===


[//]: # (AUTOGENERATED FROM libstrict - START)

`strict::mode()`
----------------

Enables "strict mode" for Bash, based off [unofficial bash strict mode]. Errors will kill the program.  Accessing undefined variables will cause errors (and exit the program).  Commands in pipelines that return a non-zero status code will also cause errors and kill the program.  An ERR trap is also enabled that will produce a stack trace when errors happen.

Not all errors are caught. There's an error context in Bash that ignores errors, so be careful when you intend to rely on this behavior. It is only here as a safety net to catch you in case of problems, not a babysitter to prevent you from starting the house on fire.

This is intended to be used at the beginning of your shell scripts in order to ensure correctness in your programming.

Examples

    #!/usr/bin/env bash
    . bpm
    bpm::include strict
    strict::mode

[unofficial bash strict mode]: http://redsymbol.net/articles/unofficial-bash-strict-mode/

Returns nothing.


`strict::disable()`
-------------------

Turns off strict mode.

Examples

    # Turn on strict mode
    strict::mode

    # Turn off strict mode
    strict::disable

Returns nothing.


`strict::failure()`
-------------------

The ERR trap calls this function to report on the error location right before dying.  See `strict::mode` for further details.

* $1 - Status from failed command.

Example

    # This sets the error trap.
    strict::mode

    # Cause an error
    ls some-file-that-is-not-there
    # This calls the ERR trap, passing 2 as the status code.

Returns nothing.


`strict::run()`
---------------

Runs a command and captures its return code, even when strict mode is enabled.  The variable name you specify is set to the return code of the command.

* $1   - Name of variable that should get the return value / status code.
* $2-@ - Command and arguments to run.

This is intended to be used along with `strict::mode`.  It helps counter the newer Bash behavior where the error exit flag is suppressed in specific contexts.

Execution of a command happens in a subshell.  If you are running a function with `strict::run`, please keep in mind that it can not export values to the parent context for use by the caller.

Example:

    #!/usr/bin/env bash
    . bpm
    bpm::include strict
    strict::mode

    strict::run result grep "some-string" /etc/some-file.cfg > /dev/null 2>&1

    if [[ "$result" -eq 0 ]]; then
        echo "some-string was found"
    else
        echo "some-string was not found"
    fi

Returns nothing.

[//]: # (AUTOGENERATED FROM libstrict - END)


License
=======

This project is placed under an [MIT License](LICENSE.md).
