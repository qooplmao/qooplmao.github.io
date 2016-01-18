---
layout: post
title: CQRS Basics
comments: false
tags: [ cqrs ]
---

CQRS = Command Query Responsibility Segregation  
Commands and events are managed/created the write side.  
The read side just runs through all of the events and creates an object based off those.  
With the read side only using the available events in a repeatable way the database could  
be destroyed and rebuilt to give exactly the same read data.

Image the situation where we have a gym.

**The Write Side (Commands and creating events)**

We have commands like:

- JoinGym
- EnterGym
- LeaveGym
- QuitGym

Now imagine this process:

| COMMAND                  | EVENT                                          |
| :----------------------- | :--------------------------------------------- |
| JoinGym (customer Jim)   | MemberJoinedGym (Jim)                          |
| JoinGym (customer Bob)   | MemberJoinedGym (Bob)                          |
| JoinGym (customer Jim)   | *No event created as Jim is already a member*  |
| EnterGym (customer Jim)  | MemberEnteredGym (Jim)                         |
| EnterGym (customer Bob)  | MemberEnteredGym (Bob)                         |
| LeaveGym (customer Jim)  | MemberLeftGym (Jim)                            |
| QuitGym (customer Jim)   | MemberQuitGym (Jim)                            |
| QuitGym (customer Jim)   | *No event created as Jim already quit*         |
| LeaveGym (customer Bob)  | MemberLeftGym (Bob)                            |

All this is handled on the write side and events are stored to a table like(-ish):

| ID  | DATETIME               | SERIALIZED EVENT (a super basic version)    |
| :-- | :--------------------- | :------------------------------------------ |
| 1   | 2015-01-01 10:00:00    | {"MemberJoinedGym":{"Customer":"Jim"}}      |
| 2   | 2015-01-02 12:00:00    | {"MemberJoinedGym":{"Customer":"Bob"}}      |
| 3   | 2015-01-03 10:00:00    | {"MemberEnteredGym":{"Customer":"Jim"}}     |
| 4   | 2015-01-03 10:30:00    | {"MemberEnteredGym":{"Customer":"Bob"}}     |
| 5   | 2015-01-02 12:00:00    | {"MemberLeftGym":{"Customer":"Jim"}}        |
| 6   | 2015-01-02 12:15:00    | {"MemberQuitGym":{"Customer":"Jim"}}        |
| 7   | 2015-01-02 13:00:00    | {"MemberLeftGym":{"Customer":"Bob"}}        |


**The Read Side (Read Models by pulling events)**

Then on the read side we create a, generally, single use object that runs through all 
of the events, listening for any that are relevant, and works out whatever it is required to.
This can be done as the events are created (synchronously) or after the fact (asynchronously).

From the events that we have there (*MemberJoinedGym*, *MemberEnteredGym*, *MemberLeftGym* &
*MemberQuitGym*) we can create a whole load of read models that do completely different
things.  
For example:

- A list (or the number) of all current members [^1]
- A list (or the number) of a members that joined in a certain period
- A list (or the number) of members that have quit
- A list (or the number) of members that quit within a month of joining up
- A list (or the number) of members that are current in the gym

There are probably a bunch of other things that can be generated from these events,
but the main benefit is that with the events being stored there isn't the necessity to
think of all of them at this point. One day in the future we may want to create a report
that gives us the numbers of members that regularly come on a Sunday so that you could
decide whether or not to extend the opening hours. 

[^1]: Read model would listen to `MemberJoinedGym` and `MemberQuitGym`. Each time a member joined they would be added to the list and then removed when they quite.