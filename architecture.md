Disruptive Components: Architecture and Services
================================================

Have these open: 
 * http://findingaids.princeton.edu
 * http://findingaids.princeton.edu/collections/MC174
 * http://findingaids.princeton.edu/collections/MC174/c00357

I'm Jon Stroop, Digital Initiatives Programmer/Analyst @ Princeton University Library.

URI Templates
-------------
I'm going to start with what might seem like a strange topic: our URI templates. From the project stakeholders we had an idea about the different 'pages' our 'site' would have,  and as this is, ultimately, a __data-centric site__, it made sense to me to think of the URIs as a __high-level interface__ to the data that is exposed by the site. From the outset we knew we needed:

 * A page about each collection (`/collections/{call_number}`)
 * As Regine alluded to, if we were going to slice the components into separate pages, then a page for each component (`/collections/{call_number}/{component}`)
 * A page that listed the repositories (`/repositories`) (branded as 'Locations')
 * A page that listed collection 'topics'; This is basically a local set of high-level classification terms that was developed by our curators to group our collections into broad categories that reflect Princeton collection strengths. (`/topics`)

```
/collections/{call_number}
/collections/{call_number}/{component_id}
/repositories
/topics
```

For both `/repositories` and `/topics`, we would obviously need a way to view all of the collections in that repository or with that topic. In our previous finding aids application, these both were implemented with what were essentailly canned searches on controlled values, something like:

```
/search?pi=mss
/search?sulocal=American%20politics%20and%20government
```

That's functional, I wanted to do something a little cleaner this time (obviously) that, and for reasons you'll see in a moment, I wanted a real URI for those logical resources and groupings of content.

There was a usability concern about what the scope of each topic was, and a desire to have a page each repository as well, soemething our previous application lacked, and so I added a template for each of those:

```
/repositories/{repository_id}
/topics/{topic_id}
```

Now that __topics__ and __respositories__ each had unique identifiers, it made sense to overload the `/collections` template with two more filters:

```
/collections/{topic_id}
/collections/{respository_id}
```

In other words, we can take  the ID of a topic or repository and tack it on to the end of `/collections` to filter to the collections. These are much cleaner than the canned search URLs we had before.

In the end, we wound up with the following set of URI templates:

```
/collections                              # All collections
/collections/{call_number}                # Collection with $call_number
/collections/{call_number}/{component_id} # $component of collection with $call_number
/collections/{respository_id}             # All collections in $respository
/collections/{topic_id}                   # All collections with $topic
/repositories                             # All repositories
/repositories/{repository_id}             # $repository
/topics                                   # All topics
/topics/{topic_id}                        # $topic
/names                                    # All names (eac-cpf)
/name/{name_id}                           # $name
```

For any developer who has ever worked with a framework that does routing based on URI patterns, I don't have to tell you that having this up front was priceless.

You'll notice the two `/names` patterns at the bottom. This is largely a stub, waiting for EAC-CPF data. Once some significant data is there, it will be interesting to play around with more patterns, e.g., some ideas:

```
/collections/{name_id} # All collections concerned with $name
/names/{collection_id} # All names in $collection
/names/{topic_id}      # All names related to $topic
```

I think it's fair to say that this exercise was useful not only for prettying up the URIs and informing what our discreet 'pages' should be, but from a usability perspective, we have a very clear idea about what the function of each page should be, and what to should link to what where...or at least a clear set of intentions.

Site as Service and Linked Data
-------------------------------
(use the site as visual aid in this section)

Getting back to our [`/topics`](http://findingaids.princeton.edu/topics) pages, this point of entry into the site makes it possible to browse for every finding aid in the database, and therefore to 'follow your nose' (to quote [EdSu](http://inkdroid.org/journal/2008/01/04/following-your-nose-to-the-web-of-data/)) through our whole dataset. This presented a good opportunity to expose our finding aids as Linked Data. If we  tack [`.rdf`](http://findingaids.princeton.edu/topics.rdf) on to the `topics` URI (this can also be done w/ content negotiation) you'll see that the source of this page is a SKOS file, and that we have not only broader/narrower/related relationships between our local terms (e.g. [`t2`, American History](http://findingaids.princeton.edu/topics/t2)), but `skos:exactMatch` and `skos:closeMatch` relationships between them and LCSH as well.

Similarly, the source of our [`/respositories`](http://findingaids.princeton.edu/repositories) page is a [VCard](http://www.w3.org/TR/vcard-rdf/) file. This doesn't link out to anything, but does provide location information, and, more interestingly, EADs also have a representation in RDF (e.g. [`MC174.rdf`](http://findingaids.princeton.edu/MC174.rdf)) that contains URIs for repositories, creators ([VIAF](http://viaf.org/)), and subject ([id.loc.gov](http://id.loc.gov for subjects) / [VIAF](http://viaf.org/) for name), and typed representations of the time coverage. This representation of EAD in RDF is a bit of a stub. It exposes most of the data-centric elements, but, for example, I had to make up a namespace for DACS to get to different date types of date ranges (bulk, inclusive) modelled correctly. I'm not an archives expert, but would be happy to see a set of best practices for representating archival data (EAD-based or otherwise) in RDF from that community.

You can also tack `.xml` on to the URI and get back the source EAD. And, if you ask the `collections` URI for XML [`/collections.xml`](http://findingaids.princeton.edu/collections.xml), without an EAD identifier, you'll get back the first EAD in the dataset, along with `Link` headers that tell you how to get the next one.

```
< HTTP/1.1 200 OK
< Date: Sat, 03 Nov 2012 15:08:45 GMT
< Server: Jetty/5.1.12 (Linux/3.2.0-29-generic amd64 java/1.6.0_24
< Cache-Control: public
< Content-Type: application/x-solr+xml
< Link: <http://findingaids.princeton.edu/collections?start=2>;rel=next,
    <http://findingaids.princeton.edu/collections?start=1>;rel=self,
    <http://findingaids.princeton.edu/collections?start=0>;rel=prev,
    <http://findingaids.princeton.edu/collections?start=2251>;rel=last,
    <http://findingaids.princeton.edu/collections/C0398>;rel=canonical
< X-exist-id: 15
< Last-Modified: Fri, 19 Oct 2012 06:00:46 GMT
< ...

```

This makes it possible to harvest our whole data set with a pretty simple script; in fact it's how we sync the data between the database and Solr

```
        Start
          |
     GET $uri.xml  <-----------------+
          |                          |
Harvest, Index, whatever             |
          |                          |
  Link w/ rel=next? (T) --> $uri = <(*)>;rel=next
          | (F)
         Done
```

The bare `/collections` and `/names` URIs supports HTTP POST as well, which is how and so all of our interaction with the site is completely RESTful. The data is validated upon receipt, and authentication is obviously required.


Slicing EAD Components into Records
-----------------------------------
Switching gears, I want to talk just briefly about how we turned individual components, many of which were quite brief, into fuller records that make sense to users.

Our working group agreed that every component must have all of the DACS Single Level Minimum elements:

 * identifier
 * title
 * date
 * extent
 * creator
 * scope
 * access/rights
 * language
 * repository

But in the deeper parts of collections, we frequently only have one or two of those elements and without context of the whole collection, there's not a lot to go on, especially when the user arrived at a result via search:

Take this for example:

http://findingaids.princeton.edu/collections/MC174/c00357

Again, if I do the `.xml` trick:

Show http://findingaids.princeton.edu/collections/MC174/c00357.xml

From the native data I wouldn't know, for example what collection this came from, who the creator was, I have no rights information, and I know where the collection is located.

The solution was to enhance the records on the fly by looking up in the component tree and/or making calculations based on the descendant components. Adding [scope=enhanced](http://findingaids.princeton.edu/collections/MC174/c00357.xml?scope=enhanced) to the URI shows the result: a fuller record that we use internally, with many added elements, some of which are in the EAD namesspace, others of which are in our own (the ability to show this XML is just a debugging tool; it's really just a stage in the pipeline before we serialize as HTML or XML for Solr). Some of these elements are for display only, while others are for Solr only, but most serve both display and index purposes. You'll see we've added, for example, all of the required DACS element and links to the parent and children components and collection level records.

If I had this to do over...I think we wouldn't have bothered with EAD internally at all, and just used DC or maybe some other generic XML--maybe something that just maps to DACS, since that really was, ultimately, our internal data model--and just kept the EAD as a transmission format. Mucking with the different namespaces and a lot of superfluous structure in the XML really turned out to be more trouble than it was worth.

~~One more tiny but important feature: the tree along the left is driven by a separate service that can be used to get the outline (title + date) of a whole EAD as `json` or `xml`, for example [An outline of MC174 as json](http://findingaids.princeton.edu/_services/tree.json?id=MC174). When [asking for XML](http://findingaids.princeton.edu/_services/tree.xml?id=MC174) the record is returned as an `ead:arrangement` element, which means we don't have hard-coded outlines in our EADs and [pages like this that show users how collections are arranged](http://findingaids.princeton.edu/collections/C0910#arrangement) are generated dynamically.~~

Tech stack
----------
(Skip this unless someone asks, talk about eXist, pros and cons).
