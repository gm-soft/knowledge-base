---
layout: post
title: Why do browsers match CSS selectors from right to left?
category: software development
tags: [css, web]
---

Keep in mind that when a browser is doing selector matching it has one element (the one it's trying to determine style for) and all your rules and their selectors and it needs to find which rules match the element. This is different from the usual jQuery thing, say, where you only have one selector and you need to find all the elements that match that selector.

If you only had one selector and only one element to compare against that selector, then left-to-right makes more sense in some cases. But that's decidedly *not* the browser's situation. The browser is trying to render Gmail or whatever and has the one <span> it's trying to style and the 10,000+ rules Gmail puts in its stylesheet (I'm not making that number up).

In particular, in the situation the browser is looking at most of the selectors it's considering *don't*match the element in question. So the problem becomes one of deciding that a selector doesn't match as fast as possible; if that requires a bit of extra work in the cases that do match you still win due to all the work you save in the cases that don't match.

If you start by just matching the rightmost part of the selector against your element, then chances are it won't match and you're done. If it does match, you have to do more work, but only proportional to your tree depth, which is not that big in most cases.

On the other hand, if you start by matching the leftmost part of the selector... what do you match it against? You have to start walking the DOM, looking for nodes that might match it. Just discovering that there's nothing matching that leftmost part might take a while.

So browsers match from the right; it gives an obvious starting point and lets you get rid of most of the candidate selectors very quickly. You can see some data at [http://groups.google.com/group/mozilla.dev.tech.layout/browse_thread/thread/b185e455a0b3562a/7db34de545c17665](http://groups.google.com/group/mozilla.dev.tech.layout/browse_thread/thread/b185e455a0b3562a/7db34de545c17665) (though the notation is confusing), but the upshot is that for Gmail in particular two years ago, for 70% of the (rule, element) pairs you could decide that the rule does not match after just examining the tag/class/id parts of the rightmost selector for the rule. The corresponding number for Mozilla's pageload performance test suite was 72%. So it's really worth trying to get rid of those 2/3 of all rules as fast as you can and then only worry about matching the remaining 1/3.

Note also that there are other optimizations browsers already do to avoid even trying to match rules that definitely won't match. For example, if the rightmost selector has an id and that id doesn't match the element's id, then there will be no attempt to match that selector against that element at all in Gecko: the set of "selectors with IDs" that are attempted comes from a hashtable lookup on the element's ID. So this is 70% of the rules which have a pretty good chance of matching that *still* don't match after considering just the tag/class/id of the rightmost selector.