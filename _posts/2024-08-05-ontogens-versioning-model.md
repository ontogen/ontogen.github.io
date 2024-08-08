---
title: Ontogens Versioning Model
date: 2024-08-08 11:00:00 +0200
categories: [introduction, blog]
permalink: /introduction/part-2
header:
  image: /assets/images/ontogen-introduction-article-2.png
---
In the [previous article](/introduction/part-1) of our introduction series to Ontogen, we introduced RDF triple compounds (RTC). These triple compounds now serve as the foundation for Ontogen as a Data Control Management (DCM) system for RDF datasets. So, how exactly do the triple compounds used in Ontogen for versioning RDF datasets look?

Ultimately, our goal is to annotate sets of statements with metadata to automatically organize them in a version history. However, the challenge with triple compounds is that the changes we want to make and commit atomically are not necessarily a simple set with a single semantic meaning. Instead, they are sets with different change semantics that can potentially occur simultaneously in any combination: sets of statements to be added to a dataset, updated, or deleted.

Let's consider the update of personal data after a marriage involving a name change. This complex but atomically related process requires different types of changes that should be encapsulated in a single, atomic entity. Imagine Sarah Miller gets married and takes her partner's last name. Her new name is now Sarah Johnson. This scenario could involve various sets of statements with different change semantics. While we want to update the family name and marital status, we might simply want to add a new email address or wedding date in this context without overwriting old values.

Therefore, we need a potentially complex entity that encompasses these sets of changes.


## RDF SpeechActs

In the Ontogen vocabulary (prefixed in the following with `og:`), these entities are called `og:SpeechAct`s. We draw on the concept of speech acts from philosophy, which might seem unusual in this context at first glance. However, upon closer examination and when limited to RDF statements as the subject of speech acts, it perfectly models the situation in our versioning problem for RDF datasets.

A [speech act](https://en.wikipedia.org/wiki/Speech_act), a term coined by J.L. Austin and further developed by John Searle, refers in linguistics and philosophy of language to an utterance that not only conveys information but also performs an action. Central to speech act theory is also the consideration of the context in which an utterance takes place. By including context, speech act theory extends the analysis of utterances beyond the purely semantic level to the pragmatic dimension.

To bridge the gap between the general concept of speech acts and their specific application in Ontogen, it's important to understand how we've adapted this philosophical idea to our technical context. An `og:SpeechAct` represents a very specific form of speech act: the utterance of RDF statements or modification. This adaptation allows us to capture not just the content of RDF data, but also the act of asserting or changing that data, along with all the contextual information that surrounds that act.

In the context of Ontogen and RDF data, we apply this concept by treating each utterance of RDF statements as an `og:SpeechAct` - an action that does not represent the actual addition or modification of the dataset (we will continue to call this action a *commit*, following the usual versioning terminology). Instead, it represents the act of the original utterance of the statements in this dataset or subsequent acts that supplement, revise, confirm, etc. the original statements. This is because the central questions of provenance, i.e., the origin of the data, revolve around these acts. Some examples:

- Who or what uttered, derived, or generated the data and when?
- In what context was the data collected or modified?
- With what intention or for what purpose was the data created or modified?
- How was the data validated or verified?

With `og:SpeechAct`s, our aim is to provide a model that allows us to capture information related to all possible questions about these utterances and record them as metadata.

Fortunately, we don't need to develop a new ontology from scratch to model these metadata. There is already an excellent and standardized basis: the [W3C PROV vocabulary](https://www.w3.org/TR/prov-overview/). The W3C defines Provenance as follows:

> *"Provenance is information about entities, activities, and people involved in producing a piece of data or thing, which can be used to form assessments about its quality, reliability or trustworthiness. The PROV Family of Documents defines a model, corresponding serializations and other supporting definitions to enable the inter-operable interchange of provenance information in heterogeneous environments [...].*
> 
> -- <https://www.w3.org/TR/prov-overview/>

This definition fits perfectly with our approach of using speech acts as a basis for capturing provenance information in RDF datasets. In Ontogen, we therefore define our `og:SpeechAct`s as a special manifestation, i.e., a subclass of `prov:Activity`. 

> *"An activity is something that occurs over a period of time and acts upon or with entities; it may include consuming, processing, transforming, modifying, relocating, using, or generating entities."*
> 
> -- <https://www.w3.org/TR/prov-o/#Activity>

This allows us to leverage the extensive semantics of the PROV vocabulary while modelling our specific concept of speech acts for RDF data.


## Propositions

What characterizes `og:SpeechAct`s as special `prov:Activity`s, i.e., how exactly are they defined? To answer this, we return to the issue mentioned at the beginning: how can we express potentially complex sets of statements with different "change semantics" or, as we can now more precisely say, with different pragmatics using triple compounds? The recently introduced `og:SpeechAct`s now take on the role of the carrier resource with which the triple compounds are associated. An `og:SpeechAct` is thus a `prov:Activity` that can be associated with triple compounds with different pragmatics via four properties, the so-called *action properties* (which are all defined as sub-properties of `prov:used`).

1. `og:add`: A triple compound, i.e., a set of statements that is simply asserted without any further intentions. When persisted to a dataset in a triple store, these statements should be added.
2. `og:update`: A triple compound, i.e., a set of statements that is asserted with the intention to overwrite all previous statements with the same subject and predicate. When persisted to a dataset in a triple store, these statements should be added and the existing statements with the same subject and predicate should be overwritten.
3. `og:replace`: A triple compound, i.e., a set of statements that is asserted with the intention to overwrite all previous statements with the same subject. When persisted to a dataset in a triple store, these statements should be added and the existing statements with the same subject should be overwritten.
4. `og:remove`: A triple compound, i.e., a set of statements that should be retracted ("negated"). When persisted to a dataset in a triple store, it should remove these statements.

Let's first look at how our initial example of a name change would look as an `og:SpeechAct` with normal triple compounds, if we initially use simple blank nodes as identifiers for the `og:SpeechAct` and their triple compounds:

```turtle
@prefix : <http://example.com/> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix rtc: <https://w3id.org/rtc#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

_:SarahMillerMarriageUpdate
    a og:SpeechAct ;
    og:update [
        a og:Proposition ;
        rtc:elements 
            << :employee39 :familyName "Johnson" >>,
            << :employee39 :maritalStatus :Married >>
    ] ;
    og:add [
        a og:Proposition ;
        rtc:elements 
            << :employee39 :emailAddress "sarah.johnson@example.com" >> ,
            << :employee39 :marriageDate "2023-07-25"^^xsd:date >>
    ] .
```

Regarding these action properties, it should be noted that it is the properties that define the pragmatics of the triple compound. The `rdfs:range` of these properties is always a triple compound set of the same type. 

However, the action properties do not associate the `og:SpeechAct` with triple compounds in general, but with `og:Proposition`s, a subclass of `rtc:Compound`, i.e., a special kind of triple compound that should exhibit some particular properties in this versioning context. We achieve this by using URI-encoded SHA256 hashes of the statement set for the URIs of the `og:Proposition`s.

This approach allows us to achieve some important properties for our use case. On the one hand, we have a method to automatically generate the URIs of the `og:Proposition`s. On the other hand, the authenticity of the statement set of the `og:Proposition` is verifiable, as we can detect whether the statement set is unchanged. Each `og:Proposition` can be verified by recalculating the hash of its canonicalized triple sets. This ensures that the integrity of the data is guaranteed throughout the entire version history. Changes or manipulations to the data would inevitably lead to a change in the hash and thus an inconsistent URI, which would be easily detectable.

Furthermore, the `og:Proposition`s exhibit an interesting identity property: propositions with the same set of statements receive the same URI. This means that if the `og:Proposition` statement set appears in different `og:SpeechAct`s, for example, because it was `og:add`ed once and `og:remove`d once, the same statement set should not be duplicated a third, fourth, or fifth time, etc., in the `rtc:elements` of different `og:Proposition` compounds, but should reference the same `og:Proposition`.

Thus, `og:Proposition`s represent immutable, abstract sets of statements that can be used in different, independent contexts and always have the same URI. However, this is only true to a limited extent without further measures, as there is still one problem: if two `og:Proposition` statement sets contain blank nodes and differ only in the local names used for the blank nodes, but are otherwise isomorphic, they are actually the same abstract statement set, which should therefore lead to the same URI. This can be achieved by bringing the RDF dataset into a canonical form before hashing, using the W3C-standardized [RDF Dataset Canonicalization Algorithm](https://www.w3.org/TR/rdf-canon/), in which isomorphic graphs with different blank nodes receive the same blank node identifiers.

So, `og:Proposition`s are an abstraction over concrete statement sets: the same statement set in different triple stores, with potentially different blank nodes, are all identified by the same URI of a resource. We identify the resulting equivalence set with a unique hash, so to speak. These `og:Proposition` compounds are thus, like the propositions of logic, abstract entities that are not bound to any utterance. Only through the utterance within an `og:SpeechAct` do they become time- and context-bound. As abstract entities, unlike RTC compounds in general, the `og:Proposition` compounds usually do not contain metadata, as the interesting metadata is utterance-related and therefore belongs to the `og:SpeechAct`.

However, we are still missing the URI of the `og:SpeechAct` itself. In Ontogen, this is

- the URI-encoded SHA256 hash of all `og:Proposition`s linked through its action properties (i.e., their SHA256 URIs), 
- the `prov:endedAtTime` timestamp, 
- and the `og:speaker` (a subproperty of `prov:wasAssociatedWith`) of the `og:SpeechAct`, or the `og:dataSource` (a subproperty of `prov:used`) if the speaker is unknown.

Now we can provide the actual Ontogen form of our above example `og:SpeechAct` of a name change by adding the previously missing SHA256 URIs of the `og:Proposition`s and the `og:SpeechAct`:

```turtle
@prefix : <http://example.com/> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix rtc: <https://w3id.org/rtc#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix prov: <http://www.w3.org/ns/prov#> .
@prefix dc: <http://purl.org/dc/terms/> .

<urn:hash::sha256:b1f9fb63d4cbfcc48c9da35c0526a1aeb394dce9f6fc10368d5ec3d248a8f070>
    a og:SpeechAct ;
    dc:description "Update of Sarah Miller's personal information following her marriage" ;
    og:add <urn:hash::sha256:3e9347bcd3f39fba5afc5a4b2f132bb2468cf399a4dfa6e34f52aea85e755a6e> ;
    og:update <urn:hash::sha256:4d33b032442d54f94052b55643240adf79dbea7ae635c5fb167a124dc3d6444a> ;
    og:speaker :JaneSmith ;
    prov:startedAtTime "2023-07-26T09:30:00Z"^^xsd:dateTime ;
    prov:endedAtTime "2023-07-26T09:31:15Z"^^xsd:dateTime ;
    prov:wasInformedBy :MarriageCertificateSubmission ;
    prov:used :MarriageCertificate20230725  .

<urn:hash::sha256:3e9347bcd3f39fba5afc5a4b2f132bb2468cf399a4dfa6e34f52aea85e755a6e>
    rtc:elements 
        << :employee39 :emailAddress "sarah.johnson@example.com" >>, 
        << :employee39 :marriageDate "2023-07-25"^^xsd:date >> .

<urn:hash::sha256:4d33b032442d54f94052b55643240adf79dbea7ae635c5fb167a124dc3d6444a>
    rtc:elements 
        << :employee39 :familyName "Johnson" >>, 
        << :employee39 :maritalStatus :Married >> .

:JaneSmith a prov:Person ;
    foaf:name "Jane Smith" ;
    foaf:mbox <mailto:jane.smith@example.com> .

:MarriageCertificate20230725 a prov:Entity ;
    dc:title "Marriage Certificate for Sarah Miller and Michael Johnson" ;
    prov:generatedAtTime "2023-07-25"^^xsd:date .

:MarriageCertificateSubmission a prov:Activity ;
    prov:wasAssociatedWith :employee39 ;
    prov:generated :MarriageCertificate2023-07-25 ;
    prov:endedAtTime "2023-07-26T09:00:00Z"^^xsd:dateTime .
```


## Commits

In Ontogen, commits represent the actual changes made to a repository, resulting from applying an `og:SpeechAct` to this repository with its existing data. Like an `og:SpeechAct`, `og:Commit`s are a `prov:Activity`. However, they represent the act of adding or modifying data in a dataset within a triple store in a specific state, rather than the act of uttering these statements, which may have occurred at a much earlier time and by a different speaker.

Like an `og:SpeechAct`, an `og:Commit` is a structure composed of `og:Proposition`s linked through various action properties. However, since they have a slightly different pragmatics here and a different `rdfs:domain`, different properties and an additional one are used for this purpose. The semantics of this set of properties is characterized by encoding repository-relative changes, i.e., expressing the minimal changes relative to the current state of the dataset. This is crucial to ensure that `og:Commit`s are revertible, as otherwise ambiguities in the history would arise:

- Can we simply remove every statement added by a commit from the triple store during a revert? If we cannot assume minimal changesets, it cannot be automatically decided whether this deletion can be performed. If the statement already existed, it must be removed to reproduce the old state. If a statement already existed before, it must not be removed.
- The same applies to removals of statements that do not actually exist in the current dataset and therefore should not be restored during a revert.
- Additionally, the specific statements implicitly deleted by `og:update`s and `og:replace` must be explicitly recorded in an additional proposition so that they can be restored during a revert.

To ensure the reversibility of commits, the internally so-called "effective changeset" must be determined. From this, corresponding propositions are then generated and linked to the `og:Commit` via the following action properties, along with the `og:SpeechAct` that this commit reproduces on the repository:

- `og:committedAdd`, `og:committedRemove`, `og:committedUpdate`, `og:committedReplace`: These action properties represent the minimal sets of statements as propositions necessary to change the state of the repository to match the corresponding `og:Proposition`s of the `og:SpeechAct`. They contain only the actually required changes.
- `og:committedOverwrite`: This property contains a set of statements as a proposition that represents the statements to be implicitly deleted due to updates or replacements.

If none of the changes conflict with existing triples (or in the case of `deletion`s, with non-existing triples) in the triple store, the `og:Proposition`s of the `og:Commit` are the same as the `og:Proposition`s in the `og:SpeechAct` and do not require additional space for dedicated modified `og:Proposition`s. Theoretically, we could rely on two simple sets for additions and deletions to determine the statement sets of the effective changes. However, to increase the chance of reusability of `og:Proposition`s, we continue to use the same actions for commits, so that only in case of overlap with existing data, a separate, dedicated `og:Proposition` for the commit must exist. (To further optimize this case, support for `og:SpeechAct`-relative commits is planned for the future, where only `og:Proposition`s of the overlap are necessary, which is much more efficient in many cases.)

Beyond this composition of `og:Proposition`s, a commit is, of course, like in other version control systems, a sequential, uni-directionally linked list of commits to the respective predecessor commits. The predecessor commit is defined in Ontogen via the `og:parentCommit` property.

The automatically generatable identifiers of an `og:Commit` are, like those of `og:SpeechAct`s and `og:Proposition`s, again URI-encoded SHA256 hashes, in this case of:

- the hash URI of the parent commit
- the hash URIs of the propositions of the commit
- the URI of the committer
- the timestamp of the commit
- and the commit message

Finally, let's look at an `og:Commit` for our above example `og:SpeechAct` of the name change. Let's assume that Sarah was previously married once, but the divorce was not recorded in our data, so the `:maritalStatus` is already set to `:Married`and the corresponding change effectively does not need to be made.

```turtle
@prefix : <http://example.com/> .
@prefix og: <https://w3id.org/ontogen#> .
@prefix rtc: <https://w3id.org/rtc#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix prov: <http://www.w3.org/ns/prov#> .
@prefix dc: <http://purl.org/dc/terms/> .

<urn:hash::sha256:a4f8c480f406140af4259783154811982d2bdd45eb39560c461a3f0601564357>
    a og:Commit ;
    og:add <urn:hash::sha256:3e9347bcd3f39fba5afc5a4b2f132bb2468cf399a4dfa6e34f52aea85e755a6e> ;
    og:update <urn:hash::sha256:0ca0d5b6b453830221787836bd8f85083fe571db2b340fbee106a04a7d206720> ;
    og:committer :JohnDoe ;
    prov:endedAtTime "2023-07-27T09:31:15Z"^^xsd:dateTime ;
    og:commitMessage "Update of Sarah Miller's personal information following her marriage" .

<urn:hash::sha256:0ca0d5b6b453830221787836bd8f85083fe571db2b340fbee106a04a7d206720>
    rtc:elements 
        << :employee39 :familyName "Johnson" >> .


<urn:hash::sha256:b1f9fb63d4cbfcc48c9da35c0526a1aeb394dce9f6fc10368d5ec3d248a8f070>
    a og:SpeechAct ;
    dc:description "Update of Sarah Miller's personal information following her marriage" ;
    og:add <urn:hash::sha256:3e9347bcd3f39fba5afc5a4b2f132bb2468cf399a4dfa6e34f52aea85e755a6e> ;
    og:update <urn:hash::sha256:4d33b032442d54f94052b55643240adf79dbea7ae635c5fb167a124dc3d6444a> ;
    og:speaker :JaneSmith ;
    prov:startedAtTime "2023-07-26T09:30:00Z"^^xsd:dateTime ;
    prov:endedAtTime "2023-07-26T09:31:15Z"^^xsd:dateTime ;
    prov:wasInformedBy :MarriageCertificateSubmission ;
    prov:used :MarriageCertificate20230725  .

<urn:hash::sha256:3e9347bcd3f39fba5afc5a4b2f132bb2468cf399a4dfa6e34f52aea85e755a6e>
    rtc:elements 
        << :employee39 :emailAddress "sarah.johnson@example.com" >>, 
        << :employee39 :marriageDate "2023-07-25"^^xsd:date >> .

  <urn:hash::sha256:4d33b032442d54f94052b55643240adf79dbea7ae635c5fb167a124dc3d6444a>
    rtc:elements 
        << :employee39 :familyName "Johnson" >>, 
        << :employee39 :maritalStatus :Married >> .

:JaneSmith a prov:Person ;
    foaf:name "Jane Smith" ;
    foaf:mbox <mailto:jane.smith@example.com> .

:MarriageCertificate20230725 a prov:Entity ;
    dc:title "Marriage Certificate for Sarah Miller and Michael Johnson" ;
    prov:generatedAtTime "2023-07-25"^^xsd:date .

:MarriageCertificateSubmission a prov:Activity ;
    prov:wasAssociatedWith :employee39 ;
    prov:generated :MarriageCertificate2023-07-25 ;
    prov:endedAtTime "2023-07-26T09:00:00Z"^^xsd:dateTime .
```

An interesting possibility, which is not yet used in these early versions of Ontogen but is planned for future versions, should be mentioned in conclusion. The speech act-based commit model outlined here allows for some useful validity checks of changes that are not possible in conventional version control systems. For example, a deletion expressed at an earlier point in time can be detected as actually obsolete because a later expressed insert of the same statement was committed earlier, or analogously, an insert expressed at an earlier point in time is actually obsolete because a later expressed deletion of the same statement was committed earlier or was not effectively committed there because the statement did not exist yet. Let's imagine, for example, that we had imported a dataset X in the very latest version and would later import a more comprehensive dataset Y that includes X, but in an older version. We can, for example, recognize from the more recent deletion of a statement from the already imported dataset that an addition in the now imported older version is already obsolete. Similarly, conflicts can be detected here that arise, for example, when updating a dataset on whose old version we have made some data cleansing.


## Summary and Outlook

In this article, we have delved into the core of Ontogen's versioning model, exploring how it leverages RDF Triple Compounds (RTC) to create a robust and flexible system for managing changes in RDF datasets. We introduced the concept of `og:SpeechAct`s, a novel approach that applies the philosophical notion of speech acts to the domain of RDF data versioning. These `og:SpeechAct`s, implemented as subclasses of `prov:Activity`, allow us to capture not just the content of changes, but also the context and intention behind them.

We then examined `og:Proposition`s, which serve as immutable, abstract sets of statements within our model. By using SHA256 hashes for their URIs, we ensure the integrity and verifiability of our data throughout the versioning process. This approach also allows for efficient storage and referencing of repeated statement sets across different `og:SpeechAct`s.

Finally, we discussed how Ontogen implements commits through `og:Commit`s, which represent the actual application of `og:SpeechAct`s to the repository. We explored the challenges of ensuring reversibility in commits and how Ontogen addresses these through careful management of change sets.

In the next article, we will expand our focus from the internal versioning mechanisms to the broader architecture of Ontogen. We'll explore how Ontogen organizes and manages repositories as DCAT catalogs and implements Ontogen instances as DCAT services.