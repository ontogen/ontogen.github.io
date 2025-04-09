---
title: "Project Update: New Roadmap for 2025 and release announcements"
date: 2025-04-09 12:00:00 +0200
categories:
  - blog
permalink: 
header:
  image: /assets/images/ontogen-roadmap-article-2.png
mermaid: false
---
The work on Mud, the extracted and enhanced version of Bog, had been progressing well towards solving the problem of static UUID graph names in Ontogen repositories. However, when it came to extending automatic URI generation and management to the resources of the dataset itself, it became increasingly clear that Bog's strategy alone was insufficient. So, after going back to the drawing board, I've developed a new approach. While requiring significantly more effort than initially planned, it promises a comprehensive solution to the URI generation problem which, as described in the [Bog article](/introduction/part-4), I consider one of the fundamental challenges of the Semantic Web.

During the design of this new solution, it also became clear that the project must be split up further, so that there's now also a need for an additional extraction: that of the DCAT-based Repository and Service model (described in [this article](/introduction/part-3)). This separation will make the dataset management capabilities available independently while allowing Ontogen to focus exclusively on its core versioning functionality.

The project's evolution, particularly additional requirements that emerged for the planned DID implementation, has also led to several updates to the RDF on Elixir libraries, some of which are being released today:

- JSON-LD.ex v1.0 with JSON-LD 1.1 support
- RDF.ex v2.1 with `rdf:JSON` literal support
- Grax v0.6 with `rdf:JSON` support and ordered lists based on `rdf:List`s

Please refer to the respective CHANGELOGs for a comprehensive list of the changes.

Further planned library updates include:

- Support for the upcoming RDF 1.2 specification in RDF.ex
- Support for JSON-LD framing
- Two patterns used repeatedly to solve problems in the new design will be implemented as Grax extensions:
    - Managing temporal values in RDF, tracking how values change over time while maintaining their history
    - Handling multiple language-tagged string values as a unique value, enabling functional property behavior for localized strings

Some previously announced features, at least Sagas, will need to be deferred to a future development phase to accommodate this more fundamental work.

While this represents a shift from the original roadmap, I strongly believe that the benefits of the planned comprehensive URI management outweigh the delay of other features.