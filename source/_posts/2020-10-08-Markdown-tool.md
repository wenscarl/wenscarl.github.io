---
layout: post
title: Markdown syntax supported by the document
date: 2017-11-20 
tags: markdown    
---


## What is Markdown? 

## What Is It?

Markdown is a plain text formatting syntax aimed at making writing  for the internet easier. The philosophy behind Markdown is that plain  text documents should be readable without tags mussing everything up,  but there should still be ways to add text modifiers like lists, bold,  italics, etc. It is an alternative to WYSIWYG (what you see is what you  get) editors, which use rich text that later gets converted to proper  HTML.

It’s possible you’ve encountered Markdown without realizing it.  Facebook chat, Skype, and Reddit all let you use different flavors of  Markdown to format your messages.

Here’s a quick example: to make words bold using Markdown, you simply enclose them in * (asterisks). So, *bold word* would look like **bold word** when everything is said and done.

All told, Markdown is a great way to write for the web using plain text.

## Basic grammers

Title            
H1 :# Header 1            
H2 :## Header 2           
H3 :### Header 3           
H4 :#### Header 4           
H5 :##### Header 5            
H6 :###### Header 6      
Link :[Title](URL)        
Bold :**Bold**        
Italics :*Italics*         
*strikebreak :~~text~~          
inlines : `alert('Hello World');`        

### List

* list1
* list2
* list3

### list reference

>* list1
>* list2
>* list3


## Advanced syntax

### 1. Todo list 
- [ ] support for exporting the text in PDF format
- [ ] Improving the Cmd rendering algorithm to improve rendering efficiency using local rendering techniques
- [x] New Todo List Feature
- [x] Fix Icon function
### 2. Highliget a snippet of code

```python

@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'

```

#### 3. Table

| item        | price   |  quantity  |
| --------   | -----:  | :----:  |
| computer     | \$1600 |   5     |
| cellphone        |   \$12   |   12   |
| cable        |    \$1    |  234  |


