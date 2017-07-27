---
title: Orbit Protocol Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell: cURL

toc_footers:
  - <a href='https://github.com/MarsOS'>Package Manager Protocol for MarsOS</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

should respond. It exists, so that you do not have to use our server to host
This document specifies the way a package repository for our package manager
packages because you can create your own compatible servers.

It is important to know, that the server **MUST** respond exactly as required.
The only possible exception is when you want to add additional fields to the
response. But you need to be carefull, future specifications of this protocol
might use this name, so we suggest prefixing your fields with a more or less
unique vendor prefix.

All API responses will be encoded in [JSON](json.org)

Current version of the Specification: **1**

### Base Endpoint

`GET http://example.com/orbitapi/`

where `http://example.com` will be replaced with your url as specified by
the client.


# Server Information

```shell
curl -X GET "http://example.com/orbitapi/info"
```

```json
{
  "name": "name-of-your-server",
  "email": "webmaster@your-server.com",
  "canSearch": true,
  "canMeta": true,
  "specVersion": 1
}
```
> Please note, that the name field should only contain letters, digits and dashes

`GET /orbitapi/info`

Returns a ServerSpec object as shown. This will be used to ensure
compatability between server and client.
So a client should make sure that the server is using the right protocol.

## Geo Information

```shell
curl -X GET "http://example.com/orbitapi/info/geo"
```

```json
{
  "country": "USA",
  "state": "New York",
  "city": "New York City",
  "zip": "10004"
}
```
> This is the only response, where only the `country` field is required, the others
> need to exist, but can be empty as they are only used to give information to a
> user, but will not be used by any form of algorithm.

> The country field is encoded in the three letter [ISO 3166 (ALPHA-3)](https://en.wikipedia.org/wiki/ISO_3166-1) standard.

`GET /orbitapi/info/geo`

Returns a GeoInfo object which will contain information about the servers location,
this is usefull to find servers closer to the user, which should normaly improve
connection speed and should help to spread the load on the servers.

## Server Statistics

```shell
curl -X GET "http://example.com/orbitapi/info/stats"
```

```json
{
  "uniquePackages": 42,
  "uptime": 86400,
  "availableArches": [
    "x86_64",
    "i386"
  ],
  "arches": {
    "x86_64": {
      "packages": 84,
      "uniquePackages": 42,
    },
    "i386": {
      "packages": 80,
      "uniquePackages": 40,
    }
  }
}
```
> uptime in seconds, all arches in `availableArches` **MUST** have an entry in the
> `arches` object.

`GET /orbitapi/info/stats`

Returns a ServerStats object. It will contain some basic status information about
your server, including the number of packages or the supported hardware platforms.

<aside class="notice">
A Server will contain multiple versions of a package and maybe even host the same
version for multiple architectures.
So if a package is hosted in five different variants, that this should count as
one unique and five total packages.
</aside>

# Package Search
```shell
curl -X GET "http://example.com/orbitapi/search?name=example&author=john"
```
```json
{
  "query": {
    "regex": "true",
    "name": "example",
    "author": "john",
    "spec": " ",
    "version": " "
  },
  "results": [
    "example-package",
    "libexample",
    "other-by-doe"
  ]
}
```
> The results array should contain every matching package name exactly one time

`GET /orbitapi/search`

Returns the parameters of the search as well as an array of matching package
names.

<aside class="warning">
If a package exists in different variants, it will still only be included once,
so make sure to check the package meta data for a match if you need more
information about it.
</aside>

### URL Paramters

Parameter | Default | Description
----------|---------|---------
regex | `true` | if true, all other parameters will be used as regex
name | ` ` | package name
author  | ` ` | author/Packager name
version  | ` ` | package version
arch | ` ` | package architecture


# Metadata

```shell
curl -X GET "http://example.com/orbitapi/meta?package=example-package"
```

```json
{
  "package": "example-package",
  "arches": ["x86_64", "i386"],
  "versions": {
    "x86_64": ["1.0.0", "1.0.2"],
    "i386": ["1.0.0", "1.0.2", "1.0.2-1"]
  }
}
```

`GET /orbitapi/meta`

Returns meta information about a package. Use this to check if a version you need
is available.

<aside class="notice">
Please note, that the Metadata query can be splitted to only contain curtain parts
of the PackageMeta object to save bandwith and/or parsing time.
</aside>

<aside class="warning">
To save a lot of work for the server, the package size information is not included
in the root meta query, use the size query to get this for every version.
</aside>

### Required Parameters

Parameter | Description
----------|---------
name | package name you want to query


## Available Arches

```shell
curl -X GET "http://example.com/orbitapi/meta/arches?package=example-package"
```

```json
["x86_64", "i386"]
```
> For the above example of `example-package`

`GET /orbitapi/meta/arches`

Only returns an array of all Available architecture for this package.

### Required Parameters

Parameter | Description
----------|---------
name | package name you want to query


## Available Versions

```shell
curl -X GET "http://example.com/orbitapi/meta/versions?package=example-package&arch=i386"
```

```json
["1.0.0", "1.0.2", "1.0.2-1"]
```
> For the above example of `example-package` with `arch=i386`

`GET /orbitapi/meta/versions`

Returns an array of versions for this package on a specified architecture.

### Required Parameters

Parameter | Description
----------|---------
name | package name you want to query
arch | requested architecture


## Package Size

```shell
curl -X GET "http://example.com/orbitapi/meta/versions?package=example-package&arch=i386&version=1.0.2"
```

```json
{
  "downloadSize": 640,
  "rootSize": 820
}
```
> Units in KB

`GET /orbitapi/meta/size`

Return size information for the specified package. Download size represents the
total size of the package while the root Size represents the size of the files
that will be installed.

<aside class="notice">
The total install size is slightly bigger than rootSize because the client will
create some database entries.
</aside>

### Required Parameters

Parameter | Description
----------|---------
name | package name you want to query
arch | requested architecture
version | requested version


# Package Download

You can either download the prebuild packages or download the blueprint file which
will act as a 'manual' on how to build the package from source.

Please note, that all requests to the childs of the `/orbitapi/package/` endpoint
will return raw file contents and no `JSON`.

## Download a Package

```shell
curl -X GET "http://example.com/orbitapi/package/get?package=example-package&arch=i386&version=1.0.2"
```

`GET /orbitapi/package/get`

### Required Parameters

Parameter | Description
----------|---------
name | package name you want to Download
arch | requested architecture
version | requested version

## Download a blueprint

```shell
curl -X GET "http://example.com/orbitapi/package/blueprint?package=example-package"
```

`GET /orbitapi/package/blueprint`

Parameter | Description
----------|---------
name | package name you want the blueprint of
