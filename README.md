# Osysharp.S3 — your app's files in S3 or Cloudflare R2

An `app.BlobStore` for S3-compatible object storage — AWS S3 and Cloudflare R2 — written in Osy# and signed with
AWS Signature Version 4.

## Use case

An app that keeps uploads — invoices, photos, contracts, exports — and should not hold large files in its own
database. Register a bucket and every file too large to live in its row is written there, and read, replaced and
deleted there. Choose it when the bytes must sit in your own cloud account for retention or compliance, when you
already run storage in AWS or Cloudflare, or when you want R2's free egress for files people download often.

It has no user interface, so there is nothing to screenshot: your app's file features keep working exactly as they
did, with the bytes somewhere else.

## Install and use

```osy
// app.osy
app Shop {
  model "**/*.osy";
  use Osysharp.S3 {
    egress "*.amazonaws.com";
    egress "*.r2.cloudflarestorage.com";
    secret "S3AccessKeyId";
    secret "S3SecretAccessKey";
  }
}
```
```osy
using Osysharp.S3;

app.Secrets = [ new Secret("S3AccessKeyId"), new Secret("S3SecretAccessKey") ];

class Files : S3Store {
  public Files() { Endpoint = "https://s3.eu-north-1.amazonaws.com"; Region = "eu-north-1"; Bucket = "shop-files"; }
}

app.BlobStore = Files;
```
```console
osy secret set S3AccessKeyId
osy secret set S3SecretAccessKey
```

That is the whole surface. Every file the app keeps that is too large to live in its own row goes to the bucket;
reading it, replacing it and deleting it go there too. For Cloudflare R2 the endpoint is
`https://<account-id>.r2.cloudflarestorage.com` and the region is `auto`.

## What it ships

| file | what it is |
|---|---|
| `model/S3Store.osy` | `class S3Store : IBlobStore` — Put, Get, Exists, Delete as signed `Http.*` requests |
| `model/SigV4.osy` | `SigV4Authorization(…)` — the AWS Signature Version 4 `Authorization` header for one request |

Requests use path-style addressing (`<endpoint>/<bucket>/<key>`), which both services accept for any bucket name.
`Exists` asks for one byte rather than the whole object. A request the bucket refuses throws
`ExternalServiceException`, so a caller can catch "the bucket said no" apart from a bug in its own code.

## Build your own on top

**A different bucket or account** needs no change to the package: derive from `S3Store` once per bucket and register
the one your app uses.

**Another S3-compatible service** — MinIO, Wasabi, Backblaze B2 — needs a fork, and for a reason worth knowing: a
package can reach only the hosts its own manifest declares *and* your app grants, so this package cannot be pointed at
a host it never asked for. Fork this repository into your GitHub account or organisation, rename the package to match
(`package Acme.S3` lives at `github.com/acme/s3`), add the host to its manifest, and `osy publish`:

```osy
package Acme.S3 {
  version "1.0.0";
  model   "model/*.osy";
  use Osysharp.Http;
  use Osysharp.Storage;

  egress "s3.eu-central-1.wasabisys.com";
  secret "S3AccessKeyId";
  secret "S3SecretAccessKey";
}
```

**A different kind of store** is an `IBlobStore` of your own, and this package is the worked example of one: a class
implementing the Storage package's interface, signing with `Crypto.*`, sending with `Http.*`, and declaring the hosts
and secrets it needs. Nothing in it is privileged — a third party could have written it exactly as it is. The signer
is checked against the signature AWS publishes for its own GET Object example, so a fork has a known-good vector to
test against.
