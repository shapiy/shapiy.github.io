---
title: "Implementing an XSLT-like transformation in bare Clojure"
published: true
categories: [blog, software engineering]
tags: [clojure, xslt, xml, poc]
---

![Ronchamp](/assets/Ronchamp.jpg)

Chapelle Notre-Dame-du-Haut de Ronchamp is one of the greatest buildings 
of the 20th century. Being architecturally complex, it remains visually simple.
To achieve artistic objectives, the creator might have picked more elements, 
but he didn't.

# Rationale

XSLT is a powerful programming language with a single purpose: to 
transform XML documents. The idea may sound confusing at first: why do 
we even need a separate technology to deal with XML transformations? 
After all, every modern programming language has an XML library. 

DOM manipulations are good for simple documents — but [how do you 
make a web page](https://www.dita-ot.org) out of a 1G source file 
with deep hierarchy and hundreds of various tags? XSLT solves the problem by small means: 

- Declarative approach
- XPath based pattern matching
- A library of instructions and functions to manipulate DOM

I practiced XSLT for a couple of years. Every time I used it, I
was getting more impressed of how well it was designed in its essence. 
However, versions 2.0 and 3.0 of the spec have brought more advanced tooling such as: 

- Complex XPath expressions
- High-order functions 
- XML streaming

The hard work behind the newer specs and reference implementations 
prompted doubts: is there a better way? Why should we strive so hard to pull 
more functionality from general purpose programming languages when there
are, well, general purpose programming languages? ¯\_(ツ)_/¯

In a series of blog posts, we'll discover how a modern functional programming
language like [Clojure](https://clojure.org) can be used to build XML 
transformations. Similarly to XSLT, Clojure has an elementary syntax (XSLT's 
syntax is all XML). The difference is that the capacities are enormous. Moreover, 
Clojure runs on JVM and supports Java interop.

In this first article, we'll develop a simple XML transformation in Clojure, 
taking ground ideas from XSLT. The upcoming articles will expand on the topic. 

# Learn XSLT in five minutes 

What is XSLT? Basically:

1. The source file
1. The stylesheet
1. The result file

The central building blocks of stylesheets are `xsl:template` instructions.
They tell how nodes corresponding to certain XPaths must be converted.

This is how you can map all `li` elements to plain-text bullet points:

```xml
<xsl:template match="li">
    <xsl:text>• </xsl:text>
    <xsl:value-of select="."/>
</xsl:template>
```

XSLT can be used to produce any text output: PDF, Markdown, anything. But if you do XML in, XML out, 
you probably start with _identity transform_. The identity transform is nothing more 
but a stylesheet to transform the source XML into itself. There are variations of this idiom,
one of them is this:

```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

    <xsl:template match="@*|node()">
        <xsl:copy>
            <xsl:apply-templates select="@*|node()"/>
        </xsl:copy>
    </xsl:template>

</xsl:stylesheet>
```

Let's see what's new here:  

- `@*|node()` XPath selects all attributes `@*` and nodes in their input order.
- `xsl:copy` copies the current node
- `xsl:apply-templates` applies template rules for the inner nodes, if any

Congratulations! You've just learnt XSLT. There are secondary language constructs,
but the main point is this. XSLT is similar to CSS in its declarative nature.

# The problem

Now that you're familiar with XSLT, let's imagine we have an HTML file:

```html
<!DOCTYPE html>
<html>
<body>
<h1>Programming languages</h1>
<p>There are many, for example:</p>
<ol>
    <li>Clojure (<a href="http://clojure.org">official site</a>)</li>
    <li>XSLT (<a href="http://www.w3.org/Style/XSL/">W3C page</a>)</li>
</ol>
</body>
</html>
```

It is pretty nice, but we'd like to replace the anchors with `https://`
links. 

First of all, we have to switch to XSLT 2.0 that supports string `replace()` function 
(ouch!). Second, we'll add another template to the identity transform. 
`xsl:attribute` instruction can be used to generate attributes. 

```xml
<xsl:template match="@href">
    <xsl:attribute name="href" select="replace(., 'http://', 'https://')"/>
</xsl:template>
```

All links look relevant now:

```xml
<a href="https://www.w3.org/Style/XSL/">W3C page</a>
```

If I was not used to XSLT, I would also notice how verbose things become: repeated 
XML tags, long instruction names (`xsl:chose`/`xsl:when`/`xsl:otherwise` triad being my favourite).
Let us see if Clojure can help.

# Closer to the mark with Clojure

You have many options when it comes to XML manipulations in Clojure: 

- You can parse XML with [data.xml](https://github.com/clojure/data.xml)
- or convert XML into a Clojure sequence using [xml-seq](https://clojuredocs.org/clojure.core/xml-seq)
- or even construct a functional [zipper](https://clojure.github.io/data.zip/)
  
…and it is not even close to an end.

These approaches have one thing in common: they are imperative. XSLT, on the other hand,
is strictly declarative (well, almost). Still, it can be used to build transformations of 
incredible complexity. And we want to build something similar in Clojure.

We'll start with identity transform and update it to match the demonstrated XSLT stylesheet. 

`copy` function imitates the `xsl:copy` instruction. It accepts a node and 
a mapping function applied on the node attributes and children.

```clojure
(defn copy [node f]
  (cond
    ;; Element?
    (.isInstance Element node) [(:tag node)
                                (f (:attrs node))
                                (f (:content node))]
    ;; Attribute map?
    (map? node) node
    ;; Text node
    (string? node) node))
```

Next comes a simple `apply-templates` helper. Given a node or a list of nodes,
it invokes `map-identity` on each one.

```clojure
(defn apply-templates [node-or-nodes]
  (if (seq? node-or-nodes)
    (map map-identity node-or-nodes)
    (map-identity node-or-nodes)))
```

`map-identity` is even simpler: given a node, it invokes the `copy` function.
The `apply-templates` invoked against the node's children will ensure that the input
tree is processed recursively.

```clojure
(defn map-identity [node] (copy node apply-templates))
```

This is identical to:

```xml
<xsl:copy>
  <xsl:apply-templates select="@*|node()"/>
</xsl:copy>
```

We're almost done. To put it all together, we'll parse XML string as sexp-like
data structure, apply the transformation and convert the result back to an
XML string. 

```clojure
(defn transform []
  (-> (parse-str html)
      apply-templates
      sexp-as-element
      indent-str))
```

The source code: [http://bit.ly/328u6Fx](http://bit.ly/328u6Fx)

Notice we don't have any templates yet. No notion of stylesheet either. The declarative
part is mixed with imperative. So, let us fix it and do the `http` to `https`
replacement along the way. To achieve that, let's define a simple rule dispatcher
based on the node type.

```clojure
(defn element? [node] (.isInstance Element node))
(defn attrs? [node] (and (map? node) (not (element? node))))
(defn text? [node] (string? node))

(defn stylesheet [node]
  (condp #(%1 %2) node
    element? map-identity
    attrs? map-attrs
    text? map-identity))
```

We also want to update `apply-templates` to use the new stylesheet. 

```clojure
(defn apply-template [node]
  (let [template (stylesheet node)]
    (template node)))

(defn apply-templates [node-or-nodes]
  (if (seq? node-or-nodes)
    (map apply-template node-or-nodes)
    (apply-template node-or-nodes)))
```

The only thing left is to implement the attribute mapping function:

```clojure
(defn map-attrs [attrs]
  (if (contains? attrs :href)
    (let [https-url (clojure.string/replace (:href attrs) #"http://" "https://")]
      (assoc attrs :href https-url))
    attrs))
```

The source code: [http://bit.ly/2JHQKOQ](http://bit.ly/2JHQKOQ)

# Conclusions

In this article, we provided a brief intro to the XSLT programming language. We discussed 
its core ideas and used it to transform an XML document. Also, we reviewed
the limitations of the language and implemented the same conversion 
in a general-purpose functional language. 

Thanks to its simplicity and functional approach, Clojure was a great choice
for the proof of concept. 

Now that we have the basic blocks, we can build on top of that. In the next article, 
we'll implement an HTML to Markdown transform in bare Clojure.

Thanks for your time.