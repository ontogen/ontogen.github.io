---
title: Ontogen Configuration with Bog
date: 2024-08-14 9:00:00 +0200
categories:
  - introduction
  - blog
permalink: /introduction/part-4
header:
  image: /assets/images/ontogen-introduction-article-4.png
mermaid: true
---
In the [previous article](/introduction/part-3), we explored the interpretation of `dcat:Catalog` and `dcat:DataService` concepts within the framework of Ontogen as a Data Control Management (DCM) system. Now, let's turn our attention to another crucial aspect: the configuration of Ontogen components.

We've seen that an Ontogen service represents a concrete instantiation of a repository on a user's machine, consisting of the repository itself and its associated triple store. Configuring these components presents us with several challenges, particularly when it comes to naming and identifying resources.

In this article, we'll delve into Ontogen's configuration system. We'll explore Bog, a specialized configuration language developed for Ontogen. Bog addresses some of the fundamental challenges in configuring RDF-based systems and offers innovative solutions for resource naming and identification.

## Bog

"Naming is hard." This is especially true in the RDF world with its URIs, and even more so under Linked Data rules, where the preference for HTTP URIs further complicates the issue by introducing DNS as a social authority, thus bringing societal and political aspects into play.

To be clear: URIs as a central component of the RDF model are one of the reasons that make it so powerful. Nevertheless, the additional complexity this creates for the naming problem is, in my opinion, one of the reasons why it still causes reservations compared to other data models. Particularly when combined with the lack of automated solutions for this problem, it makes traditional, relational data model proponents shake their heads and happily continue to generate records (or models of their ORMs) with automatically generated primary keys.

Bog aims to automate the problem of resource naming (aka URI minting) for a specific class of resources, initially and specifically for the resources of an Ontogen service that need to be minted as part of the configuration.

To this end, Bog introduces some special RDF properties and resources with particular semantics that are processed by a Bog interpreter. The most fundamental of these properties, to which all others are ultimately reduced, is `bog:ref`. With `bog:ref`, a resource initially identified by a blank node can be given a locally valid name.

Example:

```turtle
# we omit this prefix in the following code snippets
@prefix : <https://w3id.org/bog#> .

[ :ref "this-service-instance" ; a og:Service ] .
```

When first interpreted by the Bog interpreter, this is interpreted as minting a new, locally named resource. A random salt is then stored in a file with the specified name. The interpreter then replaces the blank node with a generated UUIDv5 URI. For each subsequent interpretation, loading the salt reproduces exactly the same UUIDv5 URI, allowing for consistent translation to the same graph.

Note: the name itself is not part of this hash, meaning that the locally used `:ref` name can be changed at any time by renaming the file and all names in the local configuration files, yet still leads to the same UUID URIs.

To prevent accidental changes, Bog throws an error if the salt file is not present. This is only allowed during initial minting.

Bog offers another solution to avoid name changes: the indexical `bog:this` property. The concept of [indexicality](https://en.wikipedia.org/wiki/Indexicality) comes from the philosophy of language and refers to linguistic expressions whose meaning depends on the context of their utterance. In Bog, this concept is applied to the referencing of resources. It allows referencing individual instances of classes that represent a unique individual relative to this instance in the context of the executing instance. Consequently, these quasi-relative singletons can also be referenced directly via their class in the context of the executing instance.

Example: In the context of execution on an Ontogen instance, there is exactly one distinguished individual of the class `og:Service` that represents this very instance as an `og:Service`. Thus, in Bog, it can also be referenced using the `bog:this` property as follows:

```turtle
[ :this og:Service ] .
```

This is interpreted by the Bog interpreter as follows:

```turtle
[ :ref "service" ; a og:Service ] .
```

In fact, almost all elements of an `og:Service` are unique singletons relative to this instance and thus indexically referenceable via `bog:this` and the corresponding class name.

Additionally, for referencing the user of the application, Bog provides the indexical `bog:I` resource, which can be used to reference the user of the system:

```turtle
:I ex:p ex:O .
```

This is interpreted as:

```turtle
[ :this og:Agent ; ex:p ex:O ] .
```

Through local interpretation, Bog thus allows us to reference the various components of an Ontogen service without having to name them, but instead getting automatically generated, stable URIs for these resources.

It is planned to spin off Bog as its own project with expanded functionality in the next version of Ontogen. In particular, it should be possible to give the resources managed with Bog proper Linked Data URIs at a later point in time if needed.

In future versions, there are plans to apply the Bog-based minting process to resources of the version-controlled dataset itself, and to include this minting process as a speech act within the version history. The secret salts possessed by the creator of such resources would then give them cryptographic control over the further use and modification possibilities of these resources.

These developments aim to enhance Bog's capabilities and integrate it more deeply with Ontogen's versioning system, potentially offering new levels of security and control over resource management.


## Bog-based configuration of Ontogen services and repositories

With this mechanism, we have a method for generating persistent and reproducible UUID URIs. Now, let's look at how the concrete specification of Ontogen services and their components (store, repository, etc.) works in practice with the Bog interpreter.

For a project created with the Ontogen CLI, the directory structure typically looks like this:

```
my_dataset/  
├── .ontogen  
│    ├── .salts  
│    │   ├── agent.salt  
│    │   ├── dataset.salt  
│    │   ├── fuseki.salt  
│    │   ├── history.salt  
│    │   ├── oxigraph.salt  
│    │   ├── repository.salt  
│    │   ├── service.salt  
│    │   └── store.salt  
│    └── config  
│        ├── agent.bog.ttl  
│        ├── dataset.bog.ttl  
│        ├── fuseki.bog.ttl  
│        ├── oxigraph.bog.ttl  
│        ├── repository.bog.ttl  
│        ├── service.bog.ttl  
│        └── store.bog.ttl  
│── ...
```

Each of the `.ontogen/config/*.bog.ttl` files configures exactly one singleton instance of the respective class of an Ontogen service instance, to which the `bog:this` in the respective Bog Turtle file refers. The salt for generating the URI is persisted in the respective `.ontogen/.salts/*.salt` file of the same name.

Rather than using a single large configuration file, a modular approach is followed here, where each component is configured in its own Bog Turtle file. This approach is particularly advisable because the component descriptions should not only contain the configuration of the respective component (for which there are not many configuration options at this point in this first version), but also the general description of the respective resource belongs here, i.e., the description of the `og:Service` as a `dcat:DataService`, the description of the `og:Dataset` as a `dcat:Catalog` or the `og:Agent` as a `foaf:Agent`, etc. A complete description of all these resources in one file would quickly become very large and confusing.

Here's an example of the configuration of the `og:Dataset` with a complete DCAT description:

```turtle
@prefix : <https://w3id.org/bog#> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix dcat: <http://www.w3.org/ns/dcat#> .
@prefix dcterms: <http://purl.org/dc/terms/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

[ :this og:Dataset
  ; a dcat:Dataset
  ; dcterms:title "My Dataset"@en
  ; dcterms:description "An example dataset"@en
  ; dcterms:created "2023-08-13"^^xsd:date
  ; dcterms:creator :I
  ; dcterms:publisher :I
  ; dcat:contactPoint :I
  ; dcat:keyword "RDF"@en, "Ontology"@en, "Versioning"@en
  ; dcat:theme <http://example.org/themes/semantic-web>
  ; dcterms:license <http://creativecommons.org/licenses/by/4.0/>
]
```

(You can find more examples of the configuration in the [User Guide](/docs/user-guide/#setting-up-a-repository-in-a-triple-store).)

Additionally, in global files, similar to Git, Bog Turtle files with more general default values for certain configurations and metadata can be specified, for example, about the user agent in general:

- `/etc/ontogen.conf.bog.ttl`
- `~/.ontogen.conf.bog.ttl`

The structure and interpretation of the description of an Ontogen service and its components thus looks as follows:

1. When Ontogen starts, all configuration files are loaded into a graph, starting with the global files. During this incremental construction of the graph, it should be noted that values for properties of the same resource are overwritten. This ensures that the values from the global configuration files only serve as default values and the repository-specific configuration files can completely overwrite these values if necessary.

2. The following Bog Turtle fragment is then added to this graph to ensure the basic static linking of the resources of the `og:Service` aggregate:
    ```turtle
    [ :this og:Service  
        ; og:serviceRepository [ :this og:Repository  
            ; og:repositoryDataset [ :this og:Dataset ]  
            ; og:repositoryHistory [ :this og:History ]  
        ]  
      
        ; og:serviceOperator :I  
    ] .
    ```

3. Then the Bog precompiler is applied to this graph, which resolves blank nodes to URIs according to the salts or mints them.

4. Finally, the resulting graph is loaded using [Grax](https://github.com/rdf-elixir/grax) (the RDF Data Mapper for Elixir used) into a deeply nested, native Elixir structure for the Ontogen service and all its components, which then serves as the state of the `Ontogen` singleton GenServer.

An issue that needs to be addressed soon is that the repository description is copied to the repository metadata graph in the store when the repository is set up (with `og setup`). Currently, there's no way to update this metadata when the configuration has been changed.

This issue, along with the previously mentioned lack of ability to introduce custom URIs for resources, is the reason why users currently have to work with cryptic graph names. Specifically, the Bog-generated URI of the `og:Dataset` is used as the name of the graph for the version-controlled dataset, the Bog-generated URI of the `og:History` serves as the name for the history graph, and the Bog-generated URI of the `og:Repository` is employed as the name of the repository metadata graph.

I consider this a significant drawback of the current implementation. Addressing these two issues - allowing metadata updates and introducing custom URI support - is therefore of the highest priority in the upcoming development work. The goal is to provide users with more intuitive and manageable graph naming conventions.

## Store adapters

An careful reader may have noticed that the basic structure of an `og:Service` described earlier lacks the link to the `og:Store`. The reason for this is that the configuration of the `og:Store` differs from that of other components, as different triple stores, although all SPARQL-based, require different configurations. Ontogen addresses this challenge through the implementation of store adapters.

In Ontogen, triple store adapters are implemented as subclasses of `og:Store`. This solution is not only conceptually very simple but also provides a comprehensive basis for solving this problem thanks to Grax and its support for polymorphic links. (In particular, we overcome the ["Walled Gardens within Elixir"](https://marcelotto.medium.com/the-walled-gardens-within-elixir-d0507a568015), which potentially allows store adapters to be developed and versioned as separate Hex packages if their complexity should increase over time, which might happen quickly when triple store-specific extensions are implemented in a store adapter.)

This should now also make clear why the statically added basic structure does not define a link to the `og:Store`: It is the user's decision which triple store their Ontogen service instance should run with. The user makes this decision by specifying an instance of the corresponding `og:Store` subclass in the service configuration. Consequently, the configuration files generated during the initialization of a repository look like this:

```turtle
# all triple store adapters have URIs in a dedicated namespace
@prefix oga: <https://w3id.org/ontogen/store/adapter/> .

[ :this og:Service

    #################################################
    # store selection

    # Select the adapter of your choice and feel free to remove the unused
    # (incl. their respective config files)
    ; og:serviceStore [ :this og:Store ]
    # ; og:serviceStore [ :this oga:Oxigraph ]
    # ; og:serviceStore [ :this oga:Fuseki ]

    #################################################
    # metadata

    ; dcterms:title "Your service name"
    ; dcterms:creator :I
    ; dcterms:publisher :I

] .
```

When initializing a repository, configuration files are generated for all available adapters. However, only the adapter specified here in the `og:Service` is actually used during operation.

If, as in the above case, no instance of a subclass is used but directly of `og:Store`, the `og:Store` configuration of the instance from `.ontogen/config/store.bog.ttl` is used, which implements a generic adapter. In this case, the URLs of the various endpoints of a triple store must be specified:

```turtle
[ :this og:Store
    ; og:storeQueryEndpoint <http://localhost:7878/query>
    ; og:storeUpdateEndpoint <http://localhost:7878/update>
    ; og:storeGraphStoreEndpoint <http://localhost:7878/store>
] .
```

In the configurations of the adapters, on the other hand, where the logic for generating the various URLs of the endpoints is implemented, only the corresponding components of these URLs need to be specified. Since the default values of these components for a standard installation of the corresponding triple store are also known in the adapter and are assumed if not present, an adapter configuration could theoretically be empty and still functional:

```turtle
[ :this oga:Oxigraph ] .
```

However, during initialization, complete configurations with all properties for the components (using the defaults) are generated to make them explicit and easily adjustable if necessary:

```turtle
[ :this oga:Oxigraph
    ; og:storeEndpointScheme "http"
    ; og:storeEndpointHost "localhost"
    ; og:storeEndpointPort 7878
] .
```

In some adapters, there are also components for which no sensible default exists and which therefore must be configured with a suitable value, such as in the case of Fuseki, the name of the dataset in which the repository should be stored (and which was previously created by the user in Fuseki):

```turtle
[ :this oga:Fuseki
    ; og:storeEndpointScheme "http"
    ; og:storeEndpointHost "localhost"
    ; og:storeEndpointPort 3030
    ; og:storeEndpointDataset "name-of-the-dataset"
] .
```

At this point, operation via an adapter does not yet differ from that with an analogous configuration of the generic `og:Store`. However, the implementation of the adapters is structured in such a way that the HTTP requests can be flexibly wrapped and very easily provided with additional logic if needed, for example, to support triple store-specific optimizations or extensions.

In the future, further generic configuration options should be specifiable in the generic `og:Store`, for example, for the different SPARQL HTTP request forms, so that further adaptation and optimization options exist for triple stores without their own adapter implementation in Ontogen.

Furthermore, adapters for other popular triple stores will of course be offered in the future.


## Conclusion

With this exploration of Ontogen's configuration system and the introduction to Bog, the introductory series on Ontogen comes to an end. The aim of these articles has been to present the ideas behind Ontogen, from its fundamental concepts to its practical implementation and configuration.

Ontogen aims to address some of the challenges in managing and versioning RDF datasets. While it's still in its early stages, it offers a new approach that combines established semantic web standards with novel ideas in versioning and configuration.

As an open-source project, Ontogen's future development will greatly benefit from community feedback and contributions. The project can be found on [GitHub](https://github.com/ontogen/ontogen), where stars, issues, and pull requests are always appreciated.

For those interested in following the project's progress or reading future articles, updates are available via [RSS](https://ontogen.io/feed.xml), [Mastodon](https://mastodon.social/@marcelotto), or [LinkedIn](https://www.linkedin.com/in/marcel-otto-a68245b8/).

Thank you for your interest in Ontogen. I look forward to seeing how it might be used and improved in the future.
