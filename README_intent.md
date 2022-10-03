# Test Apps On The Edge

This is an attempt to test apps the time they bootstrap, crash or recover. To be able to deliver such practical solution, let's rephrase the previous phrase to the following:

Take decisions on un-bound data under the [closed world assumption](https://en.wikipedia.org/wiki/Closed-world_assumption) 

Before understanding "intent" please read [README](README.md) for more.

## Intent

Streams can be "pulled" or "pushed". Simply values can be asked for (on demand of the consumer) or continuously ingested (on the decision by the producer).

In TAOTE, I would like to introduce the concept of "Intent". Please let me lay it down through some examples first so you can get a taste of what "Intent" in the context of streams might be.

### Examples

Having a worker A in a Factory F; Let's assume A is a consumer of tasks and the Factory is a producer of tasks.

- Somebody is doing a job while not really liking it, the only motive is to get some money:
    - When A says "enough", that means A can no longer consume anything. Before A say "enough" A is OK with it, we know A is neither happy nor sad, A just wants money.
- Now imagine, A is the only one in the Factory, no authority and the Factory is in fact a river with plenty of gold:
    - A is always happy about it if the time A asks for Gold, A gets a new amount of Gold
    - A is unhappy about it if there is a delay
    - Before or after saying "enough" has no influence on A being Happy or unhappy


To simplify, you can view the first case as "Responsibility" and the second as a "Game". You can see we introduced a whole new view of "consuming vs. producing" streams. In contrast to the classic "pull vs. push" approach.

-- to be continued.