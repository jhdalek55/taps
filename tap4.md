* TAP: 4
* Title: The map file
* Version: 1
* Last-Modified: 14-Dec-2016
* Author: Trishank Karthik Kuppusamy, Sebastien Awwad, Evan Cordell,
          Vladimir Diaz, Jake Moshenko, Justin Cappos
* Status: Draft
* Content-Type: text/markdown
* Requires: [TAP 5](tap5.md)
* Created: 09-Sep-2016

# Abstract

TAP 4 describes how users may specify that a certain repository should be used
for some targets, while other repositories should be used for other targets.
In other words, this TAP allows users to _map_ target names to repositories in a
manner similar to how targets with specific names can be delegated to different roles.
This allows users to say that a target with a specific type of name (such as 
django\* or \*.tar.gz) may be found on a specific repository.  Each repository
has its own root of trust (root role, etc.) so a compromise of one repository
does not impact other repositories.  This TAP also discusses how the AND relation can
be extended to multiple repositories to have multiple different repositories with 
separate roots of trust need to sign off on the same target before installation.

# Motivation

TAP 4 has been motivated by the following use cases.

## Use case 1: obtaining different targets from different repositories

Itmay be desirable to use the same instance of TUF to download and verify different
targets hosted on different repositories (for example, some Python packages from
their maintainers, while getting other Python packages from PyPI).  In this way,
one can securely get all Python packages regardless of where they are hosted.
There are significant advantages in using the same instance of TUF to manage
metadata across different repositories, including benefiting from security
updates, and performance enhancements.

## Use case 2: hiding sensitive metadata and targets

Enterprise users may not wish to upload some metadata and targets to a public
repository, because doing so may reveal some sensitive / proprietary
information.
Therefore, these users may use a private repository to host these sensitive
metadata and targets, and hide them from public view.
In order to use both the private and public repositories, TUF clients need to be
somehow informed to search for some targets on the private repository, and all
other targets on the public repository.

## Use case 3: improving compromise-resilience

To improve compromise-resilience, a user wish to have multiple repositories, each
with a different root of trust, to sign targets.
The effect is similar to the AND relation used in
[multi-role delegations](tap3.md).
However, in multi-role delegations, multiple roles would share the _same_ root
of trust, even though they must sign the same hashes and length of targets.
The problem is that, if attackers have compromised a common ancestor of these
multiple roles (e.g., the top-level targets role or root role), then the security 
benefits of using multi-role delegations are lost.
The difference in this use case is that multiple roles with _different_
roots of trust must sign the same hashes and length of desired targets.
This is done so that the compromise of even the root role of a single repository
is still insufficient to execute arbitrary software attacks.

# Rationale

TUF would benefit greatly from the ability to handle the above use cases,
requested by its adopters.
This TAP does not change existing security guarantees, allows a great deal of
flexibility, and should require only modest effort for support from an existing
implementation.

# Specification

We introduce a mandatory top-level metadata file called `map.json`.
This _map file_ comes into play when a TUF client requests targets.

Using a scheme similar to targets delegations within a repository, targets may
be mapped to one or more repositories in this file.

A client will keep all metadata for each repository in a separate directory.
Where these directories are kept is up to the client.

## The map file

The map file maps targets to repositories.
This file is not intended to be automatically available / refreshed from a repository.
The map file is either constructed by the user using the TUF command-line tools, or
distributed by an out-of-band bootstrap process.  The file is kept on the client
and is only modified by a user who is trusted to configure the TUF instance.

The map file contains a dictionary that holds two keys, "repositories" and
"mapping."

The value of the "repositories" key is another dictionary.
Each key in this dictionary is a _repository name_, and its value is a list of
URLs.
The repository name also corresponds to the name of the local directory on the
TUF client where metadata files would be cached.
Crucially, there is where the [root metadata file](tap5.md) for a repository
would be found.
The list of URLs specifies where TUF clients may download metadata and target
files.
Each URL points to a [directory containing metadata and target files](#metadata-and-targets-layout-on-repositories).
Each URL may be a [file URI](https://en.wikipedia.org/wiki/File_URI_scheme),
which means that these files shall be updated from a local directory on disk
instead of a remote server.
_If this list is empty, then it means that no metadata or target file for this
repository shall be updated at all._
These files would be updated following the steps detailed in
[this section](#downloading-metadata-and-target-files).

The value of the "mapping" key is a list.
Every member in this list is a dictionary with at least two keys:

* "paths" specifies a list of target paths of patterns. A desired target must
match a pattern in this list for this mapping to be consulted.
* "repositories" specifies a list of one or more repository names.
* "terminating" is a Boolean attribute indicating whether or not
  this mapping terminates [backtracking](#interpreting-the-map-file).

The following is an example of a map file:

```javascript
{
  // For each repository, its key name is the directory where files, including
  // the root metadata file, are cached, and its value is a list of URLs where
  // files may be downloaded.
  "repositories": {
    "Django": ["https://djangoproject.com/"],
    "PyPI":   ["https://pypi.python.org/"]
  },
  // For each set of targets, specify a list of repositories where files may be
  // downloaded.
  "mapping": [
    {
      // Map any target matching *Django* to both Django and PyPI.
      "paths":        ["*django*"],
      "repositories": ["Django", "PyPI"],
      // In this case, the "terminating" attribute is set to false.
      "terminating":  false,
      // Therefore, if this mapping has not signed for a *django* target,
      // the following mapping will be consulted.
    },
    {
      // Map all other targets only to PyPI.
      "paths":        ["*"],
      "repositories": ["PyPI"]
    }
  ]
}
```

## Metadata and targets layout on repositories

In order for TUF clients to download metadata and target files in a uniform way
across repositories, a repository would organize the files as follows.

All metadata files would be stored under the "metadata" directory.
This directory would contain at least four files, one for each top-level role:
root.json, timestamp.json, snapshot.json, and targets.json.
This directory may also contain a "delegations" subdirectory, which would hold
only the metadata files for all delegated targets roles.
Separating the metadata files for the top-level roles from all delegated
targets roles prevents a delegated targets role from accidentally overwriting
the metadata file for a top-level role.
It is up to the repository to enforce that every delegated targets role uses a
unique name.

All targets files would be stored under the "targets" directory.
Beyond this, the repository may organize target files into any hierarchy it
requires.

The following directory layout may apply to the PyPI repository from the example
map file:

```
-metadata
-└── root.json
-└── timestamp.json
-└── snapshot.json
-└── targets.json            // signed by the top-level targets role
-    └── delegations
-        ├── Django.json     // signed by the Django delegated targets role
-        ├── Flask.json
-        ├── NumPy.json
-targets
```

## Metadata and targets layout on clients

On a TUF client, all metadata files would be stored under the "metadata"
directory.
This directory would contain the map file, as well as a subdirectory for every
repository specified in the map file.
Each repository metadata subdirectory would use the repository name.
In turn, it would contain two subdirectories: "previous" for the previous set of
metadata files, and "current" for the current set.

All targets files would be stored under the "targets" directory.
Targets downloaded from any repository would be written to this directory.

The following directory layout would apply to the example map file:

```
-metadata
-└── map.json           // the map file
-└── Django             // repository name
-    └── current
-        ├── root.json  // minimum requirement
-        └── timestamp.json
-        ├── snapshot.json
-        ├── targets.json
-        └── ...        // see layout of targets delegations on repository
-    └── previous
-└── PyPI/current
-└── PyPI/previous
-targets
```

## Downloading metadata and target files

A TUF client would perform the following six steps while searching for a target
on a repository.

First, the client loads the latest downloaded [root metadata file](tap5.md), and
ensures that: (1) that it has been signed by a threshold of keys, and (2) it has
not expired.
Next, the client tries to update the root metadata file.
Let M denote the list of URLs associated with this repository in the map file,
and R denote the list of URLs associated with this top-level role (in this case,
the root role) in the root metadata file.
There are four cases:

1. If R is empty, then this metadata file shall not be updated.
2. If R is not empty, then this metadata file shall be downloaded in order from
each URL in R until it is found. If the file could not be found or verified
using all URLs, then report that it is missing.
3. If R has been omitted, and M is empty, then this metadata file shall not be
updated.
4. If R has been omitted, and M is not empty, then this metadata file shall be
downloaded in order from each URL in M until it is found. If the file could not
be found or verified using all URLs, then report that it is missing.

Second, the client uses similar steps to update the timestamp metadata file.

Third, the client uses similar steps to update the snapshot metadata file.

Fourth, the client uses similar steps to update the top-level targets metadata
file.
If R is not empty, then the client should be careful in interpreting the entries
of the snapshot metadata file.
Suppose that R is
["https://pypi.python.org/metadata/delegations/Django.json", "http://example.com/path/to/foo.json"].
First, the client would look up the version number for "Django.json", instead of
"targets.json" (for the original top-level targets role), in the
snapshot metadata.
Then, the client would try to find the desired target using the "Django.json"
role.
If the target could not be found or verified, then the client would try to find
the target using the "foo.json" role, being careful to first look up the version
number for "foo.json" in the snapshot metadata.

Fifth, the client uses only M to update delegated targets metadata files.
Each file is downloaded in order from each URL in M until it is found.
If the file could not be found or verified using all URLs, then report that it
is missing.

Sixth, the client uses a step similar to the previous step to download all
target files.

## Interpreting the map file

Every mapping in the map file shall be interpreted as follows.
Proceeding down the list of mappings in order, if a desired target matches a
targets path pattern in the "paths" attribute, then download and verify metadata
from every repository specified in the "repositories" attribute.
Ensure that the targets metadata, specifically length and hashes about the
target, matches across repositories.
Custom targets metadata is exempted from this requirement.
If the targets metadata matches across repositories, return this metadata.
Otherwise, report the mismatch to the user.
If all repositories in the current mapping have not signed any metadata
about the target, then take one of the following two actions.
If the "terminating" attribute is set to true, report that there is no metadata
about the target.
Otherwise, proceed to similarly interpret the next mapping.

# Security Analysis

This TAP allows users to map targets to one or more repositories.
However, it does not change the way TUF verifies metadata for any one
repository.
Each repository continues to be treated as it was previously, with TUF
performing the necessary verification of the repository metadata.

In order to avoid accidental denial-of-service attacks when multiple
repositories sign the same targets, these repositories should coordinate to sign
the same targets metadata (i.e., length and hashes).

# Backwards Compatibility

This specification is not backwards-compatible because it requires:

1. TUF clients to support additional, optional fields in the [root metadata file](tap5.md).
2. A repository to use a [specific filesystem layout](#metadata-and-targets-layout-on-repositories).
3. A client to use a [map file](#the-map-file).
4. A client to use a [specific filesystem layout](#metadata-and-targets-layout-on-clients).
5. A client to [download metadata and target files from a repository in a specific manner](#downloading-metadata-and-target-files).

# Augmented Reference Implementation

[TODO: Point to a branch containing implementation of TAP 4.]

# Copyright

This document has been placed in the public domain.
