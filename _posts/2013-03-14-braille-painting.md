---
layout: post
title: Painting with Braille
categories: [Python]
redirect_from: /article/2013/03/braille-painting/
excerpt_separator: <!--more-->
---

![Braille Python Logo](/media/image/2013/03/braille-python.png "Braille Python Logo")

This is something I've been wanting to write for a while.

Unicode page U+2800 has all the combinations of a 2x4 grid of Braille dots. Braille dots that line up neatly with the ones on all sides in most fonts. We can paint with this!

<!--more-->


So I decided to have a little fun and write the whole thing as a generator expression. Let me know if you have any improvements!

[https://gist.github.com/wardi/5131529](https://gist.github.com/wardi/5131529)

```python
D = """
      xxxxxx
     xx  xxxx
     xxxxxxxx
         xxxx
 xxxxxxxxxxxx xxx
xxxxxxxxxxxxx xxxx
xxxxxxxxxxxxx xxxx
xxxx          xxxx
xxxx xxxxxxxxxxxxx
xxxx xxxxxxxxxxxxx
 xxx xxxxxxxxxxxx
     xxxx
     xxxxxxxx
     xxxx  xx
      xxxxxx
"""

from itertools import izip_longest, chain
print u'\n'.join(u''.join(unichr(0x2800 + sum((cell == 'x') * (1 << shift)
    for cell, shift in zip(cells, [0, 3, 1, 4, 2, 5, 6, 7])))
    for cells in izip_longest(*chain(*[[iter(r)] * 2 for r in quad])))
    for quad in izip_longest(*([iter(D.strip('\n').split('\n'))] * 4),
    fillvalue='')).encode('utf-8')
```

(As mentioned above, I don't normally code this way)
