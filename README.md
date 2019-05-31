# KBase Blob store

Build status (master):
[![Build Status](https://travis-ci.org/kbase/blobstore.svg?branch=master)](https://travis-ci.org/kbase/blobstore) [![codecov](https://codecov.io/gh/kbase/blobstore/branch/master/graph/badge.svg)](https://codecov.io/gh/kbase/blobstore)

The blob store is a simple file storage service backed by an S3 compatible storage system
such as [Minio](https://min.io/). Storing a file provides a key - currently a UUID - that
allows retrival of the file when provided along with proper credentials.

The user is responsible for saving the key for use later - in the context of KBase, that means
creating a handle for the file via the [handle service](https://github.com/kbase/handle_service2)
and saving an object to the [workspace](https://github.com/kbase/workspace_deluxe) containing
that handle in an `@id handle` annotation, or saving the key directly in the workspace object
in an `@id bytestream` annotation. See the workspace documentation for details; also the
[DataFileUtil](https://github.com/kbaseapps/DataFileUtil) module can assist with these functions
in the context of KBase applications.

The API is nominally compatible with a minimal subset of the
[KBase fork of Shock's](https://github.com/kbase/Shock) API. The vast majority of functions are
not supported; only those required for the KBase codebase are included.

# API Data structures

## Node

This data structure is a subset of Shock's node data structure.

```
{
  "data": {
    "attributes": null,                               # DEPRECATED
    "created_on": "2019-05-30T23:50:19.000Z",
    "file": {
      "checksum": {
        "md5": "1b9554867d35f0d59e4705f6b2712cd1"
      },
      "name": "foo",                                  # Provided filename (see below)
      "size": 8
    },
    "format": "bar",                                  # Provided file format (see below)
    "id": "c39192c7-45b1-4fec-b196-5976d8e628f7",     # The node ID generated by the blobstore.
    "last_modified": "2019-05-30T23:50:19.000Z"
  },
  "error": null,
  "status": 200
}
```

`attributes` is deprecated, always null and is only provided for backwards compatibility reasons. 

## ACL

This data structure is a subset of Shock's ACL data structure.

```
{
  "data": {
    "delete": [User],
    "owner": User,
    "public": {
      "delete": false,
      "read:" <true if the node is publically readable, false otherwise>,
      "write": false
    },
    "read": [User...],
    "write": [User],
  },
  "error": null,
  "status": 200
}
```

`delete` and `write` ACLs are deprecated and only provided for backwards compatibility reasons.
They are always `false` for public access or contain only the node owner for standard ACLs.

A User is usually just the UUID assigned to the user by the blobstore, but when full verbosity
(see below) is requested, the User data structure is:

```
{
  "uuid": <the user's UUID assigned by the blobstore>,
  "username": <the user's KBase account name>
}
```

## Error

This data structure is identical to Shock's error data structure.
```
{
  "data": null,
  "error": [<error string>],
  "status": <http status code as an integer>
}
```

# API

Requests are authenticated by including the header `Authorization: OAuth <kbase token>` in the
request.

## Root

```
GET /
{
  "deprecationwarning": "The id and version fields are deprecated.",
  "id": "Shock",
  "servername": "blobstore",
  "servertime": <server time in epoch milliseconds>,
  "serverversion": <server version>,
  "version": "0.9.6"
}
```

The `id` and `version` fields are deprecated and present only for backwards compatibility with
Shock. The `version` field will not change.

## Upload a file / create a node
```
AUTHORIZATION REQUIRED
Content-Length header required
POST /node[?filename=<filename>&format=<file format>]
<file content>

RETURNS: a Node.
```

The `Content-Length` header must be present and accurate.

`PUT` is also supported - **but is not idempotent** - in order to ease using the `curl -T` option:

```
curl -H "Authorization: OAuth $KBASE_TOKEN" -T mylittlefile
  "http://<host>node?filename=mylittlefile&format=text"
```

## Copy a node
```
AUTHORIZATION REQUIRED
POST /node
<multipart form>

RETURNS: a Node.
```

The multipart form must have exactly one part with the name `copy_data` and the value the id of
the node to copy.

Curl example:
```
curl -H "Authorization: OAuth $KBASE_TOKEN" -F "copy_data=<node id>" http://<host>/node/
```

## Get a node
```
AUTHORIZATION OPTIONAL
GET /node/<id>

RETURNS: a Node.
```

## Get a node's ACLs
```
AUTHORIZATION OPTIONAL
GET /node/<id>/acl

RETURNS: an ACL.
```

## Download a file from a node
```
AUTHORIZATION OPTIONAL
GET /node/<id>?download[_raw]

RETURNS: the file content.
```

`?download_raw`, as opposed to `?download`, causes the `Content-Disposition` header to be
omitted. This will cause the file to be displayed in the browser rather than the browser displaying
a download dialog box.

## Set a node to be publicly readable
```
AUTHORIZATION REQUIRED
PUT /node/<id>/acl/public_read[?verbosity=full]

RETURNS: an ACL.
```

## Set a node to be privately readable
```
AUTHORIZATION REQUIRED
DELETE /node/<id>/acl/public_read[?verbosity=full]

RETURNS: an ACL.
```

## Add users to a node's read ACL

```
AUTHORIZATION REQUIRED
PUT /node/<id>/acl/read?users=<comma separated list of KBase user names>[&verbosity=full]

RETURNS: an ACL.
```

## Remove users from a node's read ACL

```
AUTHORIZATION REQUIRED
DELETE /node/<id>/acl/read?users=<comma separated list of KBase user names>[&verbosity=full]

RETURNS: an ACL.
```

## Change a node's owner
```
AUTHORIZATION REQUIRED
PUT /node/<id>/acl/owner?users=<KBase user name>[&verbosity=full]

RETURNS: an ACL.
```

The `users` parameter must contain a single user name.

# Requirements:
* go 1.12
* An S3 compatible storage system. The Blobstore is tested with Minio version 2019-05-23T00-29-34Z.
  * If Minio is used, at least version 2019-05-14T23-57-45Z is required.
* MongoDB 2.6+

# Running the server:
* Minio and MongoDB must be running.
* Copy `deploy.cfg.example` to `deploy.cfg` and adjust the values as necessary.
* In the module directory:
  * `go build app/blobstore.go`
  * `./blobstore --conf deploy.cfg`

# Developers

* Adding code
  * All code additions and updates must be made as pull requests directed at the develop branch.
    * All tests must pass and all new code must be covered by tests.
    * All new code must be documented appropriately
      * Godoc
      * General documentation if appropriate
      * Release notes
  * Exception mapping is handled in `server/errortypes.go`.
* Releases
  * The master branch is the stable branch. Releases are made from the develop branch to the master
    branch.
  * Update the version as per the semantic version rules in `app/blobstore.go`.
  * Tag the version in git and github.

## Testing

Copy `test.cfg.example` to `test.cfg` and adjust the values as necessary.

```
BLOBSTORE_TEST_CFG=[absolute path to test.cfg] go test ./...
```

Each package gets its own working directory during tests so the path to the `test.cfg` file
cannot be relative.

Mocks are generated with https://github.com/vektra/mockery v1.0.0.


# Known issues

* Providing a `Content-Type` header of `multipart/form-data; boundary=` when trying to copy a node
  will result in the `go` function that parses multipart data asserting that the http body is
  not form data, and so the body will be processed as a file upload. This is an issue in the
  `go` `mime` library.

* Providing a `Content-Length` that is larger than the http body when uploading a file will
  cause the [connection to hang forever.](https://github.com/golang/go/issues/16100#issuecomment-267594064)
  (Note that a content length > file length looks the same to the server as a hanging upload.)

# TODO
* HTTP2 support

# S3/Minio experimental server

While exploring upload speeds with various upload methods,
[this server](https://github.com/MrCreosote/minioAWSAndGoClient) was generated.

