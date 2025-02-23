---
title: I hate untyped code
date: 2025-01-05
tags:
  - thoughts
publish: true
---
Hate is a strong word here, I'm very frustrated when I have to read a piece of code that is untyped. 

Every time I have to work with a large codebase in: python / javascript with a fraction of that code being typed, I'm just annoyed that almost every little change begins with an investigation: what this thing called `data` *really* is.

Sure, some may say that this example is an exaggerated, but I personally found much worse ones throughout my career. 

How do I deal with them? I 🙏 that someone wrote a unit test for this function. The older I am, I'm more convinced that the main reason for unit tests is documentation. 

What If there's no unit test? I read code that manipulated this object or check the runtime, whichever seems faster. 

I think that it comes down to Cognitive load[^1] . Finding *what something is* and building a map of all the references and connections is tiresome. Type systems help to deal with that. 

Like everything, adding types has its disadvantages. But I think (as of 2025) that the advantages outweigh the disadvantages. Who knows? Maybe I change my mind

Feb 25 update: Ok I changed my mind a bit, maybe it's not the lack of types that frustrated my, but lack of documentation: why you're manipulating this data? How does it look like? Even copy-pasted json file is better than nothing.

[^1]: Great article about cognitive load https://minds.md/zakirullin/cognitive 