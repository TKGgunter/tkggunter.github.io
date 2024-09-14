+++
title = "Xcopyfgc: an attempt at a general Shadow Ai system."
date = 2021-07-21
description = "A high level overview on the Xcopyfgc project and it's initial results."
+++



Fighting game ai is bad. 
At least on average.
In fact fighting game ai often fails at its most basic purpose: to give players a sense of the dominant strategies and current meta of the game or character.
Unfortunately it often does the opposite, by introducing players to strategies only a computer can accomplish.
Even when a game comes out that pushes the envelope on ai the advancements rarely are these advancements adopted by newer games.


> This critique is not intended to blame the developers.
> There is a lot to do in game development, and software development in general,
> if any portion of a fighting game would get the short end of the budget I suspect it would be ai.


The Killer Instinct's Shadow Ai is the most recent, well known, push on ai in fighting games.
Its Shadow Ai attempts to mimic player behavior, by recording and playing back player inputs.
Derek Neal and Bruce Hayles thoroughly explain the system in the GDC talk below.


<iframe width="560" height="315" src="https://www.youtube.com/embed/9yydYjQ1GLg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>



This project, xcopyfgc, is my attempt to replicate the techniques used by Shadow Ai in a general purpose ai program for fighting games.
This means a tool that can be used for any fighting game, in a manner that is as close to plug and play as possible.


## The Plan

Creating something like Shadow Ai for ones own game is no small task.
This task is further complicated when trying to build and ai for someone else's. 
The lack of direct access to player/character information, or game state presents a huge problem.
It is for this problem that this project is essentially two. One aimed at deriving game state, the other creating an ai that decides what input should be delivered to the game.


Game state is a term referring is information describing everything occurring during a particular frame of a game. 
This includes distance between two characters, the of animation a character is in, among other things that the game needs to know to run successfully.
For fighting game players this extends to what has previously occurred in the match.
Players keep "track" of a number of statistics that help them choose their next action. Statistics like frequency of jump ins, reversals, all shape a players decisions.
If one wants to make a good Ai keeping track of these statistics are required.


When working within a code base these quantities are rather easily and quickly determined.
Many hobbyist fighting game ai developers determine game state by grabs values from game memory. 
This project will not. Instead, game state information will be determined using visuals and audio(future).
This choice offers significant advantages and disadvantages.
Game state from audio and visual information necessitates a general method for deriving game state. A method that should be agnostic to patches and updates.
As any hobbyist modder will tell you it is a fair bit of work to acquire to relevant the pointers to game memory and patches and update usually require some subset of this work be done again.
This project will try to side step with issue, as well as potential legal ones, by using commonalities in audio visual design seen throughout the genre.

 
<div style="width: 50%; float:left">
<img src="https://farm6.static.flickr.com/5260/5475430759_05b2d7aff3.jpg" width=500>
</div>
<div style="width: 50%; float:right">
<img src="https://gamefabrique.com/storage/screenshots/arcade/killer-instinct-02.png" width=500>
</div>



Fighting games have a rather distinct visual and auditory language. 
These design elements are so common at a glance players/spectators not only know they viewing a fighting game, but can parse much of the game state.
For the positioning of health and meter bars to where characters begin each round, this consistency can be used to construct algorithms that parse the game state.
Deriving such algorithms can be difficult.
Not all games depict information clearly.
There are occasions where the presentation is deliberately misleading.
This is a con, and rather large one at that, I'm willing to live with.


I'll discuss the specifics for parsing audio and visual information later, but the basics, as stated before, are leveraged from common fighting game design language.
For example:
- heath bars at the top of screen,
- color palettes of playable characters being distinct when compared to the opposing characters and the background,
- consistent prematch starting positions.

Through the use of these elements in addition to incorporating naive machine learning one can determine some of the basics features a game state.


Outside of eyes and ears a brain for the Ai is needed.
Here, I leverage the work described by Iron Galaxy in their Shadow Ai talk.
Xcopyfgc uses recorded user inputs and game state data to determine what the Ai should do next.
During a match the ai replays the inputs recorded so long as the game state remains reasonably close to the recorded game state.
If the current game state becomes too dissimilar the ai will change to a recording that better matched the current state.

I'll be writing an in depth post on this some time later as well, but that is the gist.



## So... does it work?

The preliminary work looks promising. 
Character position identification works well, and works across several games.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Below are a few clips of a fighting game ai project I&#39;m working on. The computer can &quot;see&quot;! <a href="https://t.co/48dRtXjvJo">pic.twitter.com/48dRtXjvJo</a></p>&mdash; ThothGunter (@TKGgunter) <a href="https://twitter.com/TKGgunter/status/1415773778060120064?ref_src=twsrc%5Etfw">July 15, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


On the ai side, basic sequences can be replicated. 
The following video replicates the demo presented by Derek and Bruce.


It's been pretty exciting to developing this and I plan on see just how far I can push it.
If your interested in this project you can follow me and it on Twitter <a href="https://twitter.com/TKGgunter">@TKGgunter</a> and for more videos <a href="https://www.youtube.com/channel/UCABoYor0BYe94swDr9dQpMg">here</a>.

Thanks for reading.
















