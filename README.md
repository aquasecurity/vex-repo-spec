# VEX Repository Specification v0.1

<!-- TOC -->
* [VEX Repository Specification v0.1](#vex-repository-specification-v01)
  * [1. Versioning](#1-versioning)
  * [2. Repository Manifest](#2-repository-manifest)
    * [2.1 Overview](#21-overview)
    * [2.2 File Location](#22-file-location)
    * [2.3 Schema](#23-schema)
    * [2.4 Example](#24-example)
    * [2.5 Field Descriptions and Usage Notes](#25-field-descriptions-and-usage-notes)
      * [Main Fields](#main-fields)
      * [Versions Subfields](#versions-subfields)
      * [Locations Subfields](#locations-subfields)
  * [3. Repository Structure](#3-repository-structure)
    * [3.1 File Structure](#31-file-structure)
    * [3.2 index.json](#32-indexjson)
    * [3.3 VEX Documents](#33-vex-documents)
    * [3.4 Usage Notes](#34-usage-notes)
      * [Directory Structure](#directory-structure)
      * [VEX Document Content](#vex-document-content)
    * [3.5 Updating the Repository](#35-updating-the-repository)
  * [4. Repository Distribution](#4-repository-distribution)
    * [4.1 Overview](#41-overview)
    * [4.2 Archive Format](#42-archive-format)
  * [5. Client Implementation Guidelines](#5-client-implementation-guidelines)
    * [5.1 Version Selection](#51-version-selection)
    * [5.2 Location Selection](#52-location-selection)
    * [5.3 Multiple Repository Support](#53-multiple-repository-support)
      * [Repository Prioritization](#repository-prioritization)
    * [5.4 Checking for Updates](#54-checking-for-updates)
    * [5.5 Efficiency Strategies](#55-efficiency-strategies)
<!-- TOC -->

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

## 1. Versioning

- The VEX (Vulnerability Exploitability eXchange) Repository Specification MUST use vX.Y versioning.
- For v1.0 and later:
    - X (major version) MUST be updated for breaking changes.
    - Y (minor version) MUST be updated for backwards-compatible changes.
- For v0.Y versions, breaking changes MAY occur with minor version updates.

When comparing versions:

- Versions MUST be compared numerically, not lexicographically.
- Major versions MUST be compared first:
    - If major versions differ, the version with the higher major version is considered newer.
    - If major versions are equal, proceed to compare minor versions.
- Minor versions MUST be compared only when major versions are equal:
    - The version with the higher minor version is considered newer.

Example comparisons:

- 1.0 < 2.0
- 1.1 < 1.2
- 1.10 > 1.2
 
## 2. Repository Manifest

### 2.1 Overview

The manifest file provides metadata about a VEX data repository.
This file MUST contain information necessary for retrieving and updating VEX data.

### 2.2 File Location

- For HTTPS: The manifest file MUST be located at `https://<domain>/.well-known/vex-repository.json`
- For GitHub repositories: `vex-repository.json` MUST be placed in the root directory of the main branch.

### 2.3 Schema

The JSON schema for the manifest file is defined [here](./vex-repository.schema.json).

### 2.4 Example

```json
{
  "name": "Example Org VEX Repository",
  "description": "VEX repository for Example Organization",
  "versions": [
    {
      "spec_version": "0.1",
      "locations": [
        {
          "url": "https://example.com/vex-hub/v0/vex-data-v0.tar.gz"
        }
      ],
      "update_interval": "24h",
      "repository_specific": {
        "location": {
          "repository_type": "db",
          "db_type": "bbolt",
          "url": "oci://ghcr.io/example.com/vex-db:0"
        }
      }
    },
    {
      "spec_version": "1.0",
      "locations": [
        {
          "url": "https://example.com/vex-hub/v1/vex-data-v1.tar.gz//subdirectory"
        },
        {
          "url": "https://example.com/vex-api/v1"
        }
      ],
      "update_interval": "1h"
    }
  ]
}
```

### 2.5 Field Descriptions and Usage Notes

#### Main Fields

| Field       | Required | Description and Usage Notes                                                                                                                                                                                                                                |
|-------------|:--------:|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name        |    ✓     | The name of the repository.                                                                                                                                                                                                                                |
| description |    ✓     | A brief description of the repository.                                                                                                                                                                                                                     |
| versions    |    ✓     | An array containing details of available versions. Each object in the array represents a version implementing a VEX Repository Specification version. Versions MUST be sorted in ascending order, from oldest to newest. See separate table for subfields. |

#### Versions Subfields

| Field               | Required | Description and Usage Notes                                                                                                               |
|---------------------|:--------:|-------------------------------------------------------------------------------------------------------------------------------------------|
| spec_version        |    ✓     | The version of the VEX Repository Specification implemented (e.g., "0.1"). Format MUST be "X.Y" as defined in [section 1](#1-versioning). |
| locations           |    ✓     | An array of objects describing VEX data locations. MUST contain at least one location object. See separate table for subfields.           |
| update_interval     |    ✓     | The recommended update check interval for this version's VEX data. Uses Go duration format (e.g., "1h", "30m", "24h").                    |
| repository_specific |    -     | Additional repository-specific information.                                                                                               |

#### Locations Subfields

| Field | Required | Description and Usage Notes                                                                                                                                                                                                                                                                                  |
|-------|:--------:|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| url   |    ✓     | A URL for the VEX data location, starting with "https://". The content adheres to the repository structure specifications in [section 3](#3-repository-structure) and [4](#4-repository-distribution). The URL may include a subdirectory specification by appending '//' followed by the subdirectory path. |

## 3. Repository Structure

### 3.1 File Structure

The repository MUST have the following structure:

```
vex-repository.<archive_extension>
[optional_subdirectory/]
├── index.json
└── pkg/
    ├── <type>/
    │   ├── <namespace>/
    │   │   ├── <name>/
    │   │   │   └── vex.json
    │   │   └── ...
    │   └── ...
    └── ...
```

Where `<archive_extension>` is one of [supported archive formats](#42-archive-format).

The `[optional_subdirectory/]` is included when the URL in the locations field ends with `//` followed by a subdirectory path.
This allows for flexibility in repository structure, particularly when using existing repository layouts such as those in GitHub repositories.

For example, if the URL is `https://github.com/org/repo/archive/refs/heads/main.tar.gz//repo-main`, the file structure would be:

```
main.tar.gz
└──repo-main/
   ├── index.json
   └── pkg/
       └── ...
```

In this case, `repo-main/` is the root directory for the VEX repository within the tar.gz file.

### 3.2 index.json

The index.json file serves as a manifest for the contents of the archive file.
It MUST be placed in the root directory of the archive or in the specified subdirectory if one is defined in the URL.
The file MUST have the following structure:

```json
{
  "updated_at": "2023-07-04T12:00:00Z",
  "packages": [
    {
      "id": "pkg:deb/debian/curl",
      "location": "pkg/deb/debian/curl/vex.json"
    },
    {
      "id": "pkg:npm/lodash",
      "location": "pkg/npm/lodash/vex.json",
      "format": "csaf"
    }
  ]
}
```

Field descriptions:

| Field               | Required | Description                                                                                                                                                 |
|---------------------|:--------:|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| updated_at          |    ✓     | Timestamp indicating when this index.json was last updated.                                                                                                 |
| packages            |    ✓     | Array of objects, each representing a package in the repository.                                                                                            |
| packages[].id       |    ✓     | Identifier of the package. Currently, only Package URL (PURL) is accepted. Version and qualifiers MUST be omitted as they are included in the VEX document. |
| packages[].location |    ✓     | Relative path to the VEX file for this package within the archive. Clients MUST use this field to locate specific package VEX files.                        |
| packages[].format   |    -     | Format of the VEX data. Either "openvex" or "csaf". If omitted, "openvex" is assumed.                                                                       |

The schema for the index file is defined [here](./index.schema.json).

### 3.3 VEX Documents

Each package's VEX information MUST be stored in a separate JSON file, following the path structure defined in the index.json file. The content of these files MUST adhere to the VEX format specification (OpenVEX or CSAF VEX) as specified in the `format` field. A single VEX document MAY include information for different versions and qualifiers of the same package.

For OpenVEX document examples, please refer to the [OpenVEX specification](https://github.com/openvex/spec/blob/main/OPENVEX-SPEC.md#example).

### 3.4 Usage Notes

#### Directory Structure
- It is RECOMMENDED to create directory structures for packages based on their PURL, excluding version and qualifiers. For example, a package with PURL "pkg:deb/debian/curl" could be stored in "pkg/deb/debian/curl/vex.json".
- For OCI packages, the `repository_url` qualifier of the PURL MAY be used to create the directory structure. For example, a package with PURL "pkg:oci/debian@sha256:3e45770a143ee5afd1ebde5a6aea6e32a71d2bt5602f5dac8025db0d9cc19f10?repository_url=docker.io/library/debian" could be stored in "pkg/oci/docker.io/library/debian/vex.json".
- The actual location of VEX files MAY be freely defined in the index.json file's `location` field, regardless of the recommended structure.
- All file paths within the archive MUST use forward slashes (/) as separators, regardless of the operating system.
- Package names in the directory structure MUST be URL-encoded if they contain special characters.

#### VEX Document Content
- A single VEX document MAY include information for different versions and qualifiers of the same package.
- When querying for a specific version or qualifier, clients MUST parse the entire VEX document to find the relevant information.

### 3.5 Updating the Repository

When updating the VEX repository:

1. Generate new or updated vex.json files for affected packages.
2. Update the index.json file to reflect any changes, including updating the `updated_at` timestamp.
3. Create a new archive with the updated contents.
4. Upload the new archive to the location specified in the manifest file (vex-repository.json).
5. Update the relevant `locations` URL in the manifest file (vex-repository.json) if necessary.

## 4. Repository Distribution

### 4.1 Overview

The VEX Repository MUST be distributed as an archive file containing the VEX data and associated metadata.
This archive MUST be referenced by the locations field in the vex-repository.json file and is the primary means of distributing VEX information.

### 4.2 Archive Format

The archive file MUST be in one of the following formats:

- `tar.gz` and `tgz`
- `tar.bz2` and `tbz2`
- `tar.xz` and `txz`
- `zip`
- `gz`
- `bz2`
- `xz`


## 5. Client Implementation Guidelines

### 5.1 Version Selection

When selecting a version from the versions array:

- Clients MUST choose a version they support based on the `spec_version` field.
- Clients MUST compare versions according to the rules defined in [section 1](#1-versioning).
- The versions array is guaranteed to be sorted from oldest to newest. Clients can use this order to efficiently select an appropriate version.
- Clients MAY select the newest version they support within the same major version, as backwards compatibility is maintained within major versions.
- If multiple major versions are supported, clients SHOULD prefer the newest major version they support, unless otherwise configured.
- If no supported version is available, clients MUST NOT use the repository and SHOULD notify the user.

### 5.2 Location Selection

When dealing with multiple locations in the `locations` array:

1. Priority Order: Clients MUST prioritize locations based on their order in the array.
   The location listed first should be attempted before moving to subsequent locations.
2. Schema Support:
    - Currently, only the "https" scheme is supported in the specification.
    - Future versions of this specification may introduce additional schemes.
    - Clients SHOULD check the URL scheme of each location and only use those with supported schemes.
3. Fallback Mechanism: If a client encounters an error with one location, it SHOULD attempt to use the next available location in the array.

### 5.3 Multiple Repository Support

Clients SHOULD be designed to support multiple VEX repositories.

#### Repository Prioritization
- Clients SHOULD implement a prioritization mechanism for repositories.
- When multiple repositories provide VEX data for the same PURL, clients SHOULD select the data based on the repository priority.
- The prioritization method SHOULD be configurable to allow users to adjust based on their specific needs and trust in different data sources.

### 5.4 Checking for Updates

Clients SHOULD use the following process to check for updates:

1. Store the timestamp of the last successful update or update check locally.
2. When considering an update, retrieve the `update_interval` from the vex-repository.json file.
3. Calculate the next update time by adding the `update_interval` to the locally stored timestamp.
4. Compare this calculated time with the current time:
    - If the current time is later than the calculated time, proceed with checking for updates:
        * Make a request to download the latest repository content.
        * If new content is available, download and process the updated repository.
        * Update the locally stored timestamp with the current time.
    - If the current time is earlier than the calculated time, continue using the cached repository content.


### 5.5 Efficiency Strategies

For efficient operation, clients MAY implement the following strategies:

1. Use HTTP ETags or Last-Modified headers when making requests to check for updates. This can help minimize unnecessary downloads when the content hasn't changed.
2. Implement a minimum interval between update checks (e.g., 1 hour) to avoid excessive network requests, especially in cases where the `update_interval` is very short.
3. Allow for manual override of update checks, enabling users to force an immediate check regardless of the calculated next update time.
