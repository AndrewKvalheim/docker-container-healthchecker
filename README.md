# docker-container-healthchecker

Runs healthchecks against local docker containers

## Requirements

- golang 1.19+

## Usage

After creating an app.json file, execute the healthchecker like so:

```shell
# docker-container-healthchecker check $CONTAINER_ID_OR_NAME
docker-container-healthchecker check cb0ce984f2aa
```

By default, the checks specified for the `web` process type are executed. If the process-type has no checks specified, a default `uptime` container check of 10 seconds is performed.

### Check types

#### `command`

Runs a command within the specified container. If the command exits non-zero, the output is printed and the check is considered failed.

As the `command` type is run within the container environment, it can be used to perform dynamic checks using environment variables exposed to the container. For example, it may be used to simulate content checks on http endpoints like so:

```shell
#!/usr/bin/env bash

OUTPUT="$(curl http://localhost:$PORT/some-file.js)"
if ! grep jQuery <<< "$OUTPUT"; then
  echo "Expected in output: jQuery"
  echo "Output: $output"
  exit 1
fi
```

`command` checks respect the `attempts`, `timeout` and `wait` properties.

If the `command` type is in use, the `path` and `uptime` healthcheck properties must be empty.

#### `path`

Executes an http request against the container at the specified `path`. The container IP address is fetched from the `bridge` network and the port is default to `5000`, though both settings can be overridden by the `--network` and `--port` flags, respectively.

HTTP `path` checks respect the `attempts`, `timeout` and `wait` properties.

Custom headers may be specified for http `path` requests by utilizing the `--header` flag like so:

```shell
docker-container-healthchecker check cb0ce984f2aa --header 'X-Forwarded-Proto: https'
```

To further customize the type of request performed, please see the `command` check type.

If the `path` type is in use, the `command` and `uptime` healthcheck properties must be empty.

#### `uptime`

Ensures the container is up for at least `uptime` in seconds. If a container has restarted at all during that time, it is treated as an unhealthy container.

`uptime` checks _do not_ respect the `attempts`, `timeout` and `wait` properties.

If the `uptime` type is in use, the `command` and `path` healthcheck properties must be empty.

### File Format

Healthchecks are defined within a json file and have the following properties (the respective scheduler properties are also noted for comparison):

| field        | default                          | description                                    | scheduler aliases (kubernetes, nomad) |
|--------------|----------------------------------|------------------------------------------------|---------------------------------------|
| attempts     | default: `3`, unit: seconds      | Number of retry attempts to perform on failure | `nomad=check_restart.limit` |
| command      | default: `""` (empty string)     | Command to execute within container            | `kubernetes=exec.Command` `nomad=command args` |
| initialDelay | default: `0`, unit: seconds      | Number of seconds to wait after a container has started before triggering the healthcheck | `kubernetes=initialDelaySeconds` `nomad=check_restart.grace` |
| name         | default: `""` (autogenerated)    | The name of the healthcheck. If unspecified, it will be autogenerated from the rest of the healthcheck information. | `nomad=name` |
| path         | default: `/` (for http checks)   | An http path to check. | `kubernetes=httpGet.path` `nomad=path` |
| timeout      | default: `5`, unit: seconds      | Number of seconds to wait before timing out a healthcheck. | `kubernetes=timeoutSeconds` `nomad=timeout` |
| type         | default: `""` (none)             | Type of the healthcheck. Options: `liveness`, `readiness`, `startup` | |
| uptime       | default: `""` (none)             | Amount of time the container must be alive before the container is considered healthy. Any restarts will cause this to check to fail, and this check does not respect retries. | |
| wait         | default: `5`, unit: seconds      | Number of seconds to wait between healthcheck attempts. | `kubernetes=periodSeconds` `nomad=interval` |

> Any extra properties are ignored

Healthchecks are specified within an `app.json` file grouped in a per process-type basis. One process type can have one or more healthchecks of various types.

```json
{
  "healthchecks": {
    "web": [
        {
            "type":        "startup",
            "name":        "web check",
            "description": "Checking if the app responds to the /health/ready endpoint",
            "path":        "/health/ready",
            "attempts": 3
        }
    ]
}
```

An example `app.json` is located in the root of this repository.
