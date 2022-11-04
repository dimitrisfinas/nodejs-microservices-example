# INSTALLING OPENTELEMETRY ALL STEPS

This is an all-in-one OpenTelemetry installation steps.

If you are a beginner to OpenTelemetry, or prefer to do it step by step, I recommend following files [INSTALL_OTEL_STEP1.md](/INSTALL_OTEL_STEP1.md) to [INSTALL_OTEL_STEP4.md](/INSTALL_OTEL_STEP4.md).


## Step 1 - Create the tracing wrapper code

- This wrapper is shared and will be used for all your nodeJS components

- In `/opentelemetry/src` folder, create a `tracing.js` file with code below

```java
// Require dependencies
const opentelemetry = require("@opentelemetry/sdk-node");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const sdk = new opentelemetry.NodeSDK({
  // sending traces to console for debugging
  //  traceExporter: new opentelemetry.tracing.ConsoleSpanExporter(),
  traceExporter: new OTLPTraceExporter({}),
  // if you want to set url in code
  //  traceExporter: new OTLPTraceExporter({url: 'http://satellite:8383'}), // For Lightstep microsatellite
  //  traceExporter: new OTLPTraceExporter({url: 'grpc://otel-collector:4317'}), // For Otel Colelctor
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start()
```


## Step 2 - Install Otel dependencies

In this step, we will add all libraries required to instrument your nodeJS application in each component

- For each nodejs component, ie `/web` and `/service`

  - Update `package.json` file to add OpenTelemetry libraries in `dependencies:{` section:
  ```json
    "@opentelemetry/api": "^1.2.0",
    "@opentelemetry/auto-instrumentations-node": "^0.34.0",
    "@opentelemetry/sdk-node": "^0.33.0",
    "@opentelemetry/exporter-trace-otlp-grpc": "^0.33.0",
  ```


## Step 3 - Add Otel wrapper to each application component

In this step, we will refer to our wrapper in each component

- For each nodejs component, ie `/web` and `/service`

  - Update the start script to add `tracing.js` as Requirement
    - edit file `nodemon.json` and replace `node ./src/index.js` by
    ```
    node --require ./src/tracing.js ./src/index.js
    ```


## Step 4 - Add your own attributes and log events (optional)

### Add custom attributes in code
- In `/src` folder of the web component, update file `index.js` file with code below:
    - Add the OpenTelemetry library by putting this at top of your code
    ```java
    // import the open telemetry api library
    const api = require('@opentelemetry/api');
    // create a tracer and name it after your package
    const tracer = api.trace.getTracer('myInstrumentation');
    ```

    - in the `main()` function, in the `app.get("/", (req, res) => {` part, add code to create custom attributes
    ```java
    // access the current span from active context
    let activeSpan = api.trace.getSpan(api.context.active());
    // add an attribute
    activeSpan.setAttribute('nbLoop', nbLoop);
    activeSpan.setAttribute('weather', weather);
    ```

### Add log events

- In the `main()` function, in the `app.get("/api/data", (req, res) => {` part, add code to create custom log events
```java
  // access the current span from active context
  let activeSpan = api.trace.getSpan(api.context.active());
  // log an event and include some structured data.
  activeSpan.addEvent('Running on http://${HOST}:${PORT}');
```

### Create spans

- Replace the `generateWork` function with code below
```java
async function generateWork(nb) {
  for (let i = 0; i < Number(nb); i++) {
    // create a new span
    // if not put as arg, current span is automatically used as parent
    // and the span you create automatically become the current one
    let span = tracer.startSpan(`Looping ${i}`);
    // log an event and include some structured data. This replace the logger to file
    span.addEvent(`*** DOING SOMETHING ${i}`);
    // wait for 50ms to simulate some work
    await sleep(50);
    // don't forget to always end the span to flush data out
    span.end();
  }
}
```


## Step 5 - Prepare deployment in your docker-compose file

- Edit `docker-compose.yml` file in root folder, edit each of your component (`web` and `service`) container deployment section

  -  add your shared `tracing.js` library in the `volumes` section using the line below

  ```yaml
  #volumes:
    - ./opentelemetry/src/tracing.js:/usr/src/app/src/tracing.js:z
  ```

  - add OpenTelemetry environment variables like the `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_RESOURCE_ATTRIBUTES` with value a list of comma separated `<key>=<value>`
  - Example:
  ```yaml
    #web:
      #environment:
       - OTEL_EXPORTER_OTLP_ENDPOINT=grpc://otel-collector:4317
       - OTEL_RESOURCE_ATTRIBUTES=service.name=<web or service>,service.version=2.0.0
  ```

  - add an OpenTelemetry collector container using lines below
  > we will use the collector from contributors community as it contains more receivers, processors and exporters

  ```yaml
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.63.1
    container_name: otel-collector
    ports:
      # This is default port for listening to GRPC protocol
      - 4317:4317
      # This is default port for listening to HTTP protocol
      - 4318:4318
      # This is default port for zpages debugging
      - 55679:55679
    environment:
      - LIGHTSTEP_ACCESS_TOKEN=${LIGHTSTEP_ACCESS_TOKEN}
    volumes:
      # Path to Otel-Collector config file
      - ./opentelemetry/conf:/etc/otelcol-contrib/
  ```

  - add the jaeger container
  ```yaml
  jaeger:
    image: jaegertracing/all-in-one:1.39
    container_name: jaeger
    ports:
      - 14250:14250
      - 16686:16686
  ```


## Step 6 - Configure your OpenTelemetry Collector

- Create a configuration for your collector named `config.yaml` in folder `/opentelemetry/conf`

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

  # configuring otlp to public satellites
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


service:
  pipelines:
    # Simple trace pipeline that will send all traces received from otlp protocol to the logs in batch mode
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging, jaeger, otlp/lightstep]

  # Activate the extension that will allow us to debug this configuration from the collector web interface on port 55679 by default
  extensions: [zpages]
```

- On the shell where you run your docker-compose command, export the value of your `LIGHTSTEP_ACCESS_TOKEN`:
```bash
export LIGHTSTEP_ACCESS_TOKEN=<YOUR_VALUE>
```


## Rebuild and test

- Rebuild your application containers with
```bash
docker-compose up --build
```

- Web page is available at: http://127.0.0.1:4000

- You can also query it with: http://127.0.0.1:4000?weather=XXX
  - take `sun`, `rain` or `snow` as weather value
    - [sun](http://127.0.0.1:4000?weather=sun) is OK response
    - [rain](http://127.0.0.1:4000?weather=rain) is OK delayed response after 1,5s
    - [snow](http://127.0.0.1:4000?weather=snow) is KO response (HTTP 500)

- An example REST API is also available: http://127.0.0.1:4001/api/data

Once deployed:

- The opentelemetry zpages for debugging are available:
  - http://127.0.0.1:55679/debug/servicez
  - http://127.0.0.1:55679/debug/tracez

- The Jaeger tracing is available: http://127.0.0.1:16686

- The Lightstep tracing is available: https://app.lightstep.com
