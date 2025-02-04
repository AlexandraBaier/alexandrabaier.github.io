---
layout: post
title: "Book review:  \"Tidy First?\" by Kent Beck"
tags:
  - software design
  - book review
---

A tidying is a "little baby miniature refactoring" and something a lot of agile and XP developers already do intuitively.
Kent Beck's "Tidy First?" formalizes and explains what tidyings are, how and why they work, and how to integrate them
into your software development process.
In my opinion, the value of tidying is two-fold:
First, it reduces the amount of state a developer has to keep in their mental space at the same time. Given that brain
processing power is a heavily limited resource, this is one of the best ways to make software development a little bit
easier.
Second, tidying will result in a much better software design in the long term at very negligible cost in the short term.
And since this is a review: The book is very helpful but also very short making it a definite reading recommendation.

<!--more-->

Kent Beck starts with concrete and very specific topics and then moves towards the more abstract tactical and strategic
parts of tidyings.
Let's follow in his footsteps and start by looking at explicit tidyings to figure out what they are.
After that, we will look at how to integrate tidying into the development process and finish up with some opinions.

* toc
{:toc}

## Tidyings

After reading this book approximately a year ago, I actively applied these tidyings
to a somewhat legacy Java Spring backend.
We will go over literally all tidyings that Kent Beck discusses in his book,
and as the German saying goes
[I will add my own mustard](https://de.wiktionary.org/wiki/seinen_Senf_dazugeben) to stuff when I feel that I have
sufficient mustard.

### Guard clauses

Deeply nested conditional structures can sometimes be flattened with guard clauses.
For example, a very common case in the legacy application I work on is the following

```java
if (authorization.isAllowed(user)) {
  if (authorization.hasFullAccess(user)) {
    do something complicated
  } else {
    do something simple
  }
}
```

This can be simplified to

```java
if (!authorization.isAllowed(user)) {
  return not authorized;
}

if (!authorization.hasFullAccess(user)) {
  return something simple;
}

return something complicated
```

The benefits are very straightforward. Each guard clause checks or validates something trivial. Once we get past the
guard clauses, we know that the input is valid or fully authorized or whatever.
Previously, we had to always keep the level of nesting in mind, before we can implement anything.
That can be risky. If we mix up the level of nesting, we might leak confidential data or pass invalid inputs
to some deeper part of our application.

### Dead code

Dead code is code that is not part of the productive code base.
Dead code should just be deleted.
There is no real cost to deleting dead code.
You can always restore it with your version control system.
However, sometimes it is hard to find dead code.
IDEs can often detect some dead code, but fail to do so when the code references itself somehow or is still under
test.
In these cases, we might need to trace suspicious methods and classes manually to see if they are actually being used.
Luckily, IDEs can usually tell us all callers of a method or class, so it is not that dire of an issue.

### Normalize symmetries

Normalizing symmetries means that you should try to solve similar problems with similar patterns.
Kent Beck gives the example of caching a value in multiple different methods and employing the same pattern to do so
each time.
In my case, applying this to the authorization problem made a lot of sense.
We have a few groups of users with similar permissions, who will almost always request similar kinds of data.
Normalizing symmetries for me then means to solve this authorization and data access pattern the same way each time.
Once this symmetry normalization is repeated often enough, other tidyings will be enabled.
In my case, there suddenly was a lot of duplicated code (and knowledge) on how user permission and data access are
linked.
This allows us to "extract helper" methods or create "new interfaces".

### New interface, old implementation

This tidying encourages implementing the interface, which would help you in solving your current problem.
This usually means putting a thin wrapper interface on an old implementation.
Test-driven development encourages this kind of approach naturally.
In my own experience, this approach introduces awkwardly similar abstractions in the short term but helps a lot in the longer
term.
It requires some discipline to actually use the new interfaces, but if the new interfaces are good you should want to
use them.
After a while, the new interface or interfaces will fully cover the old implementation, 
which allows you to get rid of the old interface by deleting dead code.

### Reading order

Put code into reading order, so you can easily follow a chain of methods calling each other.
The further down the file you go, the more specific the methods get.
This reduces your mental work load and reduces the amount of scrolling you have to do while trying to understand some specific piece of
code.

### Cohesion order

Similar to "reading order", the goal is put methods that are coupled to each other next to each other.
All the relevant code will be visible at once and your brain has to do less work remembering everything.

### Move declaration and initialization together

A very persistent artifact of the old days (looking at you C and Delphi) is having all declarations of variables
at the top of a method or file with the initialization happening late after.
The advice is simple, initialize when you declare.
Initialization provides a lot of context and information on how a variable will be used.
For example in Java, it might tell you if a variable is nullable, since we still do nulls for some reason.

### Explaining variables

Big massive expressions, e.g. for calculations, are not a good pattern even if it allows you to compress complex logic
into a single line.
Such expressions typically require you to comment them to explain their meaning.
If there was just a way to say what an expression means... (hint, hint)
Instead of big massive expressions, consider splitting up such an expression into small expressions.
Assign each small expression to a variable with an informative name.
And finally assemble the big massive expression from these variables.
When people talk about self-documenting code, I assume this is what they mean.

### Explaining constants

This is the same idea as "explaining variables", but for magic values.
"Magic" might sound cool, but by providing incantations it allows others to understand your witchcraft.

### Explicit parameters

Sometimes methods receive large structs or value objects as arguments.
This pattern is sometimes called a "parameter
object" ([https://refactoring.guru/introduce-parameter-object](https://refactoring.guru/introduce-parameter-object))
or a "data class" ([https://refactoring.guru/smells/data-class](https://refactoring.guru/smells/data-class)) depending
on how they smell.
This is the right choice sometimes. For example, if everything inside the parameter object is actually needed in the method.
If that is not the case, the parameter object can make life difficult.
Every time such an object is passed into a method, you need to keep the entirety of the object in mind to
understand some piece of code.
In such cases replacing the parameter object by the explicit parameters is helpful.
You will now know which 2 out of 10 parameters are actually needed by a method.
Your brain thanks you for your efforts.

### Chunk statements

Big blocks of code typically can be split up into logical segments.
Putting some new lines between each of these blocks is a first easy step towards gaining understanding.
The following set of tidyings are good ways to make use of the chunks.

### Extract helper

This is similar to the [extract method refactoring](https://refactoring.guru/extract-method).
You might for example extract a chunk as a helper method.
Extracting helpers reduces mental load and might ideally yield methods that are useful in more than one place.

### One pile

Sometimes the problem is not big blocks of code but a lot of tiny pieces of code in a lot of places
(looking at you "Clean Code" [you monster](https://qntm.org/clean)).
In such cases, you have to be aware of way too many methods in way too many places.
This makes it hard to make sense of what a method is actually doing.
One pile just means inlining everything until you got a good big pile of code.
This is a great way to get a new perspective and a holistic view at what the code does.
It typically enables other tidyings like "chunk statements" and might also help you detect duplication or
unnecessary data transformations.
The latter occurs from time to time in the legacy application I work on.
It always feels very nice, when I save a couple of unnecessary passes over some large data structure.

### Explaining comments

Code is supposed to be self-documenting, but sometimes it is not.
Especially when it comes to weird domain logic or edge cases.
When you understand some weird piece of code and can't or won't change it, at least put a comment next to it.
Explain why it is weird and what it does. Future you or future others will thank you.

### Delete redundant comments

Redundant comments are what LLMs and junior devs love. Stuff like

```java
// Increment i by 1.
i++;
```

Code is usually self-documenting. We don't need these kind of comments.

### Wrapping up tidyings

We have now seen a large set of tidyings and most of them make our developer life easier.
The nice thing about of all these tidyings is that they are very cheap and add value immediately.
Tidying should not take more than 15 to 30min (a reasonable upper bound by Kent Beck), 
but by tidying you reduce your mental workload and make hard-to-change things easier to change.
In the short term, you will save a little bit of time, but in the long term you will likely save a lot of time.
Kent Beck argues that tidyings can accumulate into an avalanche and solve even big design problems.
Tidying is also very fun, since it feels good to reduce technical debt even at a small scale.

## Managing tidyings as part of feature work

Tidying is usually supposed to accompany actual feature work.
Otherwise, you might be tidying the wrong things and for the wrong reasons.
The question is now once you have tidied, how will they enter the main branch.

Kent Beck argues for putting tidyings in their own pull requests (PRs) rather than including them into the feature PR. 
He also argues for small amounts of tidying per PR.
The reasoning is straightforward. 
A PR consisting of only tidyings and not too many tidyings is extremely easy to review.
No behavior is changed and only very little code is changed. 
There is very little risk in approving such a PR.
Additionally, having tidyings enter the trunk fast and regularly ensures that your tidyings don't mess with other team
members.
There is of course an issue with this approach.
If your fixed cost for reviewing and approving PRs is high, a lot of small PRs will result in disproportionate costs
relative to their value.
So the size of tidying PRs is a variable that has to be optimized by each development team.
For example, I write my own code and also review it. 
My fixed costs for PR review and approval are very negligible especially when it comes to tidying PRs.
I already know when something is a tidying PR and accordingly have to just quickly check that I didn't accidentally commit
behavioral changes.

Another issue with tidying PRs is tangling. 
Since we tidy to simplify feature work, the tidying changes and the feature changes are typically tangled up.
Untangling the feature work and the tidying accordingly requires some effort.
However, since we ideally try to only tidy a little each time and have the tidyings in the trunk very fast, the amount
of untangling work should always be limited.

## Some final thoughts

I will mostly skip the last section of the book in this review, since the message is more abstract and harder to
summarize (but still very important and worth a read).
Kent Beck gives his view on how software creates value and uses metaphors related to options trading.
Basically, software being flexible and allowing for a lot of feature options and directions of change generates value on
its own.
Tidying is a tool to make software amenable to having a lot of options.

After having read the book and practicing some of its teachings, I feel that tidyings work, 
but I am skeptical of how powerful they can be.
Previously, I mentioned how according to Kent Beck tidyings can accumulate into an avalanche to solve big design
problems.
This might be true sometimes. However, tidyings are also greedy optimizations at a very local level.
It is questionable to me how far such optimizations can take us towards a global optimum with respect to the "ideal" software
design.

For a way more expert opinion, Eric Evans has a lot of useful ideas on refactoring, which he states
in his iconic book "Domain-Driven Design" (2003, all following page numbers refer to it).
He actually proposes a similar idea to Kent Beck's idea of an "avalanche" but calls it a "breakthrough" (p. 122),
which are a result of constant domain-driven refactorings and a developing domain expertise in a team.
However, he also cautions us that such breakthroughs are only possible given certain other conditions.
First, refactoring (or here tidying) can keep the code clean but without domain understanding powerful
new features won't emerge from existing features (p. 9).
Second, going in small steps won't always allow you to go to a new better model.
Some changes will cause cascading effects on the remaining code base.
This can create a situation with very few stable points during each larger refactoring (p. 127).
In such cases, you have to bite the bullet and move from small tidyings to concentrated larger refactoring efforts.

So in summary, tidyings are a really helpful pattern to improve code quality, reduce mental load,  
and add value on a day-to-day basis.
It unfortunately is not a panacea healing all ails.
Bigger less pleasant domain-driven refactorings remain an important tool to improve software design.
Additionally, similar to big refactorings, tidyings benefit from developers who are interested in the domain and the
larger picture.
That way the "avalanche" or "breakthrough" becomes a possibility and hopefully also an eventuality.

