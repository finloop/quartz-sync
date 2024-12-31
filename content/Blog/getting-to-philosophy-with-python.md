---
title: Getting to Philosophy With Python
date: 2021-09-20T00:41:41+02:00
publish: true
tags:
  - python
  - programming
---

There's a famous Wikipedia phenomena that by clicking the first link in the main text of the article on the English Wikipedia, you'll eventually end up on the [philosophy](https://en.wikipedia.org/wiki/Philosophy) page. An explanation can be found [here](https://en.wikipedia.org/wiki/Wikipedia:Getting_to_Philosophy). Briefly, it's because of Wikipedia [Manual of Style guidelines](https://en.wikipedia.org/wiki/Wikipedia:MOSBEGIN) that recommend that articles begin by telling "what or who the subject is, and often when and where".

This was true for roughly 97% of articles, so there's a big chance that by entering a random Wikipedia page and following the procedure you'll indeed end up on Philosophy. I could test this by hand, but this wouldn't be a dev.to article without writing some code. We'll start with how to download Wikipedia articles.

## How to get data

It's simple - just request contents of and article with `urllib3`. Wikipedia follows a convenient pattern for naming its articles. After the usual `en.wikipedia.org/` there's a `/wiki` and then `/article_name` (or media! we'll deal with that later) for example, `en.wikipedia.org/wiki/Data_mining`.

Firstly, I'll create a pool from which I'll make requests to Wikipedia.

```python
import urllib3
from bs4 import BeautifulSoup

pool = urllib3.PoolManager()
```

From now on, I'll could download the articles one by one. To automate the process of crawling through the site, the crawler will be recursive. Each iteration of it will return `(current_url, [crawler for link_on site])`, the recursion will stop, at given depth. In the end, I'll end up with tree structure.

```python
def crawl(
    pool: urllib3.PoolManager,
    url,
    phrase=None,
    deep=1,
    sleep_time=0.5,
    n=5,
    prefix="https://en.wikipedia.org",
    verbose=False,
):
    """
    Crawls given Wikipedia `url` (article) with max depth `deep`. For each page
    extracts `n` urls and  if `phrase` is given check if `phrase` in urls.

    Parameters
    ----------
    pool : urllib3.PoolManager
        Request pool
    phrase : str
        Phrase to search for in urls.
    url : str
        Link to wikipedia article
    deep : int
        Depth of crawl
    sleep_time : float
        Sleep time between requests.
    n : int
        Number of links to return
    prefix : str, default="https://en.wikipedia.org""
        Site prefix

    Returns
    -------
    tuple
        Tuple of url, list
    """
    if verbose:
        site = url.split("/")[-1]
        print(f"{deep} Entering {site}")

    # Sleep to avoid getting banned
    time.sleep(sleep_time)
    site = pool.request("GET", url)
    soup = BeautifulSoup(site.data, parser="lxml")

    # Get links from wiki (I'll show it later)
    links = get_links_from_wiki(soup=soup, n=n, prefix=prefix)

    # If phrase was given check if any of the links have it
    is_phrase_present = any([phrase in link for link in links]) and phrase is not None
    if deep > 0 and not is_phrase_present:
        return (
            url,
            [
                crawl(
                    pool=pool,
                    url=url_,
                    phrase=phrase,
                    deep=deep - 1,
                    sleep_time=sleep_time,
                    n=n,
                    prefix=prefix,
                    verbose=verbose,
                )
                for url_ in links
            ],
        )
    return url, links
```

If you read the code carefully, you'd notice a function `get_links_from_wiki`. `get_links_from_wiki` function parses the article. It works by finding a div that contains the whole article, then iterates through all paragraphs (or lists) and finds all links that match pattern `/wiki/article_name`. Because there's no domain in that pattern, it is added at the end.

```python
def get_links_from_wiki(soup, n=5, prefix="https://en.wikipedia.org"):
    """
    Extracts `n` first links from wikipedia articles and adds `prefix` to
    internal links.

    Parameters
    ----------
    soup : BeautifulSoup
        Wikipedia page
    n : int
        Number of links to return
    prefix : str, default="https://en.wikipedia.org""
        Site prefix
    Returns
    -------
    list
        List of links
    """
    arr = []

    # Get div with article contents
    div = soup.find("div", class_="mw-parser-output")

    for element in div.find_all("p") + div.find_all("ul"):
        # In each paragraph find all <a href="/wiki/article_name"></a> and
        # extract "/wiki/article_name"
        for i, a in enumerate(element.find_all("a", href=True)):
            if len(arr) >= n:
                break
            if (
                a["href"].startswith("/wiki/")
                and len(a["href"].split("/")) == 3
                and ("." not in a["href"] and ("(" not in a["href"]))
            ):
                arr.append(prefix + a["href"])
    return arr
```

Now we have everything to check the phenomena. I'll set max depth to 50 and set `n=1` (to only expand first link in the article).

```python
crawl(pool, "https://en.wikipedia.org/wiki/Doggart", phrase="Philosophy", deep=50, n=1, verbose=True)
```

Output:

```output
50 Entering Doggart
49 Entering Caroline_Doggart
48 Entering Utrecht
47 Entering Help:Pronunciation_respelling_key
...
28 Entering Mental_state
27 Entering Mind
26 Entering Thought
25 Entering Ideas

('https://en.wikipedia.org/wiki/Doggart',
 [('https://en.wikipedia.org/wiki/Caroline_Doggart',
   [('https://en.wikipedia.org/wiki/Utrecht',
     [('https://en.wikipedia.org/wiki/Help:Pronunciation_respelling_key',
       [('https://en.wikipedia.org/wiki/Pronunciation_respelling_for_English',

...
                                                 [('https://en.wikipedia.org/wiki/Ideas',
                                                   ['https://en.wikipedia.org/wiki/Philosophy'])])])])])])])])])])])])])])])])])])])])])])])])])])
```

As you can see after 25 iterations indeed we found `Philosophy` page.

Found this post interesting? Check out my [Github](https://github.com/finloop) @finloop
