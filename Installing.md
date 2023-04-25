# Installing cdc-sink

## From source
cdc-sink is a golang binary and can be installed from source:

```
go install github.com/cockroachdb/cdc-sink@latest
$HOME/go/bin/cdc-sink version
```

## Docker image

cdc-sink is packaged as a ready-to run container, available from DockerHub.

```
docker pull cockroachdb/cdc-sink

```

## Automated builds

Binaries for a number of platforms are published as artifacts of [successful builds](https://github.com/cockroachdb/cdc-sink/actions/workflows/golang.yaml?query=branch%3Amaster+is%3Asuccess).  Navigate to a recent build and scroll down the page until you see the Artifacts section:

![Artifacts listing screenshot](https://user-images.githubusercontent.com/1158548/234373883-050f95f9-a3bc-4d28-b728-11caaba41417.png)
