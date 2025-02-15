prom2edn
=========

A tool to scrape a Prometheus client and dump the result as JSON.

# Background

(Pre-)historically, Prometheus clients were able to expose metrics as
JSON. For various reasons, the JSON exposition format was deprecated.

Usually, scraping of a Prometheus client is done by the Prometheus
server, which preferably happens with the protocol buffer
format. Sometimes, a human being needs to inspect what a Prometheus
clients exposes. In that case, the text format is used (which is
otherwise meant to allow simplistic _clients_ like shell scripts to
expose metrics to a Prometheus _server_).

However, some users wish to scrape Prometheus clients with programs
other than the Prometheus server. Those programs would usually use the
protocol buffer format, but for small _ad hoc_ programs, that is too
much of an (programming) overhead. JSON comes in handy for these
use-cases, as many languages offer tooling for JSON parsing.

To avoid maintaining a JSON format in all client libraries, the
`prom2edn` tool has been created, which scrapes a Prometheus client
in protocol buffer or text format and dumps the result as JSON to
`stdout`.

# Usage

Installing and building:

    $ GO111MODULE=on go install github.com/prometheus/prom2edn/cmd/prom2edn@latest

Running:

    $ prom2edn http://my-prometheus-client.example.org:8080/metrics
    $ curl http://my-prometheus-client.example.org:8080/metrics | prom2edn
    $ prom2edn /tmp/metrics.prom
    
Running with TLS client authentication:

    $ prom2edn --cert=/path/to/certificate --key=/path/to/key http://my-prometheus-client.example.org:8080/metrics
    
Running without TLS validation (insecure, do not use in production!):

    $ prom2edn --accept-invalid-cert https://my-prometheus-client.example.org:8080/metrics
    
Advanced HTTP through `curl`:

    $ curl -XPOST -H 'X-CSRFToken: 1234567890abcdef' --connect-timeout 60 'https://username:password@my-prometheus-client.example.org:8080/metrics' | prom2edn

This will dump the JSON to `stdout`. Note that the dumped JSON is
_not_ using the deprecated JSON format as specified in the
[Prometheus exposition format
reference](https://docs.google.com/document/d/1ZjyKiKxZV83VI9ZKAXRGKaUKK2BIWCT7oiGBKDBpjEY/edit?usp=sharing). The
created JSON uses a format much closer in structure to the protocol
buffer format. It is only used by the `prom2edn` tool and has no
significance elsewhere. See below for a description.

A typical use-case is to pipe the JSON format into a tool like `jq` to
run a query over it. That looked like the following when the clients
still supported the deprecated JSON format:

    $ curl http://my-prometheus-client.example.org:8080/metrics | jq .

Now simply use `prom2edn` instead of `curl` (and change the query
syntax according to the changed JSON format generated by `prom2edn`):

    $ prom2edn http://my-prometheus-client.example.org:8080/metrics | jq .

Example query to retrieve the number of metrics in the `http_requests_total` metric family (only works with the new format):

    $ prom2edn http://my-prometheus-client.example.org:8080/metrics | jq '.[]|select(.name=="http_requests_total")|.metrics|length'

# JSON format

Note that all numbers are encoded as strings. Some parsers want it
that way. Also, Prometheus allows sample values like `NaN` or `+Inf`,
which cannot be encoded as JSON numbers.

```json
[
  {
    "name": "http_request_duration_microseconds",
    "help": "The HTTP request latencies in microseconds.",
    "type": "SUMMARY",
    "metrics": [
      {
        "labels": {
          "method": "get",
          "handler": "prometheus",
          "code": "200"
        },
        "quantiles": {
          "0.99": "67542.292",
          "0.9": "23902.678",
          "0.5": "6865.718"
        },
        "count": "743",
        "sum": "6936936.447000001"
      },
      {
        "labels": {
          "method": "get",
          "handler": "prometheus",
          "code": "400"
        },
        "quantiles": {
          "0.99": "3542.9",
          "0.9": "1202.3",
          "0.5": "1002.8"
        },
        "count": "4",
        "sum": "345.01"
      }
    ]
  },
  {
    "name": "roshi_select_call_count",
    "help": "How many select calls have been made.",
    "type": "COUNTER",
    "metrics": [
      {
        "value": "1063110"
      }
    ]
  }
]
```

## Using Docker

You can deploy this tool using the [prom/prom2edn](https://registry.hub.docker.com/r/prom/prom2edn/) Docker image.

For example:

```bash
docker pull prom/prom2edn

docker run --rm -ti prom/prom2edn http://my-prometheus-client.example.org:8080/metrics
```
