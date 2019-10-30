apptracer
----

[![GoDoc][1]][2] [![License: MIT][3]][4] [![Release][5]][6] [![Build Status][7]][8] [![Co decov Coverage][11]][12] [![Go Report Card][13]][14] [![Code Climate][19]][20] [![BCH compliance][21]][22] [![Downloads][15]][16]

[1]: https://godoc.org/github.com/evalphobia/apptracer?status.svg
[2]: https://godoc.org/github.com/evalphobia/apptracer
[3]: https://img.shields.io/badge/License-MIT-blue.svg
[4]: LICENSE.md
[5]: https://img.shields.io/github/release/evalphobia/apptracer.svg
[6]: https://github.com/evalphobia/apptracer/releases/latest
[7]: https://travis-ci.org/evalphobia/apptracer.svg?branch=master
[8]: https://travis-ci.org/evalphobia/apptracer
[9]: https://coveralls.io/repos/evalphobia/apptracer/badge.svg?branch=master&service=github
[10]: https://coveralls.io/github/evalphobia/apptracer?branch=master
[11]: https://codecov.io/github/evalphobia/apptracer/coverage.svg?branch=master
[12]: https://codecov.io/github/evalphobia/apptracer?branch=master
[13]: https://goreportcard.com/badge/github.com/evalphobia/apptracer
[14]: https://goreportcard.com/report/github.com/evalphobia/apptracer
[15]: https://img.shields.io/github/downloads/evalphobia/apptracer/total.svg?maxAge=1800
[16]: https://github.com/evalphobia/apptracer/releases
[17]: https://img.shields.io/github/stars/evalphobia/apptracer.svg
[18]: https://github.com/evalphobia/apptracer/stargazers
[19]: https://codeclimate.com/github/evalphobia/apptracer/badges/gpa.svg
[20]: https://codeclimate.com/github/evalphobia/apptracer
[21]: https://bettercodehub.com/edge/badge/evalphobia/apptracer?branch=master
[22]: https://bettercodehub.com/

`apptracer` is multiple trace system wrapper for Stackdriver Trace, AWS X-Ray and LightStep.

# Usage

```go
import (
	"github.com/evalphobia/apptracer"
	"github.com/evalphobia/apptracer/platform/localdebug"
	"github.com/evalphobia/apptracer/platform/lightstep"
	"github.com/evalphobia/apptracer/platform/opencensus/datadog"
	"github.com/evalphobia/apptracer/platform/opencensus/xray"
	"github.com/evalphobia/apptracer/platform/opencensus/stackdriver"
)

// global singleton tracer.
var tracer *apptracer.AppTracer

// Init initializes trace setting.
func Init() {
	// init tracer
	tracer = apptracer.New(apptracer.Config{
		Enable:      true,
		ServiceName: "my-cool-app",
		Version:     "v0.0.1",
		Environment: "stage",
	})

	// add localdebug trace (show trace data on the given Logger).
	if isDebug() {
		localCli := localdebug.NewClient(localdebug.Config{
			Logger: nil, // when it's nil, default std.err logger is set.
		})
		tracer.AddClient(localCli)
	}

	// add LightStep
	lsCli := lightstep.NewClient(lightstep.Config{
		AccessToken: "<LightStep access token>",
		ServiceName: "stage-my-cool-app",
	})
	tracer.AddClient(lsCli)

	// add opencensus trace
	{
		// datadog (installed agent is needed)
		ddExp, err := datadog.NewExporter(context.Background(), "my-cool-app")
		if err != nil {
			panic(err)
		}

		// stackdriver trace
		projectID := "test-morikawa"
		sdExp, err := stackdriver.NewExporter(context.Background(), stackdriver.Config{
			// auth setting
			PrivateKey: "<GCP private key>",
			Email:      "<GCP email address>",
			// Filename: "/path/to/pem.json",
		}, "<GCP projectID>")
		if err != nil {
			panic(err)
		}

		// aws xray trace
		xrayExp, err := xray.NewExporter(context.Background())

		ocCli := opencensus.NewClient(ddExp, sdExp, xrayExp)
		opencensus.SetSamplingRate(1) // 100%
		tracer.AddClient(ocCli)
	}

	tracer.Flush()
}

func MainFunction(ctx context.Context, r *http.Request) {
    // Before use tracer and get span, you have to call Init() to initialzed tracer.

	// defer tracer.Close()
	defer tracer.Flush()


	span := tracer.Trace(ctx).NewRootSpanFromRequest(r)
	defer func(){
		span.SetResponse(200).Finish()
		// debug print for localdebug client
		// span.OutputSummary()
	}()

	childFunction1(ctx)
	childFunction2(ctx)
	childFunction3(ctx)
}

func childFunction1(ctx context.Context) {
	defer tracer.Trace(ctx).NewChildSpan("childFunction1").Finish()
	// ... process something
}

// get user data
func childFunction2(ctx context.Context) error {
	var user string
	var err error

	span := tracer.Trace(ctx).NewChildSpan("childFunction2")
	defer func(){
		span.SetUser(user).SetError(err).Finish()
	}()

	user, err = getUserName()
	if err != nil {
		return err
	}

	// process something ...

	return nil
}


// exec sql
func childFunction3(ctx context.Context) error {
	var sql string

	span := tracer.Trace(ctx).NewChildSpan("childFunction3")
	defer func(){
		span.SetSQL(sql).Finish()
	}()

	sql = "SELECT * FROM users"
	users, err = execSQL(sql)
	if err != nil {
		return err
	}

	// process something ...

	return nil
}
```

## Supported Platform

- OpenCensus
	- Datadog trace
        - Google Stackdriver trace
        - New Relic
        - AWS X-Ray
- LightStep

## TODO

- Supports microservices tracing
- Supports other platforms


# Whats For?

One code for multiple tracing platforms!

- Compare and PoC tracing platforms .
- Multiple platforms with different sampling rates.

