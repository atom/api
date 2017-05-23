# Where is the API hard to use?

I want to compile a list of instances where the API is confusing.

* What are people complaining about?
* What are people screwing up in their packages?

## Links to problems / talk about package writing

* http://gnuu.org/2014/03/10/my-week-with-githubs-atom-editor/


* https://discuss.atom.io/t/how-to-speed-up-your-packages/10903
* https://discuss.atom.io/t/looking-for-best-practices-for-iterative-development-on-atom/10889

## What sucks

### Events

We need a conssitent story for subscribing to events. I personally dont know what the best practices are for subscribing to events.

From http://gnuu.org/2014/03/10/my-week-with-githubs-atom-editor/
>There Are 4 Ways To Use Events
>
>Searching for a way to register events in my plugins, I ended up reading a handful of different core Atom plugins that do event registration. A came across 4 distinct ways to actually register events:

Confusion between

* dom events and model events
