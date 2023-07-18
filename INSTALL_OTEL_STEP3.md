# INSTALLING OPENTELEMETRY STEP 3

# COLLECTOR

In this step, we will

You have 2 different options to connect between OTel SDK and the collector, either using HTTP or GRPC protocol.

Should you use GRPC or HTTP?

It depends on your tools/framework support: if possible use GRPC as it is http/v2 and more performant, but some tools or framework still don't support it well (ex: GCP Cloud run containers) or, sometimes, you may encounter gateways or firewalls that forbids GRPC.


## Option A) Using HTTP to send to OpenTelemetry collector

- Follow the steps below for each nodejs component, ie `/web` and `/service`

  - Update `package.json` file to add a new OpenTelemetry dependency for HTTP protocol support:
  ```json
  "dependencies": {
    "@opentelemetry/exporter-trace-otlp-http": "^0.28.0",
    ...
  }
  ```

  - Go to `/opentelemetry/src` and update `tracing.js` code with code below
    - import a new const, by adding line
    ```java
    const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
    ```
    > Paste this code at the beginning of the file where all const are defined

    - Replace the console exporter
      - replace `traceExporter: new opentelemetry.tracing.ConsoleSpanExporter(),`
      - by
      ```java
      traceExporter: new OTLPTraceExporter({}),
      ```
      You can put the collector URL as property here instead of using environment variables in docker-compose, simply replace `traceExporter` definition by

      ```java
      traceExporter: new OTLPTraceExporter({url: 'http://otel-collector:4318/v1/traces'}),
      ```

- Edit `docker-compose.yml` file to set nodeJS modules to export to our new OpenTelemetry Collector
  - in the `environment` section for each nodeJS container (`web` and `service`), add the variables below (an alternative would be to put these properties directly in `tracing.js` code)

```yaml
#environment:
  - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
  - OTEL_RESOURCE_ATTRIBUTES=service.name=<web or service depending on the container>
```

- Skip section "Option B)" below and go directly to "Steps common to A) and B)"


## Option B) Using GRPC to send to OpenTelemetry collector

- Follow the steps below for each nodejs component, ie `/web` and `/service`

  - Update `package.json` file to add a new OpenTelemetry dependency for GRPC protocol support:
  ```json
  "dependencies": {
    "@opentelemetry/exporter-trace-otlp-grpc": "^0.28.0",
    ...
  }
  ```

  - Go to `/opentelemetry/src` folder and update `tracing.js` code with code below
    - import a new const, by adding line
    ```java
    const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
    ```
    (at the beginning of the file where all const are defined)

    - Replace the console exporter
      - replace `traceExporter: new opentelemetry.tracing.ConsoleSpanExporter(),`
      - by
      ```java
      traceExporter: new OTLPTraceExporter({}),
      ```
      You can put the collector URL as property here instead of using environment variables in docker-compose, simply replace `traceExporter` definition by

      ```java
      traceExporter: new OTLPTraceExporter({url: 'grpc://otel-collector:4317'}),
      ```

- Edit `docker-compose.yml` file to set nodeJS modules to export to our new OpenTelemetry Collector
  - in the `environment` section for each nodeJS container (web and service), add the variables below (an alternative would be to put these properties directly in `tracing.js` code)

```yaml
#environment:
  - OTEL_EXPORTER_OTLP_ENDPOINT=grpc://otel-collector:4317
  - OTEL_RESOURCE_ATTRIBUTES=service.name=<web or service depending on the container>
```

- Go to section "Steps common to A) and B)"


## Steps common to A) and B)

- Edit `docker-compose.yml` file in root folder
  - add an OpenTelemetry collector container using lines below
  > we will use the collector from contributors community as it contains more receivers, processors and exporters

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:0.47
  container_name: otel-collector
  ports:
    # This is default port for listening to GRPC protocol
    - 4317:4317
    # This is default port for listening to HTTP protocol
    - 4318:4318
    # This is default port for zpages debugging
    - 55679:55679
  volumes:
    # Path to Otel-Collector config file
    - ./otel-collector/conf:/etc/otelcol-contrib/
    # This is old path used in container for image below or equals to v0.40.0
    #- ./otel-collector/conf:/etc/otel
```

- Create a file `config.yaml` in folder `/otel-collector/conf`

- Edit the file and put content below
```yaml
extensions:
  # This extension is used to provide a debugging zpages traces_endpoint
  # see https://github.com/open-telemetry/opentelemetry-collector/blob/main/extension/zpagesextension/README.md for more details
  zpages:
    endpoint: 0.0.0.0:55679

receivers:
  # Default receiver to collect trace and metrics received from grpc and http protocols
  # Default is port 4317 for grpc and 4318 for http
  otlp:
    protocols:
      grpc:
      http:

processors:
  # Recommended processor to proceed sending of traces or metrics in batch mode (requires less resources)
  batch:

exporters:
  # Debugging exporter directly to system log
  logging:
    loglevel: debug

service:
  pipelines:
    # Simple trace pipeline that will send all traces received from otlp protocol to the logs in batch mode
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]

  # Activate the extension that will allow us to debug this configuration from the collector web interface on port 55679 by default
  extensions: [zpages]
```


## Rebuild and test

- Rebuild and restart you docker-compose
```bash
docker-compose up --build
```

- Test standalone your collector by sending him a trace
  - from root folder, run
  ```bash
  curl -iv -H "Content-Type: application/json" http://127.0.0.1:4318/v1/traces -d @./opentelemetry/test/small_data.json
  ```
  - both in the otel-collector container console and in page http://127.0.0.1:55679/debug/tracez, you should see a new trace appearing

- Now test again your application on pages http://127.0.0.1:4000 and http://127.0.0.1:4000/api/data
  - you should also see traces appearing in your collector console
