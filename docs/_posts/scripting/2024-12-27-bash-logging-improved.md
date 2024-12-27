---
layout: post
title:  "Bash logging, an improvded way"
date:   2024-12-27 17:15:00 +0800
categories: scripting
---

Three months ago I've written [a blog post](../../09/29/bash-logging-with-funcname-lineno.html) to document how to print logs in Bash with function names and line numbers just like in C, however the method documented there relied on `eval` and is not very clean. A few weeks ago I found a better way to do this and I just had time to write it down.

Let's start this with a full demo:
```sh
log_inner() {
    if [[ "${log_enabled[$1]}" ]]; then
        echo "[${BASH_SOURCE##*/}:${1^^}] ${FUNCNAME[2]}@${BASH_LINENO[1]}: ${*:2}"
    fi
}

log_debug() {
    log_inner debug "$@"
}

log_info() {
    log_inner info "$@"
}

log_warn() {
    log_inner warn "$@"
}

log_error() {
    log_inner error "$@"
}

log_fatal() {
    log_inner fatal "$@"
}

initialize() {
    set -euo pipefail
    declare -gA log_enabled=(
        [debug]='y'
        [info]='y'
        [warn]='y'
        [error]='y'
        [fatal]='y'
    )
    local AIMAGER_LOG_LEVEL="${AIMAGER_LOG_LEVEL:-info}"
    case "${AIMAGER_LOG_LEVEL,,}" in
    'debug')
        :
        ;;
    'info')
        log_enabled[debug]=''
        ;;
    'warn')
        log_enabled[debug]=''
        log_enabled[info]=''
        ;;
    'error')
        log_enabled[debug]=''
        log_enabled[info]=''
        log_enabled[warn]=''
        ;;
    'fatal')
        log_enabled[debug]=''
        log_enabled[info]=''
        log_enabled[warn]=''
        log_enabled[error]=''
        ;;
    *)
        log_fatal "Unknown log level ${AIMAGER_LOG_LEVEL}, shall be one of the"\
            "following (case-insensitive): debug, info, warn, error, fatal"
        return 1
        ;;
    esac
}

work() {
    log_warn "Started working..."
    log_fatal "A fatal error occured!"
}

main() {
    initialize
    log_info "Started running..."
    work
    log_info "Ended"
}

main "$@"
```

Save the above to `/tmp/scripter` and run it with `bash /tmp/scripter` and you'll have the following output:

```
[scripter:INFO] main@73: Started running...
[scripter:WARN] work@67: Started working...
[scripter:FATAL] work@68: A fatal error occured!
[scripter:INFO] main@75: Ended
```

As you can see we have both the script name, log level, function name, line number, and the original log.

Let me break it down and tell you how it works, let's focus first on the essential inner logging function:

```sh
log_inner() {
    if [[ "${log_enabled[$1]}" ]]; then
        echo "[${BASH_SOURCE##*/}:${1^^}] ${FUNCNAME[2]}@${BASH_LINENO[1]}: ${*:2}"
    fi
}
```

The function's if-condition does a simple thing: check whether the log level from arg1 (`$1`) was enabled, and only print everything from arg2 (`$2`) onwards with a log prefix when it is enabled.

In the printing line, the used variables and their definitions are:
- `$BASH_SOURCE` is a Bash built-in variable, storing the path of the Bash file that was interpreted, here it would be `/tmp/scripter`, and as `##*/` removes everything before the last `/` (inclusive), `${BASH_SOURCE$$*/}` would be `scripter`;
- `$1` is the log level, `${1^^}` converts the log level to upper case, so if `$1` is `info` then `${1^^}` is `INFO`;
- `$FUNCNAME` is a Bash built-in array containing all of the function names in the stack, backwards in the order they were called, so the full array here would be `FUNCNAME=(log_inner log_[level] main)`, the one we want is thus `${FUNCNAME[2]}`, the actual function that calls our logging wrappers;
- `$BASH_LINENO` is a Bash built-in array containing all of the places where the functions are called in the stack, backwards in the order they were called, so the full array here would be `BASH_LINENO=([line No. where log_inner was called] [line No. where log_[level] was called])`, the one we want is thus `${BASH_LINENO[1]}`, the line number where our logging wrapper was called.
- `$*` is a Bash magic variable that contains all arguments to the current context (here `log_innner`) joined by the first character in `$IFS` (here space) to a single string, we want `${*:2}`, which only contains the second to last argument.

A log wrapper then is defined as follows:

```sh
log_info() {
    log_inner info "$@"
}
```

It just passes its built-in log level and all remaining arguments to the inner printing function. The reason I use dedicated `log_info`, `log_warn`, etc instead of using the inner `log_inner` directly is that I want the logging behaviour to be strictly explicit (remember that we have `set -u` (using undefined variable would result in error) and `set -e` (error would result in Bash exiting), so caller can only call `log_info`, this avoids the error that one might send the wrong logging level)

When one wants to log something they can use one of the logging wrapper in their function:
```sh
work() {
    ...
    log_fatal "A fatal error occured!"
    ...
}
```
This prints the following log, which is very helpful when debugging:
```
[scripter:FATAL] work@68: A fatal error occured!
```

Note also we have a helper struct to record whether a log level was enabled, instead of figuring it out everything logging was triggered:

```sh
declare -gA log_enabled=(
    [debug]='y'
    [info]='y'
    [warn]='y'
    [error]='y'
    [fatal]='y'
)
```

To disable a logging level just set the corresponding value to empty:
```sh
local AIMAGER_LOG_LEVEL="${AIMAGER_LOG_LEVEL:-info}"
case "${AIMAGER_LOG_LEVEL,,}" in
...
'error')
    log_enabled[debug]=''
    log_enabled[info]=''
    log_enabled[warn]=''
    ;;
...
esac
```

This saves the work that is needed when figuring out whether a log level shall be enabled: getting the value from an assotiated Bash array is much faster than doing calculation every time from environment

Like the following:
```sh
if [[ "${log_enabled[info]}" ]]; then
    do_complex_logging_when_info_level_is_enabled
fi
```
is of course lighter than the following:
```sh
case "${AIMAGER_LOG_LEVEL}" in
info|warn|error|fatal)
    do_complex_logging_when_info_level_is_enabled
    ;;
esac
```

If there's no need to figure out the log level again during runtime, then the demo can be simplified to the following:
```sh
log_inner() {
    echo "[${BASH_SOURCE##*/}:$1] ${FUNCNAME[2]}@${BASH_LINENO[1]}: ${*:2}"
}

log_debug() {
    log_inner DEBUG "$@"
}

log_info() {
    log_inner INFO "$@"
}

log_warn() {
    log_inner WARN "$@"
}

log_error() {
    log_inner ERROR "$@"
}

log_fatal() {
    log_inner FATAL "$@"
}

initialize() {
    set -euo pipefail
    local AIMAGER_LOG_LEVEL="${AIMAGER_LOG_LEVEL:-info}"
    case "${AIMAGER_LOG_LEVEL,,}" in
    'debug')
        :
        ;;
    'info')
        log_debug() { :; }
        ;;
    'warn')
        log_debug() { :; }
        log_info() { :; }
        ;;
    'error')
        log_debug() { :; }
        log_info() { :; }
        log_warn() { :; }
        ;;
    'fatal')
        log_debug() { :; }
        log_info() { :; }
        log_warn() { :; }
        log_error() { :; }
        ;;
    *)
        log_fatal "Unknown log level ${AIMAGER_LOG_LEVEL}, shall be one of the"\
            "following (case-insensitive): debug, info, warn, error, fatal"
        return 1
        ;;
    esac
}

work() {
    log_warn "Started working..."
    log_fatal "A fatal error occured!"
}

main() {
    initialize
    log_info "Started running..."
    work
    log_info "Ended"
}

main "$@"
```