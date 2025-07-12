+++
title = "LLMs in the Building Industry"
description = "or: Ways in Which the Architecture, Engineering and Construction Profession can Benefit from Large Language Models in Specific and Algorithms in Generic"
date = 2025-07-13
+++

This is a collection of ideas that may or may not be feasible to implement or to have as the base idea of a start up. It is _not_ a structured document, but more of a stream of thought.

# IFC

[Industry Foundation Classes](https://en.wikipedia.org/wiki/Industry_Foundation_Classes)[^ifc] is an open source data format for modeling BIM (Building Information Modeling). It is supported by Revit, the market leader. A well known open source implementation is [Bonsai BIM](https://bonsaibim.org). I _believe_ the implementation is inherently text based and can be accessed with programmer oriented tooling such as `git`.

[^ifc]: A pun on "Issued For Construction".

There are graphical tools that allow the user to see the difference between two files visually. An added wall in this revision would show up as green, for example. A deleted wall would be shown in red. This is all existing tooling.

---

# For Potential Homeowners

## Scenario

Imagine the following: a website where a client can submit their plot details, their budget, and make a write up on what they want in their dream house.

For example: They have three kids and their eldest wants to be a ballerina so they need a dance studio space. The father works from home so he needs a home office and a quiet space for virtual meetings. The mother is a hobbyist swimmer so they want a swimming pool. They have no need for a specific guest space, etc. Whatever a potential customer would tell their architect.

The website would take the hard data from the plot and the plot's location, and the specifics of the local area building code, and the text description provided by the client (the more detailed the better). Then out of these disparate pieces of the info and combine them together into the ideal house for the subject.

## Sanity Checks

This would require some sort of sanity checker to verify the results, similar to how LLM-based coding agents work with compilers and language serverso.[lsp] The LLMs hallucinate whatever they want, and the Editor tells them whether the code compiles or not. Sanity checks include, but are not limited to, code compliance, cost overruns, "there is a room without a door", etc.

[^lsp]: Commonly known as LSPs, short for Language Server Protocol.

This process can be automated very far. IFC can include data relevant to the other disciplines than Architecture. Most importantly, structural design. This should be an algorithmic task as it is a life and death matter and there is no room for ~~hallucinations~~ creativity. The other disciplines such as plumbing, electric wiring, etc is doable and any mistakes can be easily adjusted (or completely ignored) on site by the installation teams.

The IFC files also include dimensions. Combine that with the specifications, or the approved suppliers, (which the client may or may not choose from a drop down menu), and you get a quick and easy to calculate cost.

There *must* be a qualified human reviewer, as the final sanity check. More importantly, to shoulder the legal liability of architectural and structural designs.

The price could be fairly cheap: 1,000SAR for one plan. The absolute cheapest office is about 5,000, and these are the ones just pulling a predesigned house out of the drawer.

## Scale

Working at scale, a similar technology can be used to benefit real estate developers. Instead of the usual cookie cutting measure of a thousand identical houses, variation can be made in each individual unit. Depending on the construction methods used, this might not be more expensive to actually build than cookies.

---

# For Engineering Offices

There are many different professions working in a regular Engineering[^chagrin] practice. This is an overview following the different stages of a project.

[^chagrin]: Including Architectural and Urban Planning practices.

## Client Brief

Every project starts with a brief.[^brief] The brief includes an intro the project and the different stakeholders and potential desires etc. To respond to the brief, offices are required usually to make both a technical offer and a financial offer.

[^brief]: Basically what the potential homeowner in the previous example wrote in our website's entry

The technical offer is essentially a listing of the Office's capabilities. What is their experience like? What are their core competencies? What big projects they have done before? And what have you. The exact format is usually specified by the client, in the brief, but the details are generally all the same. The same info and files are regurgitated between different projects.

The financial offer is more of the same. It is based on however the office wants to to price the subject. The structure of which is also often dictated by the brief. It could be by man hours or by lump sum or depending on the phase of the moon. The cost can depends on the size of the project and the nature of the client (government vor private).

This is a lot of clerical and reptitive work that nonetheless must be customized for each client. It involves a large number of staff from different departments: accounting, secretaries, engineering, const estimation (which could fall under engineering).

## Specifications

Specifications are the most important, and perhaps the most overlooked, part of the design documents. They can either be dictated by the client or, more appropriately, delivered as part of the service.

First, a brief explanation of what Specifications _are_. Take for example masonry. The specification for masonry should specify the following:

- Potential suppliers,[^saudi]
- Desired properties: size, material,
- How to install
- How to maintain

[^saudi]: Not allowed in Saudi government contracts, I believe.

Sepcifications for large projects can include everything up to the type of screw used to fasten the furniture together. They would be used by the eventual contractor to procure the proper materials.

A common practice in offices is to have a local databaee of approved, or potential, suppliers of different building materials. Indeed Engineering offices are often the first line of marketing for any building material supplier.

Large offices have their own specifications library. Others make them up as they go along based on the experience of the engineer responsible for them at the time, copy pasting at need from previous projects. Some companies actually sell their specifications, and supplier databses, to potential Architectural offices. One example of that is [AIA MasterSpec](https://www.aia.org/masterspec).

A potential use of LLMs here is to genrate these specifications from a local supplier database. Perhaps even contrasting that with the client's brief and their desired level of quality. The Model could be given a template for specifications, or the specirications for a previous project, and directed to generate specificatiosn based on the design drawings and written guidelines and up to date supplier database.

## Consultancy

Work for Engineering offices is not restricted to design. Indeed the more profitable line of work is construction consultancy, or supervision. Reviewing the contraxctor's submittals for materials, shop drawings,[^shop] staff listings, Requests For Clarification (RFCs), and what have you.

[^shop]: Exact drawings of what will _actually_ be built.

The reviews could be given to a LLM, as a first pass before a human reviewer looks at them. But I do not think this is a valuable use as the details that would fail a review, especially for technical drawings, are something that is still currently beyond the scope of language models.

I will touch on this again when talking about contractors, but a record of all communication is extremely important to maintain throughtout the life cycle of the project. If, say, a feature of the building turned out not to have the desired quality, the consultant can point to the communication where they denied the contractor's shop drawings or chosen materials, but were overruled by the client in the interest of expediency or whatever. Someone, or something, which can read and remember and vaguely understand the conversation, would be invaluable to, well, win arguments.

The methods of communication vary by project. Either they are official stamped by the treplicate official letters, or more commonly WhatsApp messages, or anything inbetween.

---

# For Construction Companies

Much of the talk about the client briefs and comparing specifications with suppliers database applies here as well. Searching a local database, not necessarily neatly organized with all the relevant info buried in PDFs and websites, to find suppliers whose products match the given specifications.

There are also many applications for logistics. How to store the materials and how to transport the materials and how to protect them from the elements are all common challenged. But not unique to construction work. So solutions from other industries (such as manufacturing) could be used.

## Cost Estimations

This is perhaps a task more appropriate for proper old algorithms than language models. but a common painpoint small contractors have is how to properly estimate the cost of a building, based on drawings and a client brief. The drawings themselves are usually presented as PDFs or as DWG files (sometimes). One would be truly lucky to get the drawings in IFC (or really any BIM) format. PDFs and DWGs include no metadata about the project's elements. The data is provided by things designed for human readers: hatching, dimensions, and labels.

There are a few ways this can be approached. Either reading the drawings as a human would: looking up the labels and cross referencing them with the specifications (if any). Measuring the length of different elements, walls and windows and what have you, multiplying that by estimated labor and material costs. All directly from the drawings.

The other approach is to create a IFC model based on the drawings, and have the regular BIM tooling to calcluate this data. The prices could be tied to some live database of material prices, or not.

## Waste Reduction

Note this common particular scenario. Structual steel reinforcement comes in 12m long bars. A common floor height is 4m. However, due to structual requirements that are beyond my understanding, the bars embedded in columns should be one meter longer. This means that one bar can be cut in two 5m long pieces. You end up with a useless wasted 2m piece. There are about 12 bars in every column and the floor of a typical house can have around 25 columns. You do the marh.

There are different ways to solve this: a price concious Structural Engineer can reuse the same bar width in different locations allowing every piece to be used for other building elements.[^weld] Excellent, and expensive, labor force can find creative uses for the waste. Or the project manager can sell them for scrap to another project that might use them.

[^weld]: Unfortunately, apprently welding two bars together is not an acceptable option.

The problem exists beyond structural steel, however. Masnory units can have up to 15% waste or more, due to very different factors. Getting careful labor that minimizes waste is potentially more expensive than the material wasted. Different building methods, such as bearing walls, are not quite .. approved of in the local market.[^refrain] A resourceful project manager or designer can find creative uses for the waste.

[^refrain]: "What if I need to change the layout?" is a common refrain, even if nobody actually does.

I am actualy unsure how algorithms or AI can help here. I am mostly typing out thoughts.

However, in a regular concrete building, a commonly used perishable, but expensive, material is formwork. Most formwork used in Saudi Arabia is wood: either plywood or "white wood". It is expensive to buy. Expensive to rent. And it is necessary to cut to fit the weird concrete shapes architects come up with. And laborers, especially if it is not _their_ wood, are extremely careless in economizing it.

A few real life products can be used to solve this problem. Aluminum and Plastic based formwork is infinitely more reusable, but because it cannot be cut, it is not as flexible for different shapes.

the ideal solution would be to design, from the start, for minimizing formwork waste. However, as this is the Contractors section, an algorithm to minimize waste on a given design (how to cut the boards in a reusable manner, etc.) would be extremely useful to contractors.

## Clarifications and Change Orders

Managing clients and consultants is perhaps the Contractor's most difficult task. The scopes keep on expanding and the specifications keep on tightening while prices and schedules are expected to remain the same. The drawings or specificiations are often incomplete and missing vital info, or just plain wrong.

Having a record of all communications (often called Document Management) is extremely important for all parties involved. Being able to search and query the often disorganized communication records to find all the different instructions from the client and consultant is evidently important.

These clarifications can begin even before signing the contract. When bidding on the project, the contractor is expected, but not really, to completely understand and fully evaluate the contract documents (drawings and specifications), and point out any missing info. A bid is, usually, essentially, an agreement that the contract documents are completely clear. However, analysis and fully reading and grokking the documents is very time consuming, potentally money-losing, work. Automating that discovery phase would be a huge boon to contractors bidding.

The Model does not need to be extremely accurate. A few false negatives and false positives would be very tolerated, as long as it is more accuracy than a human reviewer in a reasonable amount of time (a couple of work days, maybe). You would get the contract documents, feed them to the Model, perhaps with prior art, and ask does it have any ommisions or incomplete specifications of unlabelled, unspecified, elements or any contradictions. A human reviewer would go over whatever the Model flags and go on with their day.

This verification step cen be useful for Engineering Offices as well.

---

# Conclusion

In conclusion, this documents outlines a few ways which Large Language Models and Algorithms can be used to ease the pain of the different people in the construction process. These omit some things like dealing with government regulations and accounting and human resources and what have you: the lements common in almost eveyr business. But it tpouches, instead, what is perhaps unique to the profession.

---
