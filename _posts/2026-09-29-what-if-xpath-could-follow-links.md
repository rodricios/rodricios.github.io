---
layout: post
title: "What if XPath could follow links?"
date: 2026-09-29
author: Rodrigo Palacios
description: "The idea behind wxpath: use XPath to describe both what to extract from a page and which pages to visit next."
excerpt: "XPath can walk a page. A crawler can walk links. I wanted one expression that could do both."
tags: [wxpath, XPath, web crawling, data extraction]
categories: [Python, Data Extraction]
---

In 2015, I wrote about [extracting data from a web page in ten lines of Python]({{ '/2015/data-extraction-in-10-lines-of-python.html' | relative_url }}). The problem I was poking at was the amount of machinery we put around a fairly simple question: *where is the data on this page?*

XPath was already a good answer to that question. Give it an HTML tree and it can point at the links, the titles, the repeated rows, or whatever else you came for.

But a website isn't one tree. It's a graph of documents connected by links. XPath stops at the edge of the current document, and then we write a crawl loop to cross that edge. Fetch a page, find a link, resolve it, fetch the next page, check whether we've seen it before, and do it all again. If you've scraped more than one page, you've probably written some version of this loop. I certainly have.

So here's the question behind [wxpath](https://github.com/rodricios/wxpath): **what if the path expression could cross that edge too?**

## A URL as part of the path

I added a `url(...)` operator to an XPath-like expression. A literal URL starts the job:

```text
url('https://quotes.toscrape.com')//a/@href
```

That says: fetch the page, then select its links. So far, this is just a request followed by XPath. The interesting part is that `url(...)` can also receive links selected *from the page you've just fetched*. It turns those links into the next documents to query.

The mental model I use in the [language design doc](https://github.com/rodricios/wxpath/blob/master/DESIGN.md) is simple:

- A document is a node in the web graph.
- A hyperlink is a directed edge to another node.
- XPath chooses things *inside* a document.
- `url(...)` moves the expression *between* documents.

In other words, I wanted to describe the crawl and the extraction in the same place, instead of writing the control flow first and attaching an XPath expression afterward.

## Let's paginate some quotes

Say I want the author and text of every quote on a set of pages. The pages have a "next" link. Here's the expression:

```python
import wxpath

expr = """
url('https://quotes.toscrape.com/tag/humor/',
    follow=//li[@class='next']/a/@href)
  //div[@class='quote']
    /map{
      'author': (./span/small/text())[1],
      'text': (./span[@class='text']/text())[1]
    }
"""

for quote in wxpath.wxpath_async_blocking_iter(expr, max_depth=3):
    print(quote)
```

The first line seeds the crawl. `follow=` says which link to follow on each visited page. The rest says what to extract from each page: find the quote blocks and build a map with an author and text. `max_depth` puts a bound on the traversal.

There is no handwritten "while next page exists" loop here. The engine schedules requests, deduplicates URLs on a best-effort basis, and yields results as they arrive. Because requests can run concurrently, I wouldn't expect the results to arrive in page order. That's a trade I'd happily make for many extraction jobs.

If you don't need a separate `follow=` rule, `///url(xpath)` is the deep-crawl form: select links on the current page, enqueue them, and repeat on the pages that come back. I call that *recursive* crawling in the design doc, but it isn't a recursive Python function walking depth-first. The engine uses a queue and works breadth-first-ish.

Why use `follow=` for the quote example? Because I want to extract quotes from the starting page *and* follow its next link. A postfixed `///url(...)` applies the extraction after that hop; `follow=` lets the seed page participate too.

## The part that wasn't just syntax

It's tempting to think of `url()` as another XPath function. It isn't quite. An ordinary XPath expression can be evaluated against every matching node in a document. Network requests are too expensive, and too consequential, for that interpretation to be useful here. In wxpath, **URL selection is evaluated per document, not once per matched DOM node**.

That rule forced me to be explicit about how XPath pieces join across a `url(...)` boundary, and which combinations should be rejected. The [design doc](https://github.com/rodricios/wxpath/blob/master/DESIGN.md) goes into the grammar, rewrite rules, and the cases I was still arguing with myself about. I think that's the interesting part of designing a small language: the easy example takes a minute; deciding what *every* expression means takes longer.

## What it is good for, and where it stops

wxpath is deterministic in a useful sense: you choose the links and the fields with an expression, and the engine follows those rules. The web itself can still change underneath you. A missing page, a changed layout, or a new pagination pattern can change the result, just as with any crawler.

The current [README](https://github.com/rodricios/wxpath#readme) covers the Python API, CLI, terminal interface, caching, and the more advanced XPath 3.1 features such as maps. It also names the boundaries: no browser-based JavaScript rendering yet, no strict result ordering, and deep crawls still need sensible XPath predicates and depth limits. Don't point `///url(//a/@href)` at the whole web and act surprised when it tries to visit the whole web.

I like scraping because it sits in an odd place. A page is structured enough that we can query it precisely, but the job surrounding that query is often messy. wxpath is my attempt to make that job expressible: start here, follow *these* links, and give me *this* data.

You can [install wxpath](https://github.com/rodricios/wxpath#install) with `pip install wxpath`. If you're curious about the language decisions, start with the [design doc](https://github.com/rodricios/wxpath/blob/master/DESIGN.md). There are still plenty of edges to work through.
