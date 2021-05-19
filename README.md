# gerg/net

Forked version of Golang's [net/http package](https://github.com/golang/go/tree/master/src/net/http)
that implements the client-side H2C upgrade flow 
described in [RFC 7540, Section 3.2](https://datatracker.ietf.org/doc/html/rfc7540#section-3.2).

## Usage

This should be a drop-in replacment for `net/http` package, except with h2c upgrade support:
```go
package main

import (
        "fmt"
        "github.com/gerg/net/http"
)

func main() {
        transport := &http.Transport{}
        client := http.Client{Transport: transport}

        req, _ := http.NewRequest("GET", "http://localhost:8080", nil)

        req.Header.Set("Connection", "Upgrade, HTTP2-Settings")
        req.Header.Set("Upgrade", "h2c")
        req.Header.Set("HTTP2-Settings", "AAMAAABkAARAAAAAAAIAAAAA")

        resp, _ := client.Do(req)

        fmt.Printf("Response:\n\t%+v\n\n", resp)
}
```

## How it Works

See the [diff](https://github.com/Gerg/net/compare/fbd0c6a...main) for details.

In brief, we mirror the ALPN upgrade flow 
when we receive `101 Switching Protocols` from the backend.
This will:
1. Set the `http.Transport`'s peristant connection to have a `http2.Transport` for future use
2. Add the existing TCP connection to the `http2.Transport`'s connection pool
3. Seed that connection with Stream 1 (represented the upgrade request)
4. Send the HTTP/2 prefix, to complete the HTTP/2 upgrade and trigger the backend to send the response.

After we establish the HTTP/2 connection,
we then call `completeUpgrade` on the `http2.Transport`
to return the response for the upgrade request.
