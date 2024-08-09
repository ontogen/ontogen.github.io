---
layout: single
title: User Guide
permalink: /docs/user-guide/
excerpt: User guide for the Ontogen CLI
last_modified_at: 2024-08-04T02:00:00+02:00
toc: true
toc_sticky: true
---
## Installation

### Homebrew
  
On macOS and Linux, the Ontogen CLI `og` can be installed using Homebrew:  
  
```sh  
$ brew install ontogen/tap/ontogen  
```  
  
### Manual installation  

Precompiled binaries for macOS, Linux and Windows are available in the [GitHub Releases page](https://github.com/ontogen/cli/releases)

For manual installation:

1. Download the appropriate binary for your system.
2. Rename the downloaded file to `og`.
3. Make the file executable: `chmod +x og`
4. Move it to a directory in your PATH, e.g., `/usr/local/bin` or `~/bin`.

### Verifying the installation  
  
After installation, you can check if the CLI is correctly installed with the following command:  
  
```sh  
$ og --version
```

Note: The first start of the command takes a bit longer than subsequent starts, as it initially installs an Erlang runtime environment on your system.


## Setting up a new repository

Since Ontogen versions datasets in a triple store, setup occurs in two steps:

1. Create a local directory for configuration.
2. Initialize a triple store for the repository.

### Initializing a directory for the repository

In the first step, use the `init` command to initialize a directory in the file system with the configuration files for a repository.

```sh
$ mkdir example
$ cd example
$ og init
Initialized empty Ontogen repository in /Users/JohnDoe/example
```

This command initializes a new Ontogen repository in the current directory. It creates an `.ontogen/` subdirectory with the necessary configuration files.

You can also specify a directory for the repository:

```sh
$ og init --directory DIRECTORY
```

This initializes the repository in the specified `DIRECTORY`.

If you want to use a custom configuration template, you can specify it with the `--template` option:

```sh
$ og init --template TEMPLATE_DIRECTORY
```

This option uses the configuration files from `TEMPLATE_DIRECTORY` as the basis for initializing the repository.

Ontogen supports various triple store adapters. You can select a specific adapter during initialization, so that a corresponding configuration is initialized (it can also be reconfigured later from the standard configuration):

```sh
$ og init --adapter STORE_ADAPTER
```

Currently, Ontogen comes with only two supported adapters (more are planned):

- `Fuseki`: [Apache Jena Fuseki](https://jena.apache.org/documentation/fuseki2/)
- `Oxigraph`: [Oxigraph Server](https://crates.io/crates/oxigraph-server), which unfortunately has some known issues that currently limit its use with Ontogen and will hopefully be resolved in the future:
    - [Datetime issue](https://github.com/oxigraph/oxigraph/issues/524): When storing XSD date specifications, trailing zeros for microseconds are truncated, resulting in a different representation than the original. This can lead to distorted results since the DateTime timestamps are also included in the hashing.
    - [Inflated CONSTRUCT query results](https://github.com/oxigraph/oxigraph/issues/525): Due to massively duplicated triples in CONSTRUCT results, the performance of the corresponding Ontogen commands that use these is severely impaired.

If no adapter is specified, the generic `og:Store` is used, where the endpoints must be configured manually.

### Setting up a repository in a triple store

In the next step, the Ontogen service is configured, particularly the triple store in which the repository of the dataset should be stored. Let's first look at the configuration files.

During initialization, a directory `.ontogen` is created, which looks like this:

```sh
$ tree .ontogen
.ontogen/
└── config
    ├── agent.bog.ttl
    ├── dataset.bog.ttl
    ├── fuseki.bog.ttl
    ├── oxigraph.bog.ttl
    ├── repository.bog.ttl
    ├── service.bog.ttl
    └── store.bog.ttl
```

In `.ontogen/config/`, there are Bog configuration files used to configure the Ontogen service and its components and provide them with metadata. Bog is a special Turtle dialect developed for Ontogen configuration. It allows for simple and flexible specification of resources and their properties. In Bog files, RDF triples are used to define settings and metadata. The main features of Bog are the use of `:this` to reference the main resources an Ontogen service and `:I` to reference the current user.

In `agent.bog.ttl`, the user is configured, which is used for commits and other occasions when no other user is specified. At least the name and email address should be specified here, but any other metadata for this FOAF and PROV agent can also be defined.

```turtle
# We omit these prefixes in the following examples.
@prefix : <https://w3id.org/bog#> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

:I a og:Agent
    ; foaf:name "John Doe"
    ; og:email <mailto:john.doe@example.com>
.
```

In `repository.bog.ttl` and `dataset.bog.ttl`, the metadata of the Ontogen repository and the dataset as a DCAT catalog are specified. All entries in these are optional. By default, the agent specified in `agent.bog.ttl` is defined as the creator by referencing `:I`.

```turtle
[ :this og:Dataset
    ; dcterms:title "Example dataset"
    ; dcterms:creator :I
    ; dcterms:publisher :I
] .
```

During initialization, configuration files for the generic triple store adapter, `store.bog.ttl`, as well as all triple stores for which Ontogen store adapters are implemented, i.e., `fuseki.bog.ttl` and `oxigraph.bog.ttl`, are generated. Unused ones can be deleted if needed or simply left unused for later. The actual selection of the adapter is done by linking in the configuration of the Ontogen service in `service.bog.ttl`, where currently only metadata can be specified apart from this store adapter selection.

```turtle
# all triple store adapters have URIs in a dedicated namespace
@prefix oga: <https://w3id.org/ontogen/store/adapter/> .

[ :this og:Service

    #################################################
    # store selection

# Select the adapter of your choice and feel free to remove the unused
# (incl. their respective config files)
#    ; og:serviceStore [ :this og:Store ]
#    ; og:serviceStore [ :this oga:Oxigraph ]
    ; og:serviceStore [ :this oga:Fuseki ]

    #################################################
    # metadata

    ; dcterms:title "Example service"
    ; dcterms:creator :I
    ; dcterms:publisher :I

] .
```

The `--adapter` option of the `init` command currently does nothing other than uncomment the corresponding line in this file.

The configuration files of the triple store adapters contain default values for all properties necessary for operation, i.e., the statements can be deleted in principle if they are not to be adjusted. However, it is recommended to leave them explicitly for documentation purposes or future adjustments. The default values of these properties are chosen to match the default values of the respective triple store. Here, for example, is the generated `fuseki.bog.ttl`:

```turtle
[ :this oga:Fuseki
    ; og:storeEndpointScheme "http"
    ; og:storeEndpointHost "localhost"
    ; og:storeEndpointPort 3030
    ; og:storeEndpointDataset "name-of-the-dataset"
] .
```

While nothing more needs to be done with Oxigraph with matching default values, with Fuseki, a dataset for the Ontogen repository still needs to be created in Fuseki, and the corresponding name must be specified via the `og:storeEndpointDataset` property.

With the generic triple store adapter configurable in `store.bog.ttl`, Ontogen can also be operated with triple stores for which no adapter implementation is yet available by specifying the various endpoints.

```turtle
[ :this og:Store
    ; og:storeQueryEndpoint <http://localhost:7878/query>
    ; og:storeUpdateEndpoint <http://localhost:7878/update>
    ; og:storeGraphStoreEndpoint <http://localhost:7878/store>
] .
```

However, it should be noted that operation for triple stores without adapter implementation is not tested, and no guarantees for correct functionality can be given, so it is at your own risk.

After completing the configuration, the repository can be created in the configured triple store using the `setup` command. This command initializes the necessary graphs and structures in the triple store based on the configuration.

```sh
$ og setup
```

## Staging changes

Unlike conventional VCS, the version-controlled data in Ontogen is kept in a triple store. This raises the question of how and where changes to this data are made, especially in light of Ontogen's SpeechAct-based commit model. A more in-depth introduction can be found in [this article](/introduction/part-2).

Ontogen uses a unique SpeechAct-based commit model. A SpeechAct represents a group of changes to the repository, similar to a commit in traditional VCS, but with additional metadata such as speaker, timestamp, and data source. This model allows for more detailed tracking of data provenance and evolution. SpeechActs are prepared through the staging system and then stored as commits in the repository.

This staging system allows users to carefully prepare and review changes before they are committed as a SpeechAct. It provides more control and transparency in the versioning process, especially when working with complex RDF data.

### Staging file

The Ontogen CLI implements a modified form of the staging concept known from Git. Instead of an implicitly held staging area, there is a single staging file with which the next SpeechAct to be committed can be incrementally prepared. By default, this file is called `STAGE.trig` and contains an RDF dataset consisting of specially named graphs for the different action types:

- `og:Addition` graph with changes that should simply be added to the repository without overwriting existing statements
- `og:Update` graph with changes that should overwrite existing statements with the same subject and predicate
- `og:Replacement` graph with changes that should replace all existing statements with the same subject
- `og:Removal` graph with statements that should be removed from the repository

In the default graph of the staging dataset, the metadata of the SpeechAct can be specified, along with further descriptions of resources that should be stored together with these in the history graph of the repository during commit. The URI used for the SpeechAct here doesn't matter, as it will be automatically determined by hashing during the commit later. Therefore, a blank node is usually used for this in the staging file. The SpeechAct is currently identified by an `rdf:type og:SpeechAct` statement. This means that currently only one SpeechAct per staging file can be typed (later versions are planned to support multiple SpeechActs per staging file, so that a corresponding `rdf:type` statement will also be allowed when referencing other SpeechActs).

### Staging commands

Although manual creation of a staging file is basically possible, this is usually done via the `stage` command, which can be given RDF files via the combinable and repeatedly applicable `--add`, `--update`, `--replace`, and `--remove` options, whose content can be added to the corresponding graph in the staging file.

```sh
$ cat data/employee38.ttl 
@prefix : <http://www.example.org/> .

:employee38 
    :firstName "John" ;
    :familyName "Smith" ;
    :jobTitle "Assistant Designer" .

$ og stage --add data/employee38.ttl
```

The content of the staging file after this command looks like this:

```turtle
@prefix : <http://www.example.org/> .
@prefix dcat: <http://www.w3.org/ns/dcat#> .
@prefix dct: <http://purl.org/dc/terms/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix prov: <http://www.w3.org/ns/prov#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

[
    a og:SpeechAct ;
    prov:endedAtTime "2024-08-04T20:57:23.301818Z"^^xsd:dateTime ;
    og:speaker <urn:uuid:7e2deed3-55c7-5bba-80c0-481fd03bb4f2>
] .

GRAPH og:Addition {
    :employee38
        :familyName "Smith" ;
        :firstName "John" ;
        :jobTitle "Assistant Designer" .
}
```

If only one action type is to be staged, one of the dedicated staging commands `add`, `update`, `replace`, and `remove` can be used, to which corresponding files can be passed directly (or in the future also via STDIN). The same data could thus also have been staged via the following command:

```sh
$ og add data/employee38.ttl
```

By default, the current time is generated for `prov:endedAtTime` and the user configured in `agent.bog.ttl` as `og:speaker` of the SpeechAct. The options `--created_at` and `--created-by` can be used to pass other values for these properties to all staging commands (alternatively, the values can also be conveniently adjusted directly in the staging file). An `og:dataSource` for the SpeechAct can also be defined via the `--source` option. Note: in addition to `prov:endedAtTime`, at least the specification of an `og:speaker` or an `og:dataSource` is necessary for hashing the URI of the SpeechAct.

```sh
$ og add data/employee38.ttl \
  --created-at 2023-01-01T12:57:54Z \
  --created-by http://example.com/JoeBloggs \
  --source     http://example.com/dataset
```

For the `stage` command, the staging file can be specified as a direct argument to use a file other than the default `STAGING.trig`. The staging commands `add`, `update`, `replace`, `remove`, as well as all commands described in the following sections that work with staging files, can specify an alternative staging file via the `--stage` option. 

```sh
$ og stage my_changes.trig --add data/employee38.ttl

$ og add data/employee38.ttl --stage my_changes.trig
```

In principle, other RDF serialization formats are also possible here, as long as they support RDF datasets, i.e., different named graphs, and an implementation for RDF.ex exists. This currently means that, in addition to TriG, N-Quads (with file extension `.nq`) and JSON-LD (with file extension `.jsonld`) are also supported.

### The `status` command

The `status` command can be used to view the currently staged changes in the Ontogen-specific diff format (which will be described in more detail in the "Investigating the history" section below). In particular, it also shows which effective changes the staged SpeechAct would lead to through a commit, i.e., which statements would be overwritten by updates or replacements, or which changes of the SpeechAct are actually superfluous because the corresponding statements already exist or, in the case of staged deletions, do not exist at all.

```sh
$ og status
speech_act 88c0102ead74bcbbeebd93caa27dfa5067b1706e301cc4a17cd60479ecf6c491
Source: <http://example.com/dataset>
Author: Joe Bloggs <joe.bloggs@example.com>
Date:   Sun Jan 01 12:57:54 2023 +0000

   <http://www.example.org/employee38>
 +     <http://www.example.org/familyName> "Smith" ;
 +     <http://www.example.org/firstName> "John" ;
 +     <http://www.example.org/jobTitle> "Assistant Designer" .
```


## Committing changes

After changes have been prepared in the staging file, they can be incorporated into the repository using the `commit` command. A commit in Ontogen represents the application of a SpeechAct to the repository and creates a new state of the dataset.

### Committing staged changes

The simplest form of the commit command uses the standard staging file and only requires a commit message via the `--message` (or `-m`) option:

```sh
$ og commit --message "Initial commit"
[(root-commit) ec8108e3f45c7c4166108a7a4d54e4fc0907cb6000ca9948fc2ce7ea6fdf907e] Initial commit
 3 insertions, 0 deletions, 0 overwrites
```

The staging file is deleted after a successful commit.

An alternative staging file can be specified as a direct argument:

```sh
$ og commit custom_stage.trig -m "Use custom stage"
```

By default, the current time is used for the commit timestamp and the user configured in `agent.bog.ttl` is used as the author of the commit. Different values can be specified for these using the `--committed-at` and `--committed-by` options.

### Committing with implicit staging

It is also possible to stage changes directly during the commit process without explicitly using the `stage` command beforehand:

```sh
$ og commit --add new_data.ttl --message "Add new employee data"
```

The options `--add`, `--update`, `--replace`, and `--remove` work here analogously to the corresponding staging commands. The associated `--created-at`, `--created-by`, and `--source` options for specifying the SpeechAct metadata are therefore also supported on the `commit` command.

## Reverting commits

Ontogen offers the ability to undo changes by reverting earlier commits. This is done with the `revert` command, which creates a new commit that undoes the changes of the specified commits (a `reset` command rolling back to a previous commit does not yet exist but is planned).

The `revert` command expects a commit range as an argument, specifying which commits should be reverted. A commit range can be specified in various ways:

- Single commit: `HEAD` or a full SHA-256 hash
- Relative commit: `HEAD~n`, where `n` is the number of steps back in the history
- Commit range: `<base>..<target>`, where `<base>` and `<target>` can each be a commit ref

Examples of valid commit ranges:

```sh
$ og revert HEAD
$ og revert HEAD~2
$ og revert HEAD~3..HEAD~1
$ og revert <full-sha>
$ og revert <full-sha>..HEAD
```

Note that in the current version of Ontogen, only full SHA-256 hashes are supported. Short forms of SHAs are planned for future versions.

The revert process creates a new commit that undoes the changes. By default, an automatically generated commit message is used that refers to the reverted commits. However, you can specify a custom message with the `--message` option:

```sh
$ og revert HEAD~2 --message "Revert changes from two commits ago"
```

As with normal commits, you can also adjust the timestamp and author of the revert commit.

## Investigating the history

### The `log` command

The `log` command in Ontogen offers a flexible way to examine the commit history of the repository and is very similar to Git's `log` command.

```sh
$ og log
201844e2d3 - Add another record (now) <John Doe john.doe@example.com>
ec8108e3f4 - Initial commit (1 minute ago) <John Doe john.doe@example.com>
```

The displayed history can be limited to a specific range of commits by specifying a commit range as described in the previous section:

```sh
$ og log HEAD~3..HEAD
```

With the `--resource` option, the history of changes to a specific resource can be displayed. This not only filters the commits that change this resource but also shows only the changes affecting this resource within these commits.

```sh
$ og log --resource http://example.com/resource1
```

By default, the commits are displayed in descending commit order. The `--date-order` option allows sorting the commits by commit timestamp, which can deviate from the actual commit order due to the `--committed-at` option during commit. The `--author-date-order` option allows sorting by the SpeechAct timestamp. The `--reverse` option can reverse the sort order, which is also possible in combination with the `--date-order` and `--author-date-order` options.

```sh
$ og log --author-date-order --reverse
```

#### Display formats

The output format of the commit and SpeechAct metadata can be selected with the `--format` option, which supports some of the formats known from Git:

- `default`: Standard format with commit hash, message, timestamp, and author
- `oneline`: Compact single-line format with short hash and message
- `short`: Short format with hash, author, and message
- `medium`: Medium format with hash, author, date, and message
- `full`: Detailed format with all commit details
- `speech_act`: Format with only the SpeechAct metadata of a commit
- `raw`: Shows the raw format that is the basis of the commit hash
- `speech_act_raw`: Shows the raw format that is the basis of the SpeechAct hash

```sh
$ og log --format full
commit 201844e2d3d3528409b6aac8d6f339e8f2e08a3f64fdd4411fda439844947682
Source:     -
Author:     John Doe <john.doe@example.com>
AuthorDate: Sun Aug 04 19:20:58 2024 +0000
Commit:     John Doe <john.doe@example.com>
CommitDate: Sun Aug 04 19:21:05 2024 +0000

Add another record

commit ec8108e3f45c7c4166108a7a4d54e4fc0907cb6000ca9948fc2ce7ea6fdf907e
Source:     <http://example.com/dataset>
Author:     Joe Bloggs <joe.bloggs@example.com>
AuthorDate: Sun Jan 01 12:57:54 2023 +0000
Commit:     John Doe <john.doe@example.com>
CommitDate: Sun Aug 04 19:20:39 2024 +0000

Initial commit
```

The changes themselves can be displayed in different forms in the history through various flags, also in combination.

The `--resource-only` flag shows only the IRIs of the changed resources:

```sh
$ og log --resource-only
201844e2d3 - Add another record (7 minutes ago) <John Doe john.doe@example.com>
http://www.example.org/employee39

ec8108e3f4 - Initial commit (8 minutes ago) <John Doe john.doe@example.com>
http://www.example.org/employee38
```

The `--stat` and `--shortstat` flags provide statistical information about the changes:

- `--stat`: Shows detailed statistics for each changed resource
- `--shortstat`: Outputs a short statistic with the total number of changed resources and the number of insertions and deletions

```sh
$ og log --stat
201844e2d3 - Add another record (8 minutes ago) <John Doe john.doe@example.com>
 http://www.example.org/employee39 | 4 ++++
 1 resources changed, 4 insertions(+)

ec8108e3f4 - Initial commit (8 minutes ago) <John Doe john.doe@example.com>
 http://www.example.org/employee38 | 3 +++
 1 resources changed, 3 insertions(+)
```

Finally, the `--changes` flag can show the full changes of the commit and SpeechAct in an Ontogen-specific diff format:

```sh
$ og log --changes
201844e2d3 - Add another record (9 minutes ago) <John Doe john.doe@example.com>
   <http://www.example.org/employee39>
 +     <http://www.example.org/familyName> "Miller" ;
 +     <http://www.example.org/firstName> "Sarah" ;
 +     <http://www.example.org/jobTitle> "CIO" ;
 +     <http://www.example.org/maritalStatus> <http://www.example.org/Married> .


ec8108e3f4 - Initial commit (9 minutes ago) <John Doe john.doe@example.com>
   <http://www.example.org/employee38>
 +     <http://www.example.org/familyName> "Smith" ;
 +     <http://www.example.org/firstName> "John" ;
 +     <http://www.example.org/jobTitle> "Assistant Designer" .
```

The diff format uses the following symbols and colors:

- `+` (green): New triples added to the repository as part of an `add` action
- `±` (cyan): New or updated triples added as part of an `update` action
- `⨦` (light cyan): New triples added as part of a `replace` action
- `-` (red): Triples removed from the repository as part of a `remove` action
- `~` (light red): Existing triples overwritten by `update` or `replace` actions
- `#` (white, strikethrough): This symbol is used in combination with the other change symbols and marks a change that was intended in the SpeechAct but is ignored because it is already implemented in the repository.

Here's an example using all symbols:

```
commit 758252aa0f2d87da156d9be90e45f7325ee165822778bf7bc0dff4cba8706868
Author: Jane Doe <jane.doe@example.com>
Date:   Fri Aug 4 10:00:00 2023 +0000

Update employee information

   <http://example.com/employee/1>
 +     <http://example.com/jobTitle> "Senior Developer" .
 -     <http://example.com/jobTitle> "Developer" .
#±     <http://example.com/name> "John Doe" .

   <http://example.com/employee/2>
 ±     <http://example.com/department> "Marketing" .
 ~     <http://example.com/department> "Sales" .
       <http://example.com/startDate> "2022-01-01" .

   <http://example.com/employee/3>
 ⨦     <http://example.com/position> "Manager" .
 ~     <http://example.com/position> "Team Lead" .
 ~     <http://example.com/salary> "75000" .
```

Explanation of the changes:

1. For `employee/1`:
    - The job title was changed from "Developer" to "Senior Developer" (`+` for `insert`, `-` for `remove`).
    - The `update` of the name was intended but ignored as it was already implemented (`#±`).
2. For `employee/2`:
    - The department was updated from "Sales" to "Marketing" (`±` for `update`, `~` for displaced old value).
    - The start date remained unchanged (no symbol).
3. For `employee/3`:
    - The position was changed from "Team Lead" to "Manager" by a replace (`⨦` for `replace`, `~` for displaced old value).
    - The salary was also removed by the `replace` (`~`).

The `--commit-changes` and `--speech-changes` flags offer somewhat more specific levels of detail for the changes compared to the `--changes` format, which displays all changes from the SpeechAct and the commit combined. They show either only the effective changes of the respective commits or their changes intended in the SpeechAct.

### The `changeset` command

The `changeset` command in Ontogen is closely related to the `log` command but provides a consolidated view of changes over a specific commit range. It combines all changes in the specified range into a single changeset, filtering out mutually canceling changes.

```sh
$ og changeset HEAD~2..HEAD
  <http://www.example.org/employee38>
+     <http://www.example.org/familyName> "Smith" ;
+     <http://www.example.org/firstName> "John" ;
+     <http://www.example.org/jobTitle> "Assistant Designer" .

  <http://www.example.org/employee39>
+     <http://www.example.org/familyName> "Miller" ;
+     <http://www.example.org/firstName> "Sarah" ;
+     <http://www.example.org/jobTitle> "CIO" ;
+     <http://www.example.org/maritalStatus> <http://www.example.org/Married> .
```

The `changeset` command supports many of the same options as the `log` command:

- `--changes`: Shows the actual changes in detail (default if no other formats are selected)
- `--stat`: Generates a diffstat
- `--shortstat`: Shows only a summary of the changes
- `--resource-only`: Shows only the IRIs of the changed resources
- `--resource IRI`: Filters changes for a specific resource