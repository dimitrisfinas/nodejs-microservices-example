# INSTALLING OPENTELEMETRY STEP 4

# BACKENDS

In this step, we will redirect all telemetry data (traces) collected before to different backends.

Here, Jaeger and Lightstep will serve as examples.

## Add backends

Here, we will add a local and a remote backend behind the collector in order to provide a GUI and analysis tools to our traces.

- Edit the collector `config.yaml` file
  - in the `exporters:` section, add the following
```yaml
exporters:
  logging:
    loglevel: debug

  # configuring otlp to Lightstep public satellites
  otlp/lightstep:
    endpoint: ingest.lightstep.com:443
    headers:
      "lightstep-access-token": "${LIGHTSTEP_ACCESS_TOKEN}"
    tls:
      insecure: false
      insecure_skip_verify: true
#      cert_file: file.cert
#      key_file: file.key
#      ca_file:

  # configure collector to send data to jaeger
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
```

  - in the `services:` section, edit your `traces:` pipeline with the following
```yaml
pipelines:
  traces:
    receivers: [otlp]
    processors: [batch]
    exporters: [logging, jaeger, otlp/lightstep]
```
  - save you updates

- Edit the `docker-compose.yml` file
  - add the jaeger container
```yaml
jaeger:
  image: jaegertracing/all-in-one:1.30
  container_name: jaeger
  ports:
    - 14250:14250
    - 16686:16686
```

  - add an environment variable for your otel collector container
```yaml
environment:
  - LIGHTSTEP_ACCESS_TOKEN=${LIGHTSTEP_ACCESS_TOKEN}
```

- On the shell windows where you run your docker-compose command, export the value of your `LIGHTSTEP_ACCESS_TOKEN`:
```bash
export LIGHTSTEP_ACCESS_TOKEN=<YOUR_VALUE>
```

## Restart and test

- Restart you docker-compose (no need to rebuild as we didn't change any code)
```bash
docker-compose up
```

- Test again you application with http://localhost:4000 and http://localhost:4000/api/data and look at results in
  - zpages: http://127.0.0.1:55679/debug/tracez
  - jaeger: http://localhost:16686/search
  - lightstep: https://app.lightstep.com/<your_project>/explorer

- You should see your traces in both backends and also see the services diagram presenting your architecture
> Compared to Jaeger, Lightstep add additional capabilities to correlate collected traces for root cause analysis of your errors or latency


## Troubleshoot

- getting error `"error": "Permanent error: rpc error: code = Internal desc = unexpected HTTP status code received from server: 400 (Bad Request); malformed header: missing HTTP content-type"` when sending spans to Lightstep
  => check value of your LIGHTSTEP_ACCESS_TOKEN with `echo $LIGHTSTEP_ACCESS_TOKEN`
  => if not set, initialize it with `export LIGHTSTEP_ACCESS_TOKEN=<YOUR_VALUE>` then deploy again with `docker-compose up`

- getting error `host path ("/Users/dimitris.finas/git/training/nodejs-microservices-example/otel-collector/conf") not allowed as volume source, you need to reference an Azure File Share defined in the 'volumes' section` when building the application with docker-compose
  => This is because you're using an Azure context for your docker
  => GO back to default local context using `docker context use default`
