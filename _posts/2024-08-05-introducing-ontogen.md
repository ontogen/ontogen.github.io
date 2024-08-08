---
title: Introducing Ontogen
date: 2024-08-08 10:00:00 +0200
categories: [introduction, blog]
permalink: /introduction/part-1
header:
  image: /assets/images/ontogen-introduction-article-1.png
---
After a year of intensive development, I am pleased to introduce Ontogen - a version control system for RDF datasets. My sincere thanks go to the [NLnet Foundation](https://nlnet.nl) for their support through the [NGI Assure](https://nlnet.nl/assure) fund, which enabled me to dedicate myself full-time to this extensive project.

It's important to emphasize that Ontogen is still in an early stage of development. Although the system is equipped with a comprehensive test suite and is regularly tested with two different triple stores (Fuseki and Oxigraph), it lacks extensive real-world testing. Therefore, I cannot yet recommend its productive use for critical data. In particular, some important extensions are still pending:

1. Full RDF dataset support (currently only versioning of individual graphs is supported)
2. Branching support

These and other extensions will require fundamental changes that will likely invalidate existing version histories.

In the coming weeks, I plan to publish the project's future roadmap. An application for another round of funding from the NLnet Foundation for at least one year is currently in progress.

Ontogen offers some novel approaches to versioning RDF data (at least to my knowledge). To adequately explain these complex concepts, I will be publishing an article daily for the remainder of this week, to introduce the different parts of the system and the ideas behind them. In this first post, I'd like to introduce the technical foundations of Ontogen's approach to versioning RDF datasets.

## Source Control Management vs. Data Control Management

First, however, I'd like to take a step back and discuss the versioning problem with a particular focus on datasets. This distinct perspective is, to my knowledge, rarely taken, and in practice, software versioning solutions, i.e., SCMs like Git, are too often used for versioning datasets.

Datasets, however, represent a indeed a different type of versioning object compared to software. While both datasets and software may ultimately always be text, no one would dispute the claim that data is not the same as software. But why then is there no popular versioning solution for datasets of structured data, especially in the age of "data as the new oil," where data management and analysis play central roles in almost all industries?

Some examples illustrate the inadequacies of an SCM system as a Data Control Management (DCM) system:

- Roles: In an SCM, the committer is the crucial role. While SCMs recognize the difference between author and committer, in practice, this is usually of little importance. For a dataset, however, authorship, i.e., the exact source of datasets, is of greater importance, and many other roles are relevant and should be differentiable, such as data processors (people or systems that transform, clean, or enrich raw data), data curators (experts who organize, categorize, and enrich data with metadata), data protection officers, etc.
- Lack of metadata: Datasets often require extensive metadata (e.g., origin, license, timestamps) that are not natively supported in SCMs.
- Granularity of changes: SCMs often work at the file level, while for datasets, individual records or fields may be relevant.
- Database integration: DCMs should ideally be able to interact directly with database systems, which is not provided for in SCMs.

Although increasing attention has been focused on the dataset versioning problem in recent years, it must be noted that no mainstream solution has yet emerged. Instead, SCMs are still too often resorted to for dataset versioning (if they are versioned at all).

The situation is particularly precarious in the Knowledge Graph community, especially in the area of RDF data. Here, there is a lack of mature, specialized solutions for versioning.


## Problems with Previous Versioning Systems for RDF

Attempts to develop solutions for versioning RDF data have existed for a long time. Unfortunately, all solutions developed so far are either academic proof-of-concepts or approaches that have not found broader acceptance in the community for various reasons. In my opinion, a major reason for this is that most previous solutions had to rely on named graphs due to a lack of alternatives, which leads to numerous practical problems.

The main problem with named graphs for versioning is that parts of a graph simply cannot be addressed directly. This forces us to create a separate named graph for every small group of triples we want to version, which can quickly lead to a flood of graphs and thus an unwieldy RDF dataset. This becomes particularly problematic when working with different named graphs for content purposes, which then become difficult to distinguish among this flood.

With RDF-star, which is currently being standardized as RDF 1.2, a new tool is now available. RDF-star is an extension of the RDF data model. It allows direct annotation of RDF triples by allowing triples to be used as subjects or objects in other triples. This simplifies the representation of metadata about statements and enables a more natural modeling of complex relationships without having to resort to reification or named graphs.

The possibility of making RDF-star meta-statements with other statements as the subject opens up significant new possibilities. In particular, it is now trivial to define virtual, URI-identifiable sets of statements, i.e., partial graphs within a graph, by assigning the statement to a common resource using a property. This makes RDF-star an ideal foundation for versioning RDF graphs.


## RTC as a Foundation for RDF Data Versioning

To provide a well-known property with this assignment semantics for partial graphs and thus create a foundation for tools that exploit this semantics, I published the [RDF Triple Compounds (RTC) vocabulary](https://w3id.org/rtc) last year. It consists of only one RDFS class `rtc:Compound` and a few properties, including in particular the `rtc:elementOf` with the aforementioned semantics and its inverse `rtc:elements`.

```turtle
rtc:Compound 
	a rdfs:Class, owl:Class ;
    rdfs:label "Compound" ;
    rdfs:comment "A compound is a set of triples as an RDF resource." .  
  
rtc:elementOf 
	a rdf:Property, owl:ObjectProperty ;
    rdfs:label "element of" ;
    rdfs:comment "Assigns a triple to a compound as an element. The subject must be a RDF triple." ;
    rdfs:range rtc:Compound ;
    owl:inverseOf rtc:elements .
  
rtc:elements 
	a rdf:Property ;
    rdfs:label "elements" ;
    rdfs:comment "The set of all triples of a compound. The objects must be RDF triples." ;
    rdfs:domain rtc:Compound ,
    owl:inverseOf rtc:elementOf .
```

A _triple compound_ of this vocabulary is thus a set of triples assigned to a common resource. The triples are assigned to a compound with an RDF-star statement using the `rtc:elementOf` property.

```turtle
PREFIX : <http://www.example.org/>
PREFIX rtc: <https://w3id.org/rtc#>
 
:employee38 
    :firstName "John" {| rtc:elementOf :compound1 |} ;
    :familyName "Smith" {| rtc:elementOf :compound1 |} ;
    :jobTitle "Assistant Designer" {| rtc:elementOf :compound1 |} .

:compound1 a rtc:Compound ;
    :statedBy :bob ; 
    :statedAt "2022-02-16" .
```

Alternatively, the `rtc:elements` property can be used as the inverse of `rtc:elementOf`.

```turtle
PREFIX : <http://www.example.org/>
PREFIX rtc: <https://w3id.org/rtc#>
 
:employee38 
    :firstName "John" ;
    :familyName "Smith" ;
    :jobTitle "Assistant Designer" .

:compound1 a rtc:Compound ;
    :statedBy :bob ; 
    :statedAt "2022-02-16" ;
    rtc:elements        
       << :employee38 :firstName "John" >> ,
       << :employee38 :familyName "Smith" >> ,
       << :employee38 :jobTitle "Assistant Designer" >> .
```

Simultaneously with the vocabulary, an Elixir implementation [RTC.ex](https://github.com/rtc-org/rtc-ex) was also provided, which, based on this vocabulary, provides a structure that allows working with these virtual graphs in a way that is largely API-compatible with real graphs, as implemented in the `RDF.Graph` structure of RDF.ex.

```elixir
# create a new compound with a couple triples
virtual_graph =  
  [  
    {EX.Employee38, EX.firstName(), "John"},  
    {EX.Employee38, EX.familyName(), "Smith"},  
    {EX.Employee38, EX.jobTitle(), "Assistant Designer"},  
  ] 
  |> RTC.Compound.new(EX.Compound, prefixes: [ex: EX])  

# add some triples to the compound
virtual_graph =  
  RTC.Compound.add(virtual_graph,   
    EX.Employee39  
    |> EX.firstName("Jane")  
    |> EX.familyName("Doe")  
    |> EX.jobTitle("HR Manager")  
  )  

# add some statements about the compound itself
virtual_graph =  
  RTC.Compound.add_annotations(virtual_graph,  
    %{EX.dataSource() => EX.DataSource}  
  )
```

With the `RTC.Compound.graph(virtual_graph)` function, the pure set of statements can be produced as an `RDF.Graph` at any time, which in this case generates this graph:

```turtle
@prefix ex: <http://example.com/> .

ex:Employee38
    ex:familyName "Smith" ;
    ex:firstName "John" ;
    ex:jobTitle "Assistant Designer" .

ex:Employee39
    ex:familyName "Doe" ;
    ex:firstName "Jane" ;
    ex:jobTitle "HR Manager" .
```

Whereas `RTC.Compound.to_rdf(virtual_graph)` provides the complete RDF-star graph with the RTC annotations for the compounds.

```turtle
@prefix ex: <http://example.com/> .
@prefix rtc: <https://w3id.org/rtc#> .

ex:Compound
    ex:dataSource ex:DataSource .

ex:Employee38
    ex:familyName "Smith" {| rtc:elementOf ex:Compound |} ;
    ex:firstName "John" {| rtc:elementOf ex:Compound |} ;
    ex:jobTitle "Assistant Designer" {| rtc:elementOf ex:Compound |} .

ex:Employee39
    ex:familyName "Doe" {| rtc:elementOf ex:Compound |} ;
    ex:firstName "Jane" {| rtc:elementOf ex:Compound |} ;
    ex:jobTitle "HR Manager" {| rtc:elementOf ex:Compound |} .
```

The RTC property to be used can be configured either globally via application configuration or individually as an option of the `RTC.Compound.to_rdf/2` function.


## Summary and Outlook

In this article, we have laid the foundations for Ontogen's approach to versioning RDF datasets. We have discussed the inadequacies of existing solutions and introduced RDF Triple Compounds (RTC) as the technical basis of Ontogen. RTC utilizes the capabilities of RDF-star to enable flexible and efficient groupings of RDF triples without having to accept the disadvantages of named graphs.

In the [next article](/introduction/part-2), we will take a closer look at the application of RTC in Ontogen. We will show how RTC compounds serve as building blocks for a versioning system that takes into account the various roles and aspects of data management. We will explain how Ontogen uses RTC to enable fine-grained version control while maintaining the clarity of the dataset.
