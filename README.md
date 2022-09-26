<!-- omit in toc -->
# go-nagios

Shared Golang package for Nagios plugins

[![Latest Release](https://img.shields.io/github/release/atc0005/go-nagios.svg?style=flat-square)](https://github.com/atc0005/go-nagios/releases/latest)
[![Go Reference](https://pkg.go.dev/badge/github.com/atc0005/go-nagios.svg)](https://pkg.go.dev/github.com/atc0005/go-nagios)
[![Lint and Build](https://github.com/atc0005/go-nagios/actions/workflows/lint-and-build.yml/badge.svg)](https://github.com/atc0005/go-nagios/actions/workflows/lint-and-build.yml)
[![Project Analysis](https://github.com/atc0005/go-nagios/actions/workflows/project-analysis.yml/badge.svg)](https://github.com/atc0005/go-nagios/actions/workflows/project-analysis.yml)
[![Push Validation](https://github.com/atc0005/go-nagios/actions/workflows/push-validation.yml/badge.svg)](https://github.com/atc0005/go-nagios/actions/workflows/push-validation.yml)

<!-- omit in toc -->
## Table of contents

- [Status](#status)
- [Overview](#overview)
- [Features](#features)
- [Changelog](#changelog)
- [Examples](#examples)
  - [Import this library](#import-this-library)
  - [Use only the provided constants](#use-only-the-provided-constants)
  - [Basic plugin structure](#basic-plugin-structure)
  - [Use a branding callback](#use-a-branding-callback)
  - [Override section header/label text](#override-section-headerlabel-text)
  - [Omit Errors, Thresholds sections](#omit-errors-thresholds-sections)
  - [Collect and emit Performance Data](#collect-and-emit-performance-data)
- [License](#license)
- [References](#references)

## Status

Alpha quality.

This codebase is subject to change without notice and may break client code
that depends on it. You are encouraged to [vendor](#references) this package
if you find it useful until such time that the API is considered stable.

## Overview

This package contains common types and package-level variables used when
developing Nagios plugins. The intent is to reduce code duplication between
various plugins and help reduce typos associated with literal strings.

## Features

- Nagios state constants
  - state labels (e.g., `StateOKLabel`)
  - state exit codes (e.g., `StateOKExitCode`)
- Nagios `CheckOutputEOL` constant
  - provides a consistent newline format for both Nagios Core and Nagios XI
    (and presumably other similar monitoring systems)
- Nagios `ServiceState` type
  - simple label and exit code "wrapper"
  - useful in client code as a way to map internal check results to a Nagios
    service state value
- Supports "branding" callback function to display application name,
  version, or other information as a "trailer" for check results provided to
  Nagios
  - this could be useful for identifying what version of a plugin determined
    the service or host state to be an issue
- Panics from client code are captured and reported
  - panics are surfaced as `CRITICAL` state
  - service output and error details are overridden to panic prominent
- Optional support for emitting performance data generated by plugins
  - NOTE: The implementation of this support is not yet stable and may
    change in the future. See GH-81 and GH-92 for additional details.
- Support for collecting multiple errors from client code
- Support for explicitly omitting Errors section in `LongServiceOutput`
  - this section is automatically omitted if no errors were recorded (by
    client code or panic handling code)
- Support for explicitly omitting Thresholds section in `LongServiceOutput`
  - this section is automatically omitted if no thresholds were specified by
    client code
- Automatically omit `LongServiceOutput` section if not specify by client code
- Support for overriding text used for section headers/labels

## Changelog

See the [`CHANGELOG.md`](CHANGELOG.md) file for the changes associated with
each release of this application. Changes that have been merged to `master`,
but not yet an official release may also be noted in the file under the
`Unreleased` section. A helpful link to the Git commit history since the last
official release is also provided for further review.

## Examples

### Import this library

Add this line to your imports like so:

```golang
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/atc0005/go-nagios"
)
```

and pull in a specific version of this library that you'd like to use.

```console
go get github.com/atc0005/go-nagios@v0.9.0
```

Alternatively, you can use the latest stable tag available to get started:

```console
go get github.com/atc0005/go-nagios@latest
```

### Use only the provided constants

After you've imported this library, reference the exported data types as you
would from any other package. In this example, we reference a specific exit
code for the `OK` state:

```golang
fmt.Println("OK: All checks have passed")
os.Exit(nagios.StateOKExitCode)
```

You can also use the provided state "labels" to avoid using literal string
state values (recommended):

```golang
fmt.Printf(
    "%s: All checks have passed%s",
    nagios.StateOKLabel,
    nagios.CheckOutputEOL,
)

os.Exit(nagios.StateOKExitCode)
```

### Basic plugin structure

First, create an instance of the `ExitState` type and immediately `defer`
`ReturnCheckResults()` so that it runs as the last step in your client code.

Also, avoid calling `os.Exit()` directly from your code. If you do, this
library is unable to function properly; this library expects that *it* will
handle calling `os.Exit()` with the required exit code (and specifically
formatted output).

Also, if you do not `defer` `ReturnCheckResults()` immediately any other
deferred functions in your client code *will not run*.

Here we're optimistic and we are going to note that all went well.

```golang
package main

import (
  // ...
)

func main() {

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    defer nagiosExitState.ReturnCheckResults()

    // more stuff here

    nagiosExitState.ServiceOutput = certs.OneLineCheckSummary(
        nagios.StateOKLabel,
        certChain,
        certsSummary.Summary,
    )

    nagiosExitState.LongServiceOutput := certs.GenerateCertsReport(
        certChain,
        certsExpireAgeCritical,
        certsExpireAgeWarning,
    )

```

For handling error cases, the approach is roughly the same, only you call
`return` explicitly to end execution of the client code and allow deferred
functions to run.

### Use a branding callback

In this example, we'll make a further assumption that you have a `config`
value with an `EmitBranding` field to indicate whether the user/sysadmin has
opted to emit branding information.

```golang
package main

import (
  // ...
)

func main() {

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    defer nagiosExitState.ReturnCheckResults()

    // ...

    if config.EmitBranding {
      // If enabled, show application details at end of notification
      nagiosExitState.BrandingCallback = Branding("Notification generated by ")
    }

    // ...

}
```

the `Branding` function might look something like this:

```golang
// Branding accepts a message and returns a function that concatenates that
// message with version information. This function is intended to be called as
// a final step before application exit after any other output has already
// been emitted.
func Branding(msg string) func() string {
    return func() string {
        return strings.Join([]string{msg, Version()}, "")
    }
}
```

but you could just as easily create an anonymous function as the callback:

```golang
package main

import (
  // ...
)

func main() {

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    defer nagiosExitState.ReturnCheckResults()

    if config.EmitBranding {
        // If enabled, show application details at end of notification
        nagiosExitState.BrandingCallback = func(msg string) func() string {
            return func() string {
                return "Notification generated by " + msg
            }
        }("HelloWorld")
    }

    // ...

}
```

### Override section header/label text

In this example, we override the default text with values that better fit our
use case.

```golang
package main

import (
  // ...
)

func main() {

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    defer nagiosExitState.ReturnCheckResults()

    // Override default section headers with our custom values.
    nagiosExitState.SetErrorsLabel("VALIDATION ERRORS")
    nagiosExitState.SetDetailedInfoLabel("VALIDATION CHECKS REPORT")

    // more stuff here

    nagiosExitState.ServiceOutput = certs.OneLineCheckSummary(
        nagios.StateOKLabel,
        certChain,
        certsSummary.Summary,
    )

    nagiosExitState.LongServiceOutput := certs.GenerateCertsReport(
        certChain,
        certsExpireAgeCritical,
        certsExpireAgeWarning,
    )

```

### Omit Errors, Thresholds sections

In this example, we hide or omit the Errors and Thresholds sections entirely.

```golang
package main

import (
  // ...
)

func main() {

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    defer nagiosExitState.ReturnCheckResults()

    // Hide/Omit these sections from plugin output
    nagiosExitState.HideErrorsSection()
    nagiosExitState.HideThresholdsSection()

    // more stuff here

    nagiosExitState.ServiceOutput = certs.OneLineCheckSummary(
        nagios.StateOKLabel,
        certChain,
        certsSummary.Summary,
    )

    nagiosExitState.LongServiceOutput := certs.GenerateCertsReport(
        certChain,
        certsExpireAgeCritical,
        certsExpireAgeWarning,
    )

```

### Collect and emit Performance Data

This example provides plugin runtime via a deferred anonymous function:

```golang
package main

import (
  // ...
)

func main() {

    // Start the timer. We'll use this to emit the plugin runtime as a
    // performance data metric.
    pluginStart := time.Now()

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    // defer this from the start so it is the last deferred function to run
    defer nagiosExitState.ReturnCheckResults()

    // Collect last minute details just before ending plugin execution.
    defer func(exitState *nagios.ExitState, start time.Time, logger zerolog.Logger) {

        // Record plugin runtime, emit this metric regardless of exit
        // point/cause.
        runtimeMetric := nagios.PerformanceData{
            Label: "time",
            Value: fmt.Sprintf("%dms", time.Since(start).Milliseconds()),
        }
        if err := exitState.AddPerfData(false, runtimeMetric); err != nil {
            zlog.Error().
                Err(err).
                Msg("failed to add time (runtime) performance data metric")
        }

        // Annotate errors (if applicable) with additional context to aid in
        // troubleshooting.
        nagiosExitState.Errors = annotateError(logger, nagiosExitState.Errors...)
    }(&nagiosExitState, pluginStart, cfg.Log)

    // more stuff here

    // This example also assumes that the check results indicate success. You
    // may opt to use a "IsOK()" style check and a switch statement to
    // have the below execute within a default branch.
    nagiosExitState.ServiceOutput = certs.OneLineCheckSummary(
        nagios.StateOKLabel,
        certChain,
        certsSummary.Summary,
    )

    nagiosExitState.LongServiceOutput := certs.GenerateCertsReport(
        certChain,
        certsExpireAgeCritical,
        certsExpireAgeWarning,
    )

    // ...

}
```

and this example provides multiple performance data values explicitly:

```golang
package main

import (
  // ...
)

func main() {

    // Start the timer. We'll use this to emit the plugin runtime as a
    // performance data metric.
    pluginStart := time.Now()

    var nagiosExitState = nagios.ExitState{
        LastError:         nil,
        ExitStatusCode:    nagios.StateOKExitCode,
    }

    // defer this from the start so it is the last deferred function to run
    defer nagiosExitState.ReturnCheckResults()

    // more stuff here

    pd := []nagios.PerformanceData{
        {
            Label: "time",
            Value: fmt.Sprintf("%dms", time.Since(start).Milliseconds()),
        },
        {
            Label: "datacenters",
            Value: fmt.Sprintf("%d", len(dcs)),
        },
        {
            Label: "triggered_alarms",
            Value: fmt.Sprintf("%d", len(triggeredAlarms)),
        },
    }

    if err := nagiosExitState.AddPerfData(false, pd...); err != nil {
        log.Error().
            Err(err).
            Msg("failed to add performance data")
    }

    // This example also assumes that the check results indicate success. You
    // may opt to use a "IsOK()" style check and a switch statement to
    // have the below execute within a default branch.
    nagiosExitState.ServiceOutput = certs.OneLineCheckSummary(
        nagios.StateOKLabel,
        certChain,
        certsSummary.Summary,
    )

    nagiosExitState.LongServiceOutput := certs.GenerateCertsReport(
        certChain,
        certsExpireAgeCritical,
        certsExpireAgeWarning,
    )

    // ...

}
```

This example drops the deferred handling of the plugin runtime metric in order
to illustrate how it would look if handled directly alongside other metrics.

See these issues for further details:

- [GH-81](https://github.com/atc0005/go-nagios/issues/81)
- [GH-92](https://github.com/atc0005/go-nagios/issues/92)

## License

From the [LICENSE](LICENSE) file:

```license
MIT License

Copyright (c) 2020 Adam Chalkley

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## References

- Nagios
  - <https://github.com/nagios-plugins/nagios-plugins/blob/master/plugins-scripts/utils.sh.in>
  - <http://nagios-plugins.org/doc/guidelines.html>
  - <https://www.monitoring-plugins.org/doc/guidelines.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/pluginapi.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/pluginapi.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/perfdata.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/perfdata.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/macrolist.html>
  - <https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/macrolist.html>
- Icinga
  - <https://icinga.com/docs/icinga-2/latest/doc/05-service-monitoring/#plugin-api>
  - <https://icinga.com/docs/icinga-2/latest/doc/05-service-monitoring/#service-monitoring-plugin-api-performance-data-metrics>
- Go Modules
  - <https://www.ardanlabs.com/blog/2020/04/modules-06-vendoring.html>
  - <https://github.com/golang/go/wiki/Modules>
- Panics, stack traces
  - <https://www.golangprograms.com/example-stack-and-caller-from-runtime-package.html>
