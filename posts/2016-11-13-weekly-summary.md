---
title: Weekly Summary 3 - Ending 2016-11-13
author: Brian Jones
---

This was an interesting week full of various technical achievements. Unfortunately I did not manage to stay focused and achieve the goals I lined out last week.

## Time Breakdown

```bash
#+BEGIN: clocktable :maxlevel 4 :block thisweek :scope <various>
#+CAPTION: Clock summary at [2016-11-13 Sun 23:29], for week 2016-W45.
| File             | Headline                  | Time      |       |       |      |
|------------------|---------------------------|-----------|-------|-------|------|
|                  | ALL *Total time*          | *1d 7:00* |       |       |      |
|------------------|---------------------------|-----------|-------|-------|------|
| UncannyWorks.org | *File time*               | *0:47*    |       |       |      |
|                  | Uncanny Works             | 0:47      |       |       |      |
|                  | \_  Project Management    |           |  0:47 |       |      |
|                  | \_    Web Site            |           |       |  0:47 |      |
|                  | \_      Recurring         |           |       |       | 0:47 |
|------------------|---------------------------|-----------|-------|-------|------|
| HackSlash.org    | *File time*               | *2:00*    |       |       |      |
|                  | Hack Slash                | 2:00      |       |       |      |
|                  | \_  Project Management    |           |  2:00 |       |      |
|                  | \_    General Programming |           |       |  2:00 |      |
|------------------|---------------------------|-----------|-------|-------|------|
| ArmoredBits.org  | *File time*               | *8:19*    |       |       |      |
|                  | Armored Bits              | 8:19      |       |       |      |
|                  | \_  Project Management    |           |  8:19 |       |      |
|                  | \_    Hack Slash Refactor |           |       |  8:19 |      |
|------------------|---------------------------|-----------|-------|-------|------|
| FiveMinuteRL.org | *File time*               | *19:54*   |       |       |      |
|                  | Five Minute Roguelike     | 19:54     |       |       |      |
|                  | \_  Project Management    |           | 19:54 |       |      |
|                  | \_    General Programming |           |       | 19:54 |      |
#+END:
```

### Uncanny Works

This blog update.

### Hack Slash

While I do not recommend public consumption just yet, [Hack Slash](https://github.com/uncannyworks/hack-slash) code is now available.

One of the more interesting things I did here was spend a little time getting the `HackM` monad working. This is quite simply a type synonym which wraps the `State` monad with a `Scene` and a polymorphic game type.

```haskell
type HackM a = State Scene a
```

The game developer shouldn't have to worry about threading a `Scene` through their functions, which in the beginning I was doing until this new approach dawned on me. They can now use the new `HackM` monad which gives them access to `get` and `put` allowing them to grab and update the `Scene` only when desired.

```haskell
-- old way
oldNoSceneFunc :: GameLogic -> Scene -> (GameLogic, Scene) -- <-- terrible Scene threading
oldNoSceneFunc logic scene = (doThingsToGameLogic logic, scene)

oldSceneFunc :: GameLogic -> Scene -> (GameLogic, Scene)
oldSceneFunc logic scene = (doThingsToGameLogic logic, doThingsToScene Scene)

-- new HackM way

newNoSceneFunc :: GameLogic -> HackM GameLogic
newNoSceneFunc = return . doThingsToGameLogic

newSceneFunc :: GameLogic -> HackM GameLogic
newSceneFunc logic0 = do
  scene0 <- get
  put $ doThingsToScene scene0
  return $ doThingsToGameLogic logic0
```

### Armored Bits

All of the work involved here was integrating the above `HackM` monad into the system.

This gave me a lot of insights into where I was doing some things terribly wrong, however, I'm currently pondering how to correctly handle 3 functions which don't cleanly bridge the gap and as a result didn't reach my goal of getting AB into a working state.

### Five Minute RL

I've been obsessed with the idea of building a roguelike for years now, and continue to ask myself why I never just sat down and started one. In this case I finally had a concrete idea that is much smaller in scope than Prismatica (which is a full on world simulator), and can ideally get done in a reasonable amount of time, packaged up, and sold through various channels.

The majority of time spent here was importing and manipulating a large number of sprites to work in Unreal Engine 4.

As usual Blueprint programming is kind a neat break from actual text based programming. I managed to get a basic random dungeon generator working in an afternoon yesterday.

I'll discuss the core game play in a future blog post as development progresses a little further.

## What's Next?

Primary goal is to get Armored Bits back into a 100% working state.

Secondary goal is to keep moving Five Minute Roguelike forward. I haven't sat down and built a toy in quite a long time, so this has been a nice break.
