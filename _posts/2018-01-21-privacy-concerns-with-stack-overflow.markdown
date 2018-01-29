---
layout: post
title:  "Privacy concerns with Stack Overflow"
date:   2018-01-21 10:00:00 +0000
categories: security privacy
---
As I was writing [my last blog post][last post] I ended up referencing a couple of different Stack Overflow (SOF) answers. There was a strange thing I noticed as I was using the **share** "button" though: all the generated links had the exact same suffix, a large number.

Here are some example URLs from that post:

```
https://stackoverflow.com/a/3396790/1310016
https://stackoverflow.com/a/9350415/1310016
https://stackoverflow.com/a/5829387/1310016
```

As you can see, the last path segment is identical in each of these links, but they definitely all point to different answers on Stack Overflow. This can only mean that the one before last segment must be the unique ID of the answers, but then what is the last component?

Although the number looked familiar, I wasn't quite sure where I've seen it, so I just ended up opening my profile page, and hey presto, the ID is in the URL:

```
https://stackoverflow.com/users/1310016/peter-major
```

So looks like that if you are logged into Stack Overflow and you want to share some links with others, SOF will automatically embed your user ID into those links. Since the answer's unique ID is part of the URL already, the only valid reason for having the user's ID in the URL is user-tracking...

Think of it this way: user A sends user B a link, user B is logged into SOF already and opens the link received. At this point SOF knows the current user's ID and the sender's ID, hence it now knows that there is some sort of a relation between those accounts. Depending on how many links are shared in-between, SOF can build up a map of end-user connections and a list of frequently shared topics within the group.

Now of course this can result in better search results and better post
 recommendations which is all great, but I'm not sure if I like that SOF knows with whom I'm interacting with. My solution for now is to remove the last path segment from the to-be-shared URLs.

Am I being paranoid? Would you not consider this as violation of your privacy? Let me know in the comments section.

**UPDATE**:

Stack Overflow [replied] to my tweet and explained a few things:
* The user ID is there so that people can collect certain sharing related [badges] in StackOverflow.
* There is actually a text displayed to the end-user that explains that the link contains the user ID when you click on share (really not sure how I missed this!)

Great to see that Stack Overflow is using their development blog to explain their features and decisions, wish this would be more common...

[last post]: https://aldaris.github.io/dev/js/2018/01/14/credit-card-number-validation-done-wrong.html
[replied]: https://twitter.com/StackOverflow/status/955846295683346432
[badges]: https://stackoverflow.blog/2010/09/09/announcer-booster-and-publicist-badges/
