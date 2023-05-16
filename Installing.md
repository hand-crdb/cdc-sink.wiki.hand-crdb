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
docker pull cockroachdb/cdc-sink:master
docker run cockroachdb/cdc-sink:master version
```

## Automated builds

Binaries for a number of platforms are published as artifacts of [successful builds](https://github.com/cockroachdb/cdc-sink/actions/workflows/binaries.yaml?query=is%3Asuccess+branch%3Amaster).

| Platform | Architecture | Link |
| -------- | ------------ | ---- |
| Darwin   | AMD64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-darwin-amd64-master.tgz)
| Darwin   | ARM64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-darwin-arm64-master.tgz)
| Linux    | AMD64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-linux-amd64-master.tgz)
| Linux    | ARM64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-linux-arm64-master.tgz)
| Windows  | AMD64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-windows-amd64-master.tgz)
| Windows  | ARM64 | [Download](https://storage.googleapis.com/cdc-sink-binaries/cdc-sink-windows-arm64-master.tgz)
