---
layout: post
title: "Data Extraction in 10 lines of Python"
date: 2015-05-16
author: Rodrigo Palacios
image: https://i.imgur.com/mTnYI6y.jpg
tags: [algorithm, data-extraction, Python]
categories: [Python, Reddit, Data Extraction]
excerpt: "Can the data extraction problem be solved in 10 lines of Python? Read and find out!"
---

*[Libextract](https://github.com/datalib/libextract) - extract data from the web (a Python implementation of this work)*.

Update May 22, 2015: Thank you all who are visiting for a second time and helped me reach 
the front page of [r/programming](http://www.reddit.com/r/programming/comments/3664yr/hi_rprogramming_4_months_ago_i_released_a_tiny/)!
It was a fun experience. Please feel free to share :)

---

![Data extraction at it's root](https://i.imgur.com/Xfb8zYE.jpg?2)

In the grand scheme of things, extracting data from the web is a small step in a 
much larger objective. But in the micro, *data extraction* appears to us as a problem
that has no real definitive solution, a sort of *dirty job* - how often is the task
relegated as that of "scraping" the web?

Just think about how many times you've ran into this type of web-scraping tutorial: 
download HTML, parse HTML, open dev tools, flip back-and-forth between browser and 
console, debug your [XPath](http://en.wikipedia.org/wiki/XPath) expressions, etc. 

On the other hand, data extraction via web-scraping has seamlessly, and sometimes elegantly,
found its way into larger projects ([Apache Nutch](http://nutch.apache.org/) includes [Tika](http://tika.apache.org/), 
[Scrapy](http://scrapy.org/) has hybrid crawl-n-scrape architecture). Needless to
say, they're designed to give you - the developer, the data scientist, the hacker, the
one with time to spare - a lot of control... to a fault.

In this post, I present an informal solution to the data extraction problem. It's 
short, it's sweet, and it leaves much room for improvement. Enjoy :) 

## *The algorithm*

In not-so-plain words:

> Given a tree, return a list of all subtrees - sorted by the number
> of children relative to the root node in the subtree.

In Python: 

```python
from requests import get
from collections import Counter
from lxml import html

reddit_request = get('http://www.reddit.com/')
#wiki_request = get('http://en.wikipedia.org/wiki/Information_extraction')

parsed_doc = html.fromstring(reddit_request.content)

# In SQL-like terms: select all parents in <body>
parent_elements = parsed_doc.xpath('//body//*/..')

parents_with_children_counts = []

for parent in parent_elements:
    children_counts = Counter([child.tag for child in parent.iterchildren()])
    parents_with_children_counts.append((parent, children_counts))

# This line, one could say, is what wraps this data-extraction 
# algorithm up as a maximization/optimization algorithm
parents_with_children_counts.sort(# x[1].most_common(1)[0][1] gets the frequency value
                                  key=lambda x: x[1].most_common(1)[0][1], 
                                  reverse=True)
```

## *The Problem*

According to [wikipedia](http://en.wikipedia.org/wiki/Data_extraction): 

> **Data extraction** is the act or process of retrieving data out of 
> (usually unstructured or poorly structured) data sources ... Typical 
> unstructured data sources include web pages ...

The problem I'm attempting to tackle is that of extracting data from 
websites. 

## *What sort of "data" are we extracting?*

Websites are structured by HTML (sure, the styling is also partially 
responsible for the *visual* structure, but as you'll see later on, 
we don't *have* to take style sheets into account). It's in the 
structured HTML where you'll find "unstructured data".

Before I continue, I'll give a bit of background as to why the heck 
I'm even tackling this problem. 

About 4 months ago, I debuted [eatiht](https://github.com/rodricios/eatiht) 
(a text-extracting library; the predecessor to this algorithm) on [reddit](http://www.reddit.com/r/compsci/comments/2ppyot/just_made_what_i_consider_my_first_algorithm_it/).
I can only say positive things about doing so. For one, it's landed me an
opportunity to coauthor a paper with [Tim Weninger](http://www3.nd.edu/~tweninge/). 

It was Tim who introduced me to the *structured [tabular] data extraction*
problem. He illustrated the scenario of trying to extract `<table>`'s and 
table-like structures from a website like [reddit](http://www.reddit.com). 
Here's what he says about that:

> The first [candidate] is the list of subreddits across the top of 
> the page ... The second is the list of reddit posts with link, time, 
> username, and a bunch of other things. The front page of reddit is 
> nothing more than a stylized set of multi-attribute lists or tables. 

## *A wild problem appears!*

Let's get a better picture of what Tim is saying:

![reddit tables](https://i.imgur.com/OsA7Iiyl.png)
*Websites are like a series of tables...*

What's highlighted in red is what is meant by *structured* or *tabular data*. 

Since we're dealing with HTML, let's have a look at the underlying markup: 

![subreddits](https://i.imgur.com/d3cFlB8l.png)
*subreddits - there's a lot of `li`'s*

![top posts](https://i.imgur.com/78rNdf4l.png)
*top posts - there's a lot of `div`'s*

Looking at the above pictures, one thing should be clear: although
`<table>`'s are the epitome of **tabular data**, there are no 
`<table>`'s in the above HTML. But despite the lack of tables, it 
should also be clear that there is tabular data on the front page of Reddit.

So what do? 

## Deconstruction

Clearly define what the problem is: 

***Data** (in the context of HTML) are collections of HTML elements. 
Visually, data is presented as rows or columns (usually). Structurally, 
data is presented as a collection (**parent element**) of **children 
elements**.*

Some of you guys and gals may be thinking, "This definition is ambiguous. 
We're practically talking about every element in an HTML tree."

Yup.

But now throw the phrase *frequently occurring* into the definition:

*Structurally, data is presented as a collection (parent element)
of **frequently occurring** children elements.*

To reiterate clearly (hopefully):

1. We're looking for the *parents* of elements of any tag.

2. We're looking for repetitive elements, or rather the *counts* of 
repetitive elements

In the case of reddit.com, we'd like to create some solution that will 
retrieve at least two collections: the `<ul>` containing those 
`<li>`'s; the parent `<div>` containing those inner `<div>`'s. 

Having said all that, let's take out my prototyping weapon of choice, 
Python, and get to it. 

## Solution

First, let's just get our HTML into a Python-friendly object.

```python
from requests import get

reddit_request = get('http://www.reddit.com/')
# >>> reddit_request.content[:50]
# '<<!doctype html><html xmlns="http://www.w3.org/1999'
```

Now let's process the HTML string through a library ([lxml](http://lxml.de/lxmlhtml.html)) that will take 
that string and create an DOM-traversable, [XPath](http://en.wikipedia.org/wiki/XPath)-queryable object.

```python
from lxml import html

parsed_doc = html.fromstring(reddit_request.content)
```

From here, we can start doing things like querying for different 
types of nodes (HTML elements). But we're not looking for a specific 
*type* of node - where by *type* I mean [*tag*](http://www.w3schools.com/tags/ref_byfunc.asp). 

*We're looking for the **parents** of elements of any tag.*

```python
# In SQL-like terms: select all parents in body
parent_elements = parsed_doc.xpath('//body//*/..')

# >>> parent_elements[:5]
# [<Element body at 0x5a69958>,
#  <Element div at 0x5a699a8>,
#  <Element div at 0x5c37a98>,
#  <Element div at 0x5c37ae8>,
#  <Element div at 0x5c379a8>]
```

And *we're looking for repetitive elements, or rather the **counts** of 
repetitive elements*

```python
from collections import Counter

parents_with_children_counts = []

for parent in parent_elements:
    children_counts = Counter([child.tag for child in parent.iterchildren()])
    parents_with_children_counts.append((parent, children_counts))

# >>> parents_with_children_counts[:5]
# [(<Element body at 0x5a69958>, Counter({'script': 4, 'div': 4, 'a': 1, 'p': 1})),
#  (<Element div at 0x5a699a8>, Counter({'div': 3, 'a': 1})),
#  (<Element div at 0x5c37a98>, Counter({'div': 1})),
#  (<Element div at 0x5c37ae8>, Counter({'div': 3, 'a': 1})),
#  (<Element div at 0x5c379a8>, Counter({'span': 1}))]
```

Finally, let's sort our list of parent-child counter pairs by the *frequency* 
of the most common element in each *child counter*.

```python
# This line, one could say, is what wraps this data-extraction 
# algorithm as a maximization/optimization algorithm
parents_with_children_counts.sort(# x[1].most_common(1)[0][1] gets the frequency value
                                  key=lambda x: x[1].most_common(1)[0][1], 
                                  reverse=True)

# >>> parents_with_children_counts[:5]
# [(<Element div at 0x5f4ae08>, Counter({'a': 51})),
#  (<Element div at 0x5f64048>, Counter({'div': 51})),
#  (<Element ul at 0x5f5f048>, Counter({'li': 48})),
#  (<Element div at 0x5f613b8>, Counter({'div': 23})),
#  (<Element ul at 0x5f60048>, Counter({'li': 8}))]                               
```

## Results

Let's print out the text content from each of the retrieved (extracted) 
elements' children:

```python
>>> [elem.text_content() for elem in parents_with_children_counts[0][0].iterchildren()]
announcements
Art
AskReddit
...
worldnews
WritingPrompts
edit subscriptions
```

So where's that coming from?

![Hidden list of subreddits](https://i.imgur.com/s7W7R4Bl.png)
*Here's where. It's the MY SUBREDDITS button.*

Let's print out the text content of the second and third retrieved elements:

```python
for child in parents_with_children_counts[1][0]:
    print(child.text_content())

# 1732273237324Dad Instincts (share.gifyoutube.com)submitted 4 hours ago...
# 2342634273428Transparency is important to us (self.announcements)...
# ...
```

## Conclusion

And there you have it - a simple algorithm for extracting structured data from web pages by finding parent elements with frequently occurring children. The key insight is that tabular data, whether in actual `<table>` elements or styled `<div>`s, follows a pattern of repetition that we can detect and exploit.

Check out [Libextract](https://github.com/datalib/libextract) for a full Python implementation of this approach!
