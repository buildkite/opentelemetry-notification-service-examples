# OpenTelemetry Notification Service Examples

[![Configure Notification Service](https://img.shields.io/badge/Add%20Notification%20Provider-140d30?style=for-the-badge&logo=buildkite&logoColor=30f2a2&label=Buildkite&color=30f2a2&link=https%3A%2F%2Fbuildkite.com%2Forganizations%2F~%2Fservices)](https://buildkite.com/organizations/~/services)

> [!IMPORTANT]
> OpenTelemetry Notification Service is currently in Private Preview. Please contact Buildkite support for access.

## Creating a new service

[Create a new OpenTelemetry Notification Service](https://buildkite.com/organizations/~/services/open_telemetry_tracing/new) in your organization's Notification Services settings (under Integrations).

![opentelemetry notification service form](./images/form.png)

## Endpoint

Please provide the base URL for your OTLP endpoint. Do not include the `/v1/traces` path as that automatically appended by the Buildkite OpenTelemetry exporter.

- We currently only support the [OTLP/HTTP](https://opentelemetry.io/docs/specs/otlp/#otlphttp) binary protobuf encoding.
- We currently only support sending trace data, but may introduce other OpenTelemetry signals in the future.
- The endpoint must be accessible over the internet.

## Headers

Add any additional HTTP headers to the request. Depending on the destination, you may need to specify API keys or other headers to influence the behaviour of the downstream collector. Values for headers are always stored encrypted server-side.

Here are some common examples.

### Bearer token

Key: `Authorization`
Value: `Bearer <your-token>`

See [bearer-token-auth-debug.yml](./collector-config/bearer-token-auth-debug.yml) for example OpenTelemetry Collector configuration.

### Basic auth

First, create a base64 encoded string of the username and password separated by a colon.

```bash
echo -n "${USER}:${PASSWORD}" | base64
```

Key: `Authorization`
Value: `Basic <base64 encoded ${USER}:${PASSWORD})>`

See [basic-auth-debug.yml](./collector-config/basic-auth-debug.yml) for example OpenTelemetry Collector configuration.

## Honeycomb

Set the Endpoint to `https://api.honeycomb.io`, or `https://api.eu1.honeycomb.io ` if your Honeycomb team is in the EU instance.

Add the require header:

| Key                | Value                 |
| ------------------ | --------------------- |
| `x-honeycomb-team` | `<Honeycomb API key>` |


For more information, see the honeycomb documentation: https://docs.honeycomb.io/send-data/opentelemetry/#using-the-honeycomb-opentelemetry-endpoint

## Datadog Agentless OpenTelemetry (Preview)

Endpoint: `https://trace.agent.datadoghq.com/api/v0.2/traces`

Add the required headers:

| Key              | Value               |
| ---------------- | ------------------- |
| `dd-protocol`    | `otlp`              |
| `dd-api-key`     | `<Datadog API key>` |
| `dd-otlp-source` | `${YOUR_SITE}`      |

Replace `${YOUR_SITE}` with the organization name you received from Datadog.

For more information, see the Datadog documentation:

https://docs.datadoghq.com/opentelemetry/setup/agentless/traces/

## Datadog via OpenTelemetry Collector

See [bearer-token-auth-datadog.yml](./collector-config/bearer-token-auth-datadog.yml) for an example of forwarding traces to Datadog APM using the Datadog exporter.

## Computing OpenTelemetry metrics from OpenTelemetry traces

The OpenTelemetry collector can be used to process incoming trace spans and generate custom metrics on the fly using the [signaltometrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/signaltometricsconnector) processor, which can be stored in metric stores like Prometheus or InfluxDB.

See [bearer-token-auth-signal-to-metrics-otlp.yml](./collector-config/bearer-token-auth-signal-to-metrics-otlp.yml)

## Propagating traces to Buildkite agents

See Buildkite documentation for OpenTelemetry tracing from the Buildkite Agent, including spans for plugin and agent hooks.
https://buildkite.com/docs/agent/v3/tracing#using-opentelemetry-tracing

Propagating trace spans from the OpenTelemetry Notification service requires Buildkite Agent [v3.100](https://github.com/buildkite/agent/releases/tag/v3.100.0) or newer.

### Required Agent Flags / Environment Variables

To propagate traces from the Buildkite backend through to the agent running the job, then include the following CLI flags to `buildkite-agent start` or use environment variables.

| Flag                              | Environment Variable                      |  Value                        |
| --------------------------------- | ----------------------------------------- | ----------------------------- |
| `--tracing-backend`               | `BUILDKITE_TRACING_BACKEND`               | `opentelemetry`               |
| `--tracing-propagate-traceparent` | `BUILDKITE_TRACING_PROPAGATE_TRACEPARENT` | `true`                        |
| `--tracing-service-name`          | `BUILDKITE_TRACING_SERVICE_NAME`          | `buildkite-agent` (default)   |
|                                   | `OTEL_EXPORTER_OTLP_ENDPOINT`             | `http://otel-collector:4317`  |
|                                   | `OTEL_EXPORTER_OTLP_HEADERS`              | `key1=value1,key2=value2`     |
|                                   | `OTEL_EXPORTER_OTLP_PROTOCOL`             | `grpc`                        |

### On Buildkite Hosted Agents

To export traces from hosted agents, this currently requires using a custom Agent Image

```dockerfile
# this is the same as --tracing-backend opentelemetry
ENV BUILDKITE_TRACING_BACKEND="opentelemetry"

# this is the same as --tracing-propagate-traceparent
ENV BUILDKITE_TRACING_PROPAGATE_TRACEPARENT="true"

# service name is configurable
ENV OTEL_SERVICE_NAME="buildkite-agent"

# the agent only supports GRPC transport
ENV OTEL_EXPORTER_OTLP_PROTOCOL="grpc"

# the GRPC transport requires a port to be specified in the URL
ENV OTEL_EXPORTER_OTLP_ENDPOINT="https://your-otel-endpoint:443"

# authentication of traces is done via tokens in headers
ENV OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <token>,x-my-header=value"
```

## OpenTelemetry Collector

The OpenTelemetry collector is an open source service for collecting, exporting and processing telemetry signals.

See [collector-config](./collector-config) for examples of OpenTelemetry collector configuration.

If using the `otel/opentelemetry-collector-contrib` Docker image, you can configure the collector by mounting the your config file at `/etc/otelcol-contrib/config.yaml` or by overriding the `command` to `--config=env:OTEL_CONFIG` and setting the `OTEL_CONFIG` environment variable to the *contents* of your config file.

Consult the [Deployment](https://opentelemetry.io/docs/collector/deployment/) guide in the OpenTelemetry documentation for more information about hosting the collector.

The OpenTelemetry collector also supports many downstream data stores via [exporters](https://opentelemetry.io/docs/collector/configuration/#exporters) including StatsD, Prometheus, Kafka, Tempo, OTLP as well as many other observability tools and vendors. See the [OpenTelemetry registry](https://opentelemetry.io/ecosystem/registry/?s=&component=exporter&language=collector) for a more complete list of supported exporters.

### References

- https://opentelemetry.io/docs/collector/
- https://github.com/open-telemetry/opentelemetry-collector
- https://github.com/open-telemetry/opentelemetry-collector-contrib
- https://opentelemetry.io/docs/collector/configuration/#authentication
- https://hub.docker.com/r/otel/opentelemetry-collector-contrib

## Validating OpenTelemetry Collector configuration

`otelcol validate` lets you validate your [collector configuration](https://opentelemetry.io/docs/collector/configuration/).

For example, to validate one of the example configs in [collector-config](./collector-config), say `basic-auth-debug.yml` you could run the following command:

```bash
docker run --rm -it -v $(pwd)/collector-config:/config otel/opentelemetry-collector-contrib validate --config=/config/basic-auth-debug.yml && echo "config valid"
```

Or for a config file like  `bearer-token-auth-datadog.yml` that specifes environment variables in the, you wouuld run the following command, noting the `-e` flags to provide the environment variables:

```bash
docker run --rm -e DD_API_KEY=abcd -e OTLP_HTTP_BEARER_TOKEN=example -it -v $(pwd)/collector-config:/config otel/opentelemetry-collector-contrib validate --config=/config/bearer-token-auth-datadog.yml && echo "config valid"
config valid
```

There is also an online validator available at https://www.otelbin.io/
