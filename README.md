# LUMBERJACK

Python logging meets styled output.

## Background

For a python CLI script you might be tempted to use `click.secho()` to show
what the script is doing. It offers the following advantages over standard
logging:

* You can use output colours in a meaningful way (simple logging config of info=green, warning=red isn't very meaningful)
* You can draw a progress bar instead of just spamming lines of text

However, `click.secho()` has the following drawbacks:

* If colourised output is saved to a file, the text output is obscured by all the terminal codes.
* If you run a CLI script via cron or in a production envirionment you actually need the output to go through `logging` so that it gets picked up by the rest of our logging infrastructure.

## Objectives

The goals of this project are:

* Provide minimal wrappers around click/logging such that:
  * You can use a single API to send output to click and/or logging (depending on the context)
  * You can use colours in a meaningful way, but only when the the code is used in a terminal that supports colours
  * Provide CLI options like `--verbose` that increase the output verbosity in a sensible way (e.g., it gives you DEBUG output from your module, but not necessarily DEBUG output from every other python module)

Some other nice-to-have features I'd like to support (if possible):

* Enable/disable DEBUG logging on a running process (inside a container even) without restarting the process.
* Attach to event stream of a running process (inside a container even) without restarting the process.
* control script exit code depending on presence of warnings/errors
* option to also log everything to a file if running interactively

## Contexts

Scripts could be executed in the following contexts and we want logging/output to work well in all of these places:

- Gitlab CI
- Docker interactive - via `docker exec` or `docker run`
- Docker logs (non-interactive, no color)
- CLI util (interactive)
- CLI util piped to a file (non-interactive / no color)
- A process managed by cron/systemd (non-interactive)


# API

## Logging Levels

This module adds an "OUTAGE" log level which is handled the same as "ERROR",
but is used to represent situations where some external service (database
server, internet etc) isn't available. This makes it easier to identify errors
that are fixed by simply restoring the service that's actually broken, or
errors that happen simply because you're deploying an update that by necessity
means some services are down for a few moments.

## CLI entry points

I would hope we can build a single click-style decorator that you can use like this:

    @click.command()
    @logging_options
    def main():
        pass

Which would do the following for you:

* Set logging level to a sensible default depending on the Context (e.g. turn on DEBUG for gitlab CI by default)
* Add a `--verbose` option which can be repeated (`-vvv` to increase verbosity level)
* Add default log handlers for the Context (e.g. log to stderr with timestamps, maybe colours)
* Use `sys.exit(1)` if any logs of `log.error()` or higher were generated while the script executed (maybe).


## Logging Functions

The logger would support the following extra api methods for more symantic logging:

### log.heading(msg, **extra)

Calls `log.info()` but with metadata that tells output handler to use "bright
yellow" colour if the context supports it, so that it is obvious a new section
of CLI output is starting.

### log.winning(msg, **extra)

Calls `log.info()` but with metadata that tells output handler to use "green"
(success) colour if the context supports it

### log.losing(msg, **extra)

Calls `log.info()` but with metadata that tells output handler to use "red"
(failure) colour if the context supports it

### log.remark(msg, **extra)

Calls `log.info()` but with metadata that tells output handler to use a "light
cyan" colour that doesn't draw attention to itself

### log.unknown(msg, **extra)

Calls `log.info()` but with metadata that tells output handler to use a
"yellow" colour that indicates a state of n to itself


## Meaningful Verbosity Escalation

The package would include a context manager that can be used like this:

    with lumberjack.TaskLogger("Process csv file {csvfile}") as logger:
        logger.info(...)
        with lumberjack.TaskLogger("Downloading csv file {csvfile}") as logger2:
            logger2.info(...)
        logger.info(...)
        logger.success()
        # etc

Depending on the verbosity level provided by `--verbose` flags, the first
`logger`'s info() messages might be shown on stderr, but the second `logger2`
messages might not. Even though they are both "INFO", we only want to see the
outermost "INFO" messages until a higher verbosity level is requested. Note
also that increasing the verbosity level will turn on "DEBUG" messages for
the current level as well.

We can also do some other "clever" things if our logging API includes a context
manager:

* Add increasing amounts of whitespace (indentation) to logging output so that
  it's easy to see which log entries are related
* Add an INFO "Job XXX Finished (1m 35s)" when the context manager finishes
* Rather than having separate INFO "starting the job" with an INFO "finished the
  job", we could push "starting the job ..." to your terminal and append "...
  OK" to the end of the line when the context manager exits, or if the context
  manager encounters an exception, dump out every single log entry that
  happened since the start of the context manager so that you always get all
  the log messages when something fails.


## Activity Summaries

It would also be nice to attach "summary" information to a logging context so
that even if you don't see the individual messages, you still get a summary at
the end.

    with lumberjack.TaskLogger("Process csv file {csvfile}") as logger:
        logger.summary(num_total,      'Rows processed')
        logger.summary(num_successful, 'Rows processed successfully', style='winning')
        logger.summary(num_errors,     'Rows had errors',             style='losing')
        logger.summary(num_skipped,    'Rows were skipped',           style='remark')

In addition to the summary numbers you provide via the `.summary()` method, the
TaskLogger will also output a line showing how many log entries of level
WARNING or higher were recorded during the task.



# Verbosity Management Take II

**experimental ideas**

Instead of using context managers to manage verbosity levels, what if we added some new logging levels?

    ...
    WARNING = 30
    *HEADING = 29
    *LOSING = 28
    *UNKNOWN = 27
    *WINNING = 26
    *DESTRUCTIVE = 25  # script is performing some destructive operation like truncating a table
    *REMARK = 24
    INFO = 20
    *SERVICECALL = 19  # executing mysql queries; making http requests; etc
    *STATISTIC = 18  # how many files were created, how many queries executed, etc
    DEBUG = 10


We can set the default logging level to "UNKNOWN" for CLI scripts so that you
get messages about bad things and unknown things, as well as the major
headings. As you add more `--verbose` flags you get the WINNING messages, INFO
messages, etc.


# Other Packages

Other logging packages I researched

https://click-log.readthedocs.io/en/stable/

This provides `@simple_verbosity_option()` decorator that will set log level to
DEBUG/INFO/WARNING for you.


https://coloredlogs.readthedocs.io/en/latest/

`colorlogs` will give you nicely formatted / coloured logs which at least make errors red.


https://github.com/glentner/LogAlpha

This sounded interesting, but then it turned out that none of the documentation was finished, so no.


https://pypi.org/project/Autologging/

This has some nice class/function decorators for logging enter/exit trace
events for function calls, which is useful to be able to turn on separately
sometimes.


https://pypi.org/project/slack-logging

Send logs to a slack channel



https://pypi.org/project/multilog/

Multilog is a daemon that aggregates logs from a unix/network socket before
recording them somewhere. This is useful when you have multiple python
processes all trying to log to a single file, since python file logging isn't
multi-process-safe.


https://pypi.org/project/tqdm

This just does nice progress bars for you.
