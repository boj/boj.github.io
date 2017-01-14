---
title: Weekly Summary 1 - Ending 2016-10-30
author: Brian Jones
---

One thing that has been abundantly clear to me over the years is that people I consider successful tend to do things like [post](http://sachachua.com/blog/) [regular](http://www.gridsagegames.com/blog/) [updates](https://www.truevalhalla.com/) about things they are doing. Success in this case doesn't exactly mean turning a profit, although it can. It's more like people clearly setting goals for themselves, pushing forward getting things done, and actually seeing said goals all the way through to completion.

I've been incredibly passionate about the work I have been doing on our flagship "game" product [Armored Bits](https://armoredbits.com) and I'd truly like to make Uncanny Works the indie game dev business a success, but without an outlet like a blog or posting to a community sometimes it feels like my perspective gets lost and I forget why I invest so much personal time into all of this.

Recently I've felt like I need more motivation, so I've embarked upon two new recurring tasks.

* Start writing weekly blog updates, even if it's to post that I did absolutely nothing.
* Track every chunk of time I spend on UW related work.

## Organization

While I wish I could dedicate every waking hour to making Uncanny Works successful, the reality is that I still need a *day job* to support my family and put food on the table. My nearly 2yo daughter is a full time job alone, so it's incredibly important to me to find that balance between working, being with my family, and wisely utilizing my extra time to focus on UW projects.

Starting somewhere during the middle of this week I decided to go back to using the feature rich [org-mode](http://orgmode.org/) tool to organize my project management and time. I commented a tad about how I am using org-mode in the [/r/roguelikedev](https://www.reddit.com/r/roguelikedev/comments/59rgiv/faq_friday_50_productivity/d9auju5/) FAQ Friday post on **Productivity**. Even though it's only been a short time I am already seeing how useful this tool will be for me.

## Time Breakdown

```bash
Clock summary at [2016-10-30 Sun 17:18], for week 2016-W43.

| File             | Headline                        | Time    |      |      |      |
|------------------|---------------------------------|---------|------|------|------|
|                  | ALL *Total time*                | *15:33* |      |      |      |
|------------------|---------------------------------|---------|------|------|------|
| UncannyWorks.org | *File time*                     | *6:40*  |      |      |      |
|                  | Uncanny Works                   | 6:40    |      |      |      |
|                  | \_  Project Management          |         | 6:40 |      |      |
|                  | \_    Web Site                  |         |      | 6:40 |      |
|                  | \_      One Shots               |         |      |      | 1:00 |
|                  | \_      Recurring               |         |      |      | 2:50 |
|------------------|---------------------------------|---------|------|------|------|
| HackSlash.org    | *File time*                     | *0:46*  |      |      |      |
|                  | Hack Slash                      | 0:46    |      |      |      |
|                  | \_  Project Management          |         | 0:46 |      |      |
|                  | \_    General Programming       |         |      | 0:46 |      |
|------------------|---------------------------------|---------|------|------|------|
| ArmoredBits.org  | *File time*                     | *6:22*  |      |      |      |
|                  | Armored Bits                    | 6:22    |      |      |      |
|                  | \_  Project Management          |         | 6:22 |      |      |
|                  | \_    Hack Slash Refactor       |         |      | 6:22 |      |
|------------------|---------------------------------|---------|------|------|------|
| Prismatica.org   | *File time*                     | *1:45*  |      |      |      |
|                  | Prismatica                      | 1:45    |      |      |      |
|                  | \_  Project Management          |         | 1:45 |      |      |
|                  | \_    One Shots                 |         |      | 0:30 |      |
|                  | \_      Initial Haskell and Git |         |      |      | 0:30 |
|                  | \_    Documentation             |         |      | 1:15 |      |
```
### Uncanny Works

* Decided to try and do a redesign of our primary website [http://uncannyworks.com/](http://uncannyworks.com/) - however it's not complete. My goal here is to not only pretty it up, but build a "consulting" page which showcases the myriad of work we've done over the years. I would like to leverage all of that experience I've had working at Japanese social game companies and Chris' library of Unity3d games as a way to get UW on a client work route.
* Spent an hour fixing the letsencrypt scripts to automate SSL for [https://armoredbits.com/](https://armoredbits.com/) (thanks to @[vektorweg](https://twitter.com/vektorweg/status/792104174347493376) for pointing this out).

### Hack Slash

This is a project I haven't announced just yet. Basically it is the complete extraction and reapplication of an [Entity Component System](https://en.wikipedia.org/wiki/Entity_component_system) taken from Armored Bits. The long term goal is to build a feature rich game engine in Haskell. We'll see how practical that actually is once I start throwing graphics at it all, of which I will discuss in more depth later.

There is a bit more time spent on this than reported, however, as it stands the majority of preliminary work is out of the way for all non-graphical stuff as most of it was pulled from Armored Bits as the base.

### Armored Bits

There is a bit more time spent on this than reported, however, all of the time this week has been specifically spent on refactoring Armored Bits back over the new Hack Slash engine. This has been personally satisfying in more ways than I can count, specifically in regards to writing much tighter Haskell code and decoupling things to work within a smarter ECS context.

Once this is complete (hopefully early next week) then I will be moving back to pure features and aim at our alpha2 release goal planned for early next year.

### Prismatica

Another unannounced project, this is a game/world simulation that has been in my head for nearly 15 years. It will be a roguelike of sorts and is the motivating factor for adding graphical features to Hack Slash. I will be posting in the Sharing Saturday threads in [/r/roguelikedev](https://www.reddit.com/r/roguelikedev) once I am confident I have some solid things to share.

## Social Media

There was a post in [/r/gamedev](https://www.reddit.com/r/gamedev/) about a [Twitter list for game devs](https://twitter.com/ThatDarnSteve/lists/r-gamedev) so I asked to have my account added to the list.

## What's Next?

My goals for next week are to finish refactoring AB with Hack-/ and touch up the UW web page a little more.

Thursday is a Japanese national holiday (Culture Day), and I intend to take paid leave on Friday, hopefully giving me a bit more time to do things.

Starting at the end of next month I also plan on writing a Monthly Summary with the goal of giving a larger overview of the Uncanny Works business, how personal time was spent, etc.
