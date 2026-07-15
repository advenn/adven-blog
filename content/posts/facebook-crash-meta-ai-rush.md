---
date: '2026-07-15T13:45:00+02:00'
draft: false
title: 'Facebook Loaded Itself to Death and Almost Took My Machine With It'
tags: ['meta', 'facebook', 'ai', 'chrome']
---

I almost don't use facebook, but recently, for its marketplace I had to.

It happened several times that, the landing page silently leaked memory, and at the end killed by earlyoom.

The tab opened `facebook.com/?_rdr` and then just... sat there. The little logo kept spinning, and spinning, till the underlying process was killed.

![Facebook stuck on its loading logo](/photos/fb-loading.png)

I pulled up the system monitor. 15.3 GB of 16 GB of RAM is occupied. Swap half full.

![Memory climbing to 92.9%](/photos/mem-climbing.png)

Still spinning.

![Still stuck a minute later](/photos/fb-still-loading.png)

You can actually watch the moment it ends. The memory line climbs, touches ~95%, and drops straight off a cliff. That's earlyoom, deciding heavy processes had to die so the rest of the system could live.

![Memory drops as the process gets killed](/photos/mem-oom-drop.png)

And that was that.

![Aw, Snap! Error code 4](/photos/aw-snap.png)

This isn't someone's weekend side project. It's Facebook. A redirect loop that leaks gigabytes until it kills your browser is the kind of thing you catch in week one.

Recently there were many news about Meta pulling engineers from teams and forcing them to work for AI, training, labeling, creating test sets... But somebody used to own the boring stuff. Reassign those people and this is what's left: a homepage that can't load itself.

Maybe I'm mistaken here, maybe i am concluding early or wrong. But it happened many times in the last days.

Meta, please return those engineers back to their jobs, i can't login to facebook.
