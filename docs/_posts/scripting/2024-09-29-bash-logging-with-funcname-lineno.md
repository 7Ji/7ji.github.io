---
layout: post
title:  "Bash logging with Function name and Line No."
date:   2024-09-29 17:00:00 +0800
categories: scripting
---

When writing some lengthy bash script, one might want the script to log with function name and line number so it would be easy to trace some error-prone logics in the future, like how you would use `__FUNCTION__` and `__LINE__` macros in C projects compiled with GCC.

Luckily there're `$FUNCNAME` and `$LINENO` built-in variables. So you could write your logging statement like this:

```sh
myfunc() {
    echo "[DEBUG] ${FUNCNAME}@${LINENO}: Starting some work..."
    if work; then
        echo "[INFO] ${FUNCNAME}@${LINENO}: Successfully finished the work..."
    else
        echo "[ERROR] ${FUNCNAME}@${LINENO}: Failed to do the work !"
        return 1
    fi
}
```

However writing these lengthy prefixes is both annoying and error-prone, and it reduces the information density which is unhelpful when you go back to improve the codes.

It's of course not possible to replace these with functions, as `$FUNCNAME` and `$LINENO` would then not trace the place where functions are actually called, but only their inner state. And it would be even more tedious if you want to conditionally log depending on the log level.

To simplify the latter typing work while keeping `$FUNCNAME` and `$LINENO` as where they're called, and have some conditonal log levels, you can define some "macros" that would be expanded by eval:

```sh
log_common_start='echo -n "['
log_common_end='] ${FUNCNAME}@${LINENO}: " && false'
log_info="${log_common_start}INFO${log_common_end}"
log_warn="${log_common_start}WARN${log_common_end}"
log_error="${log_common_start}ERROR${log_common_end}"
log_fatal="${log_common_start}FATAL${log_common_end}"

# Debugging-only definitions
if [[ "${aimager_debug}" ]]; then
log_debug="${log_common_start}DEBUG${log_common_end}"
else
log_debug='true'
fi
```

With the above definition you can write that function instead like this:

```sh
myfunc() {
    eval "$log_debug" || echo 'Starting some work...'
    if work; then
        eval "$log_info" || echo 'Successfully finished the work...'
    else
        eval "$log_error" || echo 'Failed to do the work !'
        return 1
    fi
}
```

The way this works is that, for logging levels enabled, the logging lines are basically expanded to:
```sh
echo "prefix" && false || echo 'content'
```
and both `echo`es would be executed in this case; 

and for logging levels disabled, the logging lines are basically expanded to:
```sh
true || echo 'content'
```
and no `echo` would be executed in this acse

As a bonus point, you can execute some logics dynamically depending on the log level, e.g.

```sh
if ! eval "$log_info"; then
    echo 'Running specific logic when logging level INFO is enabled'
    some_logic_when_info_is_enabled
else
    other_logic_when_info_is_disabled
fi
```