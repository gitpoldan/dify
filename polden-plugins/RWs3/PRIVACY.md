# Privacy Policy — RWs3

## Data this plugin processes

- **Object keys and file content** provided in tool calls (list, read, write, delete).
- **S3 / MinIO credentials**: endpoint, access key id, secret access key, bucket
  and optional prefix configured on the tool provider. These are handled by the
  Dify runtime and used only to connect to the object store you specify.

## Where data is stored

All read/write operations target **your own S3 / MinIO bucket**. The plugin keeps
no durable copy of your data on the plugin author's infrastructure.

## Third-party services

This plugin connects to **object storage endpoints you configure**:

- **Amazon S3** (when `endpoint` is empty)
- **MinIO** or other S3-compatible stores (when a custom `endpoint` is set)

Object keys, file content, and storage credentials are sent directly from the
Dify host to your configured endpoint. The plugin author does not receive,
store, or have access to this data.

## Data sharing

The plugin does not transmit your data to the plugin author or any other third
party beyond the storage endpoint you configure. There is no analytics or telemetry.

## Contact

- Author: gitpoldan
- Source: https://github.com/gitpoldan/dify/tree/main/polden-plugins/RWs3
- GitHub Issues: https://github.com/gitpoldan/dify/issues
- Email: bv2020donch@gmail.com

## Changes

If our privacy practices change, this document will be updated accordingly.
