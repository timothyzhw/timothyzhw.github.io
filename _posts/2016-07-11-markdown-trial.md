---
layout:     post
title:      "MARKDOWN示例教程"
subtitle:   "简单够用就好"
date:       2016-07-10
author:     "Tim"
header-img: "img/post/bg-window.jpg"
catalog: true
tags:
    - jekyll
    - markdown
---
> MARKDOWN 示例 from Mou

<link rel="stylesheet" href="//cdn.bootcss.com/highlight.js/9.4.0/styles/github.min.css" >
<script src="//cdn.bootcss.com/highlight.js/9.5.0/highlight.min.js"></script>
<style>
.div_pre .anchorjs-link{display:none !important;}
#div_org pre{height:800px;overflow-y:auto;}
#div_pre .hll{height:800px;overflow-y:auto;}
#div_org,#div_pre {padding-left:0;padding-right:0;}
</style>
<script type="text/javascript">
	hljs.initHighlightingOnLoad();
	$(function(){
		var s1 = $("#div_org>pre:first")[0];
		var s2 = $("#div_pre .hll:first")[0];
		$(s1).scroll(function(){

			 s2.scrollTop = s1.scrollTop / s1.scrollHeight * s2.scrollHeight;
		});
	});
</script>

## 前言

一篇博客要写的图文并茂，真的不是一件容易的事儿，尤其是像我这样毫无美感的理工男。
_工欲善其事,必先利其器。_ 一个好的编辑器是必须的，windows下推荐Aton，mac就用Mou吧。

本文中的markdown的示例，来自Mou编辑器。能把这些基本的“融会贯通”，就够折腾一阵子了。

## 示例

<div class='row'>
<div id='div_org' class="col-md-6 col-xs-12" >
<pre><code class="markdown">
# Mou

![Mou icon](/img/post/7-10-markdown/Mou_128.png)

## Overview

**Mou**, the missing Markdown editor for *web developers*.

### Syntax

#### Strong and Emphasize

**strong** or __strong__  

*emphasize* or _emphasize_  

**Sometimes I want a lot of text to be bold.
Like, seriously, a _LOT_ of text**

#### Blockquotes

> Right angle brackets &gt; are used for block quotes.

#### Links and Email

An email <example@example.com> link.

Simple inline link <http://chenluois.com>,
another inline link [Smaller](http://25.io/smaller/),
one more inline link with title
[Resize](http://resizesafari.com "a Safari extension").

A [reference style][id] link. Input id,
then anywhere in the doc,
define the link with corresponding id:

[id]: http://25.io/mou/ "Markdown editor on Mac OS X"

Titles ( or called tool tips ) in the links are optional.

#### Images

An inline image ![Smaller icon](http://25.io/smaller/favicon.ico "Title here"),
title is optional.

A ![Resize icon][2] reference style image.

[2]: http://resizesafari.com/favicon.ico "Title"

#### Inline code and Block code

Inline code are surround by `backtick` key. To create a block code:

	Indent each line by at least 1 tab, or 4 spaces.
    var Mou = exactlyTheAppIwant;

####  Ordered Lists

Ordered lists are created using "1." + Space:

1. Ordered list item
2. Ordered list item
3. Ordered list item

#### Unordered Lists

Unordered list are created using "*" + Space:

* Unordered list item
* Unordered list item
* Unordered list item

Or using "-" + Space:

- Unordered list item
- Unordered list item
- Unordered list item

#### Hard Linebreak

End a line with two or more spaces will create a hard linebreak,
called `<br />` in HTML.
Above line ended with 2 spaces.

#### Horizontal Rules

Three or more asterisks or dashes:

***

---

- - - -

#### Headers

Setext-style:

This is H1
==========

This is H2
----------

atx-style:

# This is H1
## This is H2
### This is H3
#### This is H4
##### This is H5
###### This is H6


### Extra Syntax

#### Footnotes

Footnotes work mostly like reference-style links.
A footnote is made of two things:
a marker in the text that will become a superscript number;
a footnote definition that will be placed
in a list of footnotes at the end of the document.
A footnote looks like this:

That's some text with a footnote.[^1]

[^1]: And that's the footnote.


#### Strikethrough

Wrap with 2 tilde characters:

~~Strikethrough~~


#### Fenced Code Blocks

Start with a line containing 3 or more backticks,
and ends with the first line with the same number of backticks:

```
Fenced code blocks are like Stardard Markdown’s regular code
blocks, except that they’re not indented and instead rely on
a start and end fence lines to delimit the code block.
```

#### Tables

A simple table looks like this:

First Header | Second Header | Third Header
------------ | ------------- | ------------
Content Cell | Content Cell  | Content Cell
Content Cell | Content Cell  | Content Cell

If you wish, you can add a leading and tailing pipe to each line of the table:

| First Header | Second Header | Third Header |
| ------------ | ------------- | ------------ |
| Content Cell | Content Cell  | Content Cell |
| Content Cell | Content Cell  | Content Cell |

Specify alignment for each column by adding colons to separator lines:

First Header | Second Header | Third Header
:----------- | :-----------: | -----------:
Left         | Center        | Right
Left         | Center        | Right


</code></pre>

</div>

<div id='div_pre'  class="col-md-6 col-xs-12 highlight ignore-cat">
<div class="hll" >
	<div class="hljs">

  <h1 id="mou">Mou</h1>

 <p><img src="/img/post/7-10-markdown/Mou_128.png" alt="Mou icon" /></p>

 <h2 id="overview">Overview</h2>

 <p><strong>Mou</strong>, the missing Markdown editor for <em>web developers</em>.</p>

 <h3 id="syntax">Syntax</h3>

 <h4 id="strong-and-emphasize">Strong and Emphasize</h4>

 <p><strong>strong</strong> or <strong>strong</strong></p>

 <p><em>emphasize</em> or <em>emphasize</em></p>

 <p><strong>Sometimes I want a lot of text to be bold.<br />
 Like, seriously, a <em>LOT</em> of text</strong></p>

 <h4 id="blockquotes">Blockquotes</h4>

 <blockquote>
   <p>Right angle brackets &gt; are used for block quotes.</p>
 </blockquote>

 <h4 id="links-and-email">Links and Email</h4>

 <p>An email <a href="&#109;&#097;&#105;&#108;&#116;&#111;:&#101;&#120;&#097;&#109;&#112;&#108;&#101;&#064;&#101;&#120;&#097;&#109;&#112;&#108;&#101;&#046;&#099;&#111;&#109;">&#101;&#120;&#097;&#109;&#112;&#108;&#101;&#064;&#101;&#120;&#097;&#109;&#112;&#108;&#101;&#046;&#099;&#111;&#109;</a> link.</p>

 <p>Simple inline link <a href="http://chenluois.com">http://chenluois.com</a>,<br />
 another inline link <a href="http://25.io/smaller/">Smaller</a>,<br />
 one more inline link with title<br />
 <a href="http://resizesafari.com" title="a Safari extension">Resize</a>.</p>

 <p>A <a href="http://25.io/mou/" title="Markdown editor on Mac OS X">reference style</a> link. Input id,<br />
 then anywhere in the doc,<br />
 define the link with corresponding id:</p>

 <p>Titles ( or called tool tips ) in the links are optional.</p>

 <h4 id="images">Images</h4>

 <p>An inline image <img src="http://25.io/smaller/favicon.ico" alt="Smaller icon" title="Title here" />,<br />
 title is optional.</p>

 <p>A <img src="http://resizesafari.com/favicon.ico" alt="Resize icon" title="Title" /> reference style image.</p>

 <h4 id="inline-code-and-block-code">Inline code and Block code</h4>

 <p>Inline code are surround by <code>backtick</code> key. To create a block code:</p>

 <pre><code>Indent each line by at least 1 tab, or 4 spaces.
 var Mou = exactlyTheAppIwant;
 </code></pre>

 <h4 id="ordered-lists">Ordered Lists</h4>

 <p>Ordered lists are created using “1.” + Space:</p>

 <ol>
   <li>Ordered list item</li>
   <li>Ordered list item</li>
   <li>Ordered list item</li>
 </ol>

 <h4 id="unordered-lists">Unordered Lists</h4>

 <p>Unordered list are created using “*” + Space:</p>

 <ul>
   <li>Unordered list item</li>
   <li>Unordered list item</li>
   <li>Unordered list item</li>
 </ul>

 <p>Or using “-“ + Space:</p>

 <ul>
   <li>Unordered list item</li>
   <li>Unordered list item</li>
   <li>Unordered list item</li>
 </ul>

 <h4 id="hard-linebreak">Hard Linebreak</h4>

 <p>End a line with two or more spaces will create a hard linebreak,<br />
 called <code>&lt;br /&gt;</code> in HTML.<br />
 Above line ended with 2 spaces.</p>

 <h4 id="horizontal-rules">Horizontal Rules</h4>

 <p>Three or more asterisks or dashes:</p>

 <hr />

 <hr />

 <hr />

 <h4 id="headers">Headers</h4>

 <p>Setext-style:</p>

 <h1 id="this-is-h1">This is H1</h1>

 <h2 id="this-is-h2">This is H2</h2>

 <p>atx-style:</p>

 <h1 id="this-is-h1-1">This is H1</h1>

 <h2 id="this-is-h2-1">This is H2</h2>

 <h3 id="this-is-h3">This is H3</h3>

 <h4 id="this-is-h4">This is H4</h4>

 <h5 id="this-is-h5">This is H5</h5>

 <h6 id="this-is-h6">This is H6</h6>

 <h3 id="extra-syntax">Extra Syntax</h3>

 <h4 id="footnotes">Footnotes</h4>

 <p>Footnotes work mostly like reference-style links.<br />
 A footnote is made of two things:<br />
 a marker in the text that will become a superscript number;<br />
 a footnote definition that will be placed<br />
 in a list of footnotes at the end of the document.<br />
 A footnote looks like this:</p>

 <p>That’s some text with a footnote.<sup id="fnref:1"><a href="#fn:1" class="footnote">1</a></sup></p>

 <h4 id="strikethrough">Strikethrough</h4>

 <p>Wrap with 2 tilde characters:</p>

 <p>~~Strikethrough~~</p>

 <h4 id="fenced-code-blocks">Fenced Code Blocks</h4>

 <p>Start with a line containing 3 or more backticks,<br />
 and ends with the first line with the same number of backticks:</p>

 <pre><code>Fenced code blocks are like Stardard Markdown’s regular code
 blocks, except that they’re not indented and instead rely on
 a start and end fence lines to delimit the code block.
 </code></pre>

 <h4 id="tables">Tables</h4>

 <p>A simple table looks like this:</p>

 <table>
   <thead>
     <tr>
       <th>First Header</th>
       <th>Second Header</th>
       <th>Third Header</th>
     </tr>
   </thead>
   <tbody>
     <tr>
       <td>Content Cell</td>
       <td>Content Cell</td>
       <td>Content Cell</td>
     </tr>
     <tr>
       <td>Content Cell</td>
       <td>Content Cell</td>
       <td>Content Cell</td>
     </tr>
   </tbody>
 </table>

 <p>If you wish, you can add a leading and tailing pipe to each line of the table:</p>

 <table>
   <thead>
     <tr>
       <th>First Header</th>
       <th>Second Header</th>
       <th>Third Header</th>
     </tr>
   </thead>
   <tbody>
     <tr>
       <td>Content Cell</td>
       <td>Content Cell</td>
       <td>Content Cell</td>
     </tr>
     <tr>
       <td>Content Cell</td>
       <td>Content Cell</td>
       <td>Content Cell</td>
     </tr>
   </tbody>
 </table>

 <p>Specify alignment for each column by adding colons to separator lines:</p>

 <table>
   <thead>
     <tr>
       <th style="text-align: left">First Header</th>
       <th style="text-align: center">Second Header</th>
       <th style="text-align: right">Third Header</th>
     </tr>
   </thead>
   <tbody>
     <tr>
       <td style="text-align: left">Left</td>
       <td style="text-align: center">Center</td>
       <td style="text-align: right">Right</td>
     </tr>
     <tr>
       <td style="text-align: left">Left</td>
       <td style="text-align: center">Center</td>
       <td style="text-align: right">Right</td>
     </tr>
   </tbody>
 </table>
 <div class="footnotes">
   <ol>
     <li id="fn:1">
       <p>And that’s the footnote. <a href="#fnref:1" class="reversefootnote">&#8617;</a></p>
     </li>
   </ol>
 </div>


</div>
</div>

</div>
</div>
