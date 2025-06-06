# Frequently Made Suggestions
This is a list of frequently made suggestions, and the current stance on how viable they are.
Broad categories are “This is already implemented”, “Someone is working on it now”, “This will go in as soon as someone implements it”, “This doesn’t fit the main game, but would be fine for a mod that ships with the game”, “We might add support for it to the code, but we won’t/can’t distribute it with the game”, “This isn’t reasonable to implement”, and “We’re not adding support for this to the code even for a mod”.

- [Frequently Made Suggestions](#Frequently-Made-Suggestions)
  - [Project Management](#Project-Management)
  - [Game Features](#Game-Features)
    - [Performance](#Performance)
    - [Multiplayer](#Multiplayer)
    - [Rogue Like](#Rogue-Like)
    - [Player abilities](#Player-abilities)
    - [Electrical power transmission](#Electrical-power-transmission)
    - [User interface](#User-interface)
    - [Monsters](#Monsters)
    - [Item and terrain additions](#Item-and-terrain-additions)
    - [Vehicle additions](#Vehicle-additions)
    - [Gun Behavior](#Gun-Behavior)
    - [Environment](#Environment)
  - [Failed Rationalizations](#Failed-Rationalizations)

## Project Management

#### We should vote on this: Nope
The project isn’t a democracy, no number of "me too"s or votes is going to change something we’ve decided on.

The method for users to influence development is via debate. This is intentional, not due to laziness.
Debate sidesteps all the nasty problems around deciding who gets to vote and enforcing that everyone gets the right amount of voting power etc. and validates that people actually know what they’re asking for.

A good **reason** to make a change has more impact than any number of votes. A lot of the issues in this thread are good examples of this

The project is run by Kevin Granade (“owns” the project, final say on features), a small group of core developers who review and merge changes, and a much larger group of contributors who make pull requests on GitHub. There are also translators who are effectively independent and a handful of people with moderation rights on the forums and Discord. Mod authors who host their own mods are also independent.

While we (the core contributors) ask for feedback and discuss issues on the forums pretty regularly, we aren’t asking for a vote or community consensus, just feedback and discussion.

Places where votes and popular support are important are which parts of the game are in most need of bugfixes or new features, sometimes a dev (Kevin included) finds themselves between projects and is just looking for something to improve, that’s when making it clear what needs improvement the most can make things happen.

**Steam, other app stores: not opposed, but we aren’t doing it or endorsing it either.**

Regarding Steam, this [Steam Store page](https://store.steampowered.com/app/2330750/Cataclysm_Dark_Days_Ahead/) respects the project license.

The game is also available from the [Play Store](https://play.google.com/store/apps/details?id=com.cleverraven.cataclysmdda) for Android.

In general, it’s just a ton of work for not that much benefit from the project’s point of view. If someone wants to integrate with some packaging system, they can feel free to PR it, but we're not generally going to be pursuing app store inclusion as a project priority.

**You should add a: Yes, in general, if someone is willing to do the work**

This game is open-source, welcomes contributors, and is highly customizable.  Aside from certain items on the list below, most suggestions for additions are going to get approved, either in the vanilla game, or in a mod.  The tricky part is finding a contributor to add the item.

The FMS list is here because some ideas have complicated reasons why they're not going to be accepted, or why they'll be accepted but not anytime soon. For most other ideas, they'll get accepted as soon as someone puts forth a contribution of sufficient quality and there's no need to put anything on this list.

## Game Features

### Performance

#### Improving Performance via Multithreading: Not just no, but hell no.

Several key developers (including Kevin) already have horror stories about debugging threadlock races that have a 3+ hour run time to reproduce for their day jobs and are extremely insistent that they do not want to take on that kind of debugging for their free time hobby. So adding multithreading is a non-starter, because it will substantially shrink the pool of talented developers who will contribute code changes.

Multithreading massively increases the overhead of maintaining the game. That's an ongoing overhead that just never goes away, it worms its way into every part of the system. For example, if anything but thread #1 wants to touch a data structure, that data structure must now be threading-safe and/or protected by locks

Also, we use mingw for windows builds, it turns out mingw does not have working POSIX locking primitives. (If you do multithreaded on mingw, you abandon linux portability and use windows locking primitives). This is another huge maintenance burden.

Problem #3 is that multithreading doesn't get you performance improvements as fast as almost anyone thinks it will. To break that down, threading is relatively good for throughput, but it's hard to use it to improve latency.  As it turns out, user-facing programs almost never care about throughput, they only care about latency, so that kind of sucks.  So for example, one of the expensive things we do is calculating FoV, it turns out it's what's called embarrassingly paralleizable, if a little tricky (i.e. you can break it up into 8 almost entirely independent jobs).  The catch is, that task is actually dominated by cache misses instead of computation, and multithreading makes that worse, because you have to ship the input and output data around to the various CPUs. So splitting that task up is relatively easy, but I'm not at all certain that doing so would make it any faster.

We had some mystery regressions and some not-so-mystery regressions since 0.E, subsequent improvements have clawed all that performance back and more.  So we're in pretty good shape now.

Which brings me to the "soft" problems with multithreading. Say we do get good multithreaded optimizations and performance increases linearly with cores (so much lolnope, performance almost always increases with diminishing returns). I develop on two systems, one has 8 functional cores and the other has 12 functional cores. In the perfect multithreading case, there's a 50% performance difference between the two systems, so a code change with performance that is barely tolerable on the 12 core system is untenable on the 8 core system.

Keeping DDA single threaded helps keep me (and the other developers) honest, because it narrows the gap between the best and worst systems available. It also narrows the gap within the userbase, including extreme cases where people are still on single or dual core CPUs.

Finally there's the opportunity cost issue. For the effort of multithreading key parts of the game, we can put a hell of a lot of investment into more generally applicable optimization.  There's a hell of a lot of cache coherency and algorithmic optimization we have planned that we haven't gotten around to yet. Optimizing the code would result in some sophisticated and finely tuned code we have to maintain, but we don't have to worry about the rest of the code being hard to maintain. One very large optimization performed since this entry was added was [creature reachability zones](https://github.com/CleverRaven/Cataclysm-DDA/pull/69574). This is one sort of always-applicable optimization that does not require specific software or hardware support for multithreading.

That's the worst thing about multithreading IMO, as soon as you have multiple threads, you have to start worrying about thread safety throughout your code.

#### Bringing charges back: No.

We are in the process of removing charges, that's a fact that won't be changed.  We should have made this decision much earlier or never started implementing them in the first place, but it is what it is.

The main reason we are doing this, contrary to popular belief, is to resolve technical debt. To handle both items and charges, every function of the code that interacts with them in any way needs to be effectively duplicated - each function in the code works by their own rules, each interacts with items in their own way, and usually have to be maintained separately. It introduces a tremendous amount of difficulty for any contributor, for even the most basic of tasks.

Some examples:
If you define an item without charges, its weight and volume would be equal according to the `weight` and `volume` fields:

```
"weight": "800 g", "volume": "200ml" = 1 item weight 800 g and has volume of 200 ml
```

But if you add charges, the volume would be divided by it's `count`:

```
"weight": "800 g", "volume": "200ml", "count": 50 = 1 item has weight 800g, and volume of 4 ml
```
(The same applies to the price, by the way)


> Wolfram Alpha tells me that the current density of bean seeds is greater than that of our sun. I do not think beans are denser than a star. (1L beans = 77.6kg) - RenechCDDA


It sounds like an issue that is easy to fix, but it is very difficult to properly fix if you don't know about this "feature".

Next, when you use charges in item definitions, you can't properly use `count` in item group spawns, instead you should stick to the `charges` field, that is there only and specifically for charges. Trying to use `charges` to spawn item? Only `count` of it would be spawned.  Trying to not use `count` to spawn charged items? nothing will spawn, because `count` is zero, therefore no item will spawn. Do you remember that the game spawns an amount of `charges` by default, if nothing else is specified? There are many, many instances of bugs, too many to count, that occurred simply because someone forgot to specify amount of charges in item groups.

Crafting is also one of the mechanics that has more issues than it should because of this: By default the game crafts `count` amount of item, which is trivial to miss when you edit a lot of recipes at once.

In the end, it's just odd to be able to hold 10000 rounds in your hands at once.

Again, we do not want to make the game worse with this - there are some unavoidable problems caused by this migration, but after it is done, not only it would be easier to contribute for everyone, but the game would work better, and maybe even faster.

### Options

#### I should be able to disable features I don't want (fungals, portal storms, etc.): No.

Every option has a maintenance cost, and the more options we have the more each of them individually costs in time, effort, and sanity for our dear contributors.

At the time of this writing, despite our best efforts there are *two thousand, one hundred, and fourty-three*(2143) open issues on the repository. The vast majority of them are bug reports, either confirmed or waiting to be confirmed. This does not include bugs which have been reported but were not confirmed before being closed due to a lack of activity. (Confirmed bugs are immune to such closures)

Can we afford the massive maintenance costs these options would bring? No. It would introduce many more bugs and greatly increase the workload even to fix existing ones, since the configuration of these options may be a cause.

Can we afford the massive development costs these options would bring? No. All new development would have to take into account increasing numbers of possible code paths based on whether or not major features were disabled.

Do we WANT these options to disable features? No! Features which make it into the game are desired features which contribute to the game's intended direction. We don't want to turn off the game we're making.

There are some existing "toggle/disable feature" options which are legacy leftovers (e.g. wandering hordes, no NPC needs, railways mod). Experience has shown that making these options are a mistake, despite the intention in making these optional being to avoid major issues with their implementations. 

The presence of those major issues precludes removing the option, and the fact it's an option precludes people from working on it ("oh I heard it has issues, I'll disable it" --> nobody actually uses it --> resolved issues or not, nobody would see their changes). This is a catch-22 for development, and the solution is simply not to make desired features optional.


### Multiplayer
This has come up [many times](https://discourse.cataclysmdda.org/search?q=multiplayer), and it simply can not be added to DDA.

The game loop of DDA includes a large number of activities that pass a large amount of time with no or minimal player input.

Some have proposed requiring coop mode, but that doesn't change the fact that there are huge number of scenarios where a player gets stuck with nothing to do for an extended amount of time.

This isn't just the obvious things like crafting and sleeping, but simply moving around tactically and examining the inventory and examining surroundings and menu interaction are not streamlined for short turn times.  This has implications for every part of the game, and I am not interested in making those adjustments.

The synchronization issues go much deeper than you seem to think they do, and resolving them would require overhauling most of the core game code.

I'm not interested in dealing with the security issues inherent in multiplayer network support.

### Rogue Like

#### ‘Fixing’ savescumming (in either direction): no.

People periodically point out places where savescumming breaks some part of the game, and likewise people point out “savescumming features” they want in the game. The answer to both is no.

If you encounter a bug while savescumming, you need to reproduce it without the savescumming.

Savescumming is not a normal part of the game, and there is no intention of ever adding features that facilitate it, like auto-backup of saves, tracking multiple saves, or the like.

### Player abilities

#### Dying and coming back as an NPC from your faction: Yes, with caveats

The technical implementation for this was added to the game between the release of 0.F "Frank" and 0.G "Gaiman". As of this writing, swapping to any follower is freely available on death of the player character. There are also vanilla options to swap the player character on a cooldown.

Currently there are no extra effects from this besides the changing of the player character, and your followers are perfectly content to follow whoever ends up in charge (even if the last six leaders were total failures and the new one appears to be unsuitable as leader). This is subject to change as the feature is developed.

Currently it’s way too easy to accumulate NPC followers and end up becoming effectively immortal.  Thus trivializes a lot of aspects of the game and encouraging even more reckless behavior. In the future, there may be limits on the NPC followers that you could switch to when you died, or differences in who would continue to follow the new leader.

#### Psychic powers: mod only

Not happening, it simply doesn’t fit the theme of the game.
However, Mind Over Matter is distributed with the game, and adds psychic powers. See [MAGIC.md](JSON/MAGIC.md) for more info.

#### Magic powers: mod only

Not happening, it simply doesn’t fit the theme of the game.
However, the Magiclysm mod is distributed with the game and is very extensible in JSON to support other systems of magic, such as Mind Over Matter. This infrastructure is also used with EOC's and activated mutations in the base game. See [MAGIC.md](JSON/MAGIC.md) for more info.

#### Remove skill rust: no

First things first: 99% of people ask to remove skill rust because their informations is based on very outdated information. Skill rust is a very old mechanic; it was added before Cataclysm DDA, by Whales, in Cataclysm. While I do not know why it was added in the first place, it was pretty terrible - you could level your skill now, but not being able to use it in crafting a few hours after, purely because the skill was rusted.
But not now. Skill rust was changed over the years to be a **purely positive mechanic**:
- Skills were divided into theoretical and practical levels, and a lot of functions were revisited to use the theoretical level, and increasing the practical level while having theory brings additional xp. Rust, meanwhile, is limited by only 1 ***practical*** level and grants even more free xp when restored;
- Skills overall were made floats, which means the difference between skill level 3 and skill level 2.99 is only 0.01, not an entire 1 level;
- The UI was changed to ensure the reptilian brain of ours will not associate skill rust with red numbers (because red = bad)
and skill rust impact (and benefit) would be even stronger once [#67580](https://github.com/CleverRaven/Cataclysm-DDA/issues/67580) is implemented fully.
This combined, the only way I can see someone wanting to remove skill rust is purely because of rumors from people not familiar with the changes, that are based themselves either on their outdated experience or from incorrect rumors they were told.
- To further clarify, if someone uses a mod to remove skill rust their character will level skills slower than a character experiencing skill rust.

#### Poop and related bodily functions: NO
No, just no, not even in a mod.

Hygiene facilities may become an issue with larger faction bases, and there may be some kind of *optional* furniture to allow collecting urine and manure for use in crafting and farming.

#### Bathing/accumulating scent: mostly no

Every way we look at it, this seems like it would mostly be a pain to deal with for the player and not fun if it simply accumulated over time.
One approach that might work would be specific monster attacks that make your scent stronger that you’d have to deal with somehow.

#### Craftable Automatic Weapons: mostly no

We limit crafting for the most part (exception, see cars) to things a single survivor with limited tools can create, and every reasonable plan for automatic action guns I’ve seen has required rather extensive tooling that’s not available to the survivor (metal folding/rolling machines, presses, drill presses).

The absolute closest thing to an automatic weapon I’ve been able to come up with that would be reasonable to craft is an old-school Gatling gun, and a motor for same to up the rounds per second. This currently exists ingame as the "12-gauge Gatling gun" (id: `bigun`).

At some point in the future we might build up tooling to the point where automatic weapons manufacture becomes feasible, at that point we can revisit this.

For further discussion on the subject, see https://github.com/CleverRaven/Cataclysm-DDA/issues/10787

Addendum, the Luty-pattern SMG, which should be craftable in DDA: https://github.com/CleverRaven/Cataclysm-DDA/issues/22688

#### Crafting Smokeless powder and primers: yes but needs work

Crafting basic smokeless powder is mostly a long, finicky, and tedious process with a mild risk of the powder prematurely detonating.  Having the avatar craft is tricky, because it's something like 2 weeks of tedious tasks that take about an hour/day but can't be done faster, followed by a few hours of grinding nitroglycerin-soaked cotton into powder without exploding.  The current CDDA crafting system can neither handle the repeated soaking/washing cycle, nor the risk of catastrophic failure when grinding.  Upgrading the crafting system to handle those things would be possible, but no one is currently volunteering to do the work.

As at alternate solution, faction camps have a mechanism to send an NPC off on a long and risky task, and have them either die or come back after the task is completed.  NPCs don't get bored doing tedious crafting tasks, and crafting recipes can be marked as NPC only.  Someone could write a gunpowder mill faction camp expansion and the associated recipes, and there's a good chance that it would be accepted into the game.

Crafting primers is a different set of challenges.  Primers are mildly unstable primary explosives and the risk of catastrophic explosive failure is a lot higher (and the explosions are worse).  Obviously, people historically made primers by hand, but sensible people prefer that only trained chemists do the work.  If someone wants to add a system for catastrophic explosive failures on crafting, a recipe for crafting primers might be accepted.  Alternately, skilled/proficient NPCs could craft gunpowder using the gunpowder mill faction camp expansion.

People have suggested various field expedients for making smokeless powder and primers.  Their research is interesting (and sometimes terrifying) but until there is either a system in place for extremely negative consequences on crafting failures or a system for skilled NPCs to craft this stuff, the field expedients are not going to be accepted, either.

#### Disassemble guns into components for repair or reassembly into new guns: mod only

This is not an issue with how things could or should work and more of an issue with extremely poor return on investment.

Enumerating all the parts of every gun would be a huge amount of data overhead, the code overhead isn't actually that high.

The situation where you have enough guns that you'd have a stock of swappable parts that you could make a really good gun out of, but don't just have a good gun would be vanishingly rare.

We don't have and aren't interested in having any scenarios where guns wear out in ways that you need a stock of replacement parts.

If you want to make a game that's about collecting parts and swapping them around, you streamline the parts and exaggerate the benefits of The few cases where you can do something meaningful like a calibre swap or a full auto conversion can be handled in a more targeted way with gun mods.

An extraordinarily dedicated modder could hack some of this kind of thing in with crafting recipes and gunmods, if they wanted to work with us on that process we could consider limited code support to help out.

#### “Enhancing” or “modding” melee weapons: ok for a mod

People frequently suggest that we add a system where melee weapons can have mods attached to them like guns can. There are two problems with this. First no one has seen a sensical suggestion of what these mods would be outside of fantasy type things. Second, the mod system for guns **is TERRIBLE** and shouldn't be duplicated. Someone would have to come up with something much better before it would get added.

The damage melee weapons cause is a complex combination of dynamic leverage as they are swung, weight distribution, and the interaction of the striking surface(s) with the target, and that doesn’t even get into the complexities of maneuvering past a targets defenses without opening your own guard.

#### Mining and smelting: Not a Minecraft

People lived in New England for about 15000 years.  People mined here for about 8000 years.  When some territory is mined for this long, almost any exposed, easy-to-reach ores and minerals tend to exhaust, so the only option for mining companies is to dig deeper, harder, and use more and more complex tools, machines, and reagents in order to transform rocks into useful materials.  Tools, machines, and reagents that character has no way to obtain and use, but also, **because character has no point to use it**.

Why would anyone sapient bother digging rocks and trying to convert them into steel, if they can just, you know, break a vehicle or some machine apart and get steel more pure than they would ever be able to reach otherwise? Why would anyone try to smelt stone to get some lead if they can just cut open the battery? Copper? Break into the substation, there is no electricity anyway.

For more ground substances like sand or gravel, the potential way to obtain more of them should not be by grinding big rocks into smaller rocks, but by finding a quarry or Home Depot and just loading your truck with what you need.

It doesn't apply to mods, they just need to be sure their attempt will bring more fun than tediousness.

#### Recover liquids from the ground: Partially implemented

This keeps being suggested, but it’s just not reasonable to recover spilled liquids from the ground and then get any kind of use out of them, since in general they’d be so adulterated by whatever would get mixed in with them that they wouldn’t be fit for any purpose.

Recovering specific liquids with specific properties might be acceptable, if there is some ultra-valuable liquid that gets spilled and had some special method of recovery and filtering.

Spilled liquids are recoverable only if they are spilled on terrain with specific flag, like bathtubs.

#### Dual-wielding weapons: Not practical

"Dual wielding" as in holding a pistol in each hand and firing both simultaneously is NOT going to be effective.  It will hopefully be added, but when it does it will be ludicrously ineffective because of penalties. The rationale for why this is so have been well-outlined already [here](https://discourse.cataclysmdda.org/t/dual-wield/1268) and there's no need or desire for further discussion. This will probably be represented in a very high rate of accumulation of recoil if you try to do this, most likely paired with an accuracy penalty.

Likewise, wielding and attacking with two melee weapons isn’t going to have any benefit over wielding a single melee weapon in both hands, either you’re going to be able to attack faster and deal more damage with the same weapon, or you’d be able to use a larger, more damaging weapon at the same speed and much more damage per strike. Attacking with one weapon and defending with another has a completely different set of trade-offs, and might be overall beneficial, especially if one of the items is very good at defending, like a shield.

#### Holding something in your off-hand: something that will be supported

If you’re holding something in your off hand and a gun (even a small pistol) in your primary hand, you’ll have a penalty assessed due to lack of stability. Probably based on the weight/volume of the carried item, so e.g. a small flashlight or a small melee weapon might have a minimal penalty, but another pistol is relatively heavy and might cause you problems.

Along with adding support for holding items in the off-hand, we’ll get stricter about tracking how many free hands you have to perform actions, and automatically perform the actions needed E.g. if you want to light a stick of dynamite and throw it while you are wielding a gun, the actual actions performed will be:

```
you activate the dynamite ->
    character put off the gun
    character takes out the dynamite
    character takes out the lighter
    character lights the dynamite
    character put off the lighter
    (you end up holding dynamite (lit) in your primary hand)
you throw dynamite ->
    character throws dynamite
you draw a gun
```
You may notice this sort of thing will “waste” some time by performing unnecessary actions sometimes, for example maybe you want to light and throw several sticks of dynamite in a row, in that case you’ll want to holster the gun first to avoid triggering the item swapping stuff.

On the other hand, this buffs pistols relative to rifles or shotguns, as it’s much faster to holster/draw a pistol (especially if you have a holster).

Even for simple stuff, like opening a door, if you have both hands occupied, it will put up an item, open the door, take out the item again.

When this happens (which will be a while, it’s pretty invasive and complicated), we will be VERY careful to not cause disruptions to doing things simply, with the extra actions just costing in-game time. If you’re in a hurry though and need to make every second count, it will be better to think ahead about a sequence of actions and try to minimize this sort of overhead.

#### Playing as a robot/playing as an intelligent dog: Planned, no release set

Some people want to play as a robot, or an android, or a brain in a jar piloting a robotic body, or as a dog.  The developers want to allow people to do all of that.  Unfortunately, there's a lot of changes that need to be made to get from the current state of the code to that highly desirable end-point.  Some changes have already been made, but there's nothing really visible yet.

As of 2024, there have been some very exciting changes to how we handle characters, allowing them to have non-humanoid limb configurations. This is useful for if trying to play as say, an intelligent dog, as dogs typically have four legs and no arms. However the work is still ongoing, and not yet ready for a general release. Much of the foundational work can be seen in the currently named "Work in Progress Limb Stuff" mod which ships with the game.

#### Losing limb in combat: sure, but the character is gonna die (almost always)

Often I see people asking when will combat be so dangerous that you will be able to lose your beloved arm or leg. Bullet points:

Main ways to lose a limb are:

You have so badly damaged/infected limb that your doctor/surgeon has no other way to save it but by cutting it - you have pretty high chances of survival, maybe even a guarantee if it's a faction doctor or you invest heavily in your own camp having a good surgeon with a tools reqired;

The second way is in a fight, as a result of a devastating, critical attack that deals an immense amount of damage to your limb - after all, arms and legs are not noodles; it requires a lot of force for the bone to be cut or for the shoulder to be detached. This, of course, would be followed by a massive pain shock from your beloved body part being detached, blood loss of immense intensity, and realization that whoever did this to you is still alive and trying to murder you. Because of all the aforementioned things, your chances of surviving such an event are close to zero unless you have the help of your dear comrade(s) (better to have a medic by occasion).

There are sure other ways, more technical, like mutating your arm into a crab claw or replacing it with a bionic drill, but that's another story.

In summary, the main things that stop us from introducing limb loss are, at first, a lack of NPC intelligence so they can help you if you experienced this loss, and second, a lot of our stuff is missing being tied to limb scores (something that you should have decreased because you have no limb). This is the main reason all limb stuff currently exists in a separate mod and is not part of the game.

NB: This talks specifically about average human experience; mutants are less limited to this (though a mouse or bear mutant would lose the limb in pretty similar fashion).

NB2: Please do not try to make it via EoC.

#### Bring back ICBM launch: Mod only

There used to be a partially implemented feature where you could break into an ICBM silo and hack the computer systems, then launch a missile at some target on the overmap. The results were incredibly underwhelming and didn't remotely represent the damage a many-kt warhead would cause, and was a frequent source of bugs, so it was removed.

It's not a valid idea to bring it back for a number of reasons.  One, the very large scale map destruction it would require would be a lot of work to implement, for incredibly low tangible benefit. Two, the feasibility of a survivor breaking into a nuclear silo and successfully launching a missile is negligible, the most likely situation is that the silo would have been put into some kind of lockdown or even destroyed once the staff deemed it infeasible to continue standing by, rendering a launch literally impossible. Even if that didn't happen, navigating the security mechanisms and failsafes the launch system would be expected to have is so difficult as to be effectively impossible, and that assumes that the launch systems are still accessible let alone operational. Three, the impact of an ICBM launch is the wrong scale for the game, if the desired feature is "destroy an area 150m across from a distance", which is what the previous feature amounted to, there are numerous options for achieving that without the ludicrous and impossible overkill embodied in a nuclear strike.  For example, an artillery or even mortar barrage has the capability of leveling a 100m or even larger area, and is a million times more feasible to acquire than a nuclear launch.

Code for ICBM launch is restored (with several bugfixes) and used in No Hope mod which is shipped with the game.

#### Add a lance charge for massive damage bonuses: yes, but not the way most people imagine it

The general understanding of how lances work is badly warped by depictions of jousting tournaments in books and movies, dungeons and dragons, and video games.

A successful lance charge is always going to result in the rider no longer holding a working lance, just think about the mechanics of it.

You're moving very quickly toward your target, as you approach you hit them with the lance and run them through, then you keep moving forward. At this point, either the lance breaks off, the rider drops the lance, or the rider is pulled off their mount by their grip on the lance.
Absolute best case, you're looking at expending one lance per attack, and retrieving your lances after the fight.

More likely, you're breaking lances every time you use them in a charge, and you need to re-craft a new set of lances every time you use them.

On the plus side, they would tend to do quite a lot of damage, so using a few to pick off several particularly tough targets or even just thin the numbers of a group of enemies could still be worthwhile, but not nearly the "lots more damage for free" concept that most people seem to think of.

Mechanically, this would replace the strength stat of the attacker with a calculation based on attacker body weight times vehicle speed, with a stability check to avoid being knocked off of the vehicle or having the entire vehicle knocked over.

#### Use a warhorse for lance attacks: mod only
This brings us to the topic of warhorses, which for all practical purposes do not exist in the dark days ahead scenario.  There might be a handful of people training warhorses on the planet, but they're so rare as to essentially not exist, and the chances of any trained warhorses surviving the Cataclysm are minuscule.

Mods of course are free to have warhorses or other warbeasts as they wish.

#### Use a lance on a bicycle or motorcycle: see [lance charge](#Add-a-lance-charge-for-massive-damage-bonuses-yes-but-not-the-way-most-people-imagine-it)

I'd expect the impact a bicycle rider can sustain without knocking themselves off their bike to be much lower than what a horse rider could handle.

I'd buy a lightly modified motorcycle allowing effective lance attacks, but see above regarding the shortcomings of lance attacks in general.

#### Storing blood for later to cure hypovolemia - Not realistic

Storing blood for later is far outside the reach of the survivor even if they knew how to. Blood needs to be heavily processed and stored in precise conditions, otherwise you'd end up with a bag of coagulated dried clot. The sorts of facilities necessary for this and the technical and medical knowhow is outside the scope of even a moderately size settlement, and that's assuming they even have a phlebotomist. Not to mention, drawing enough of your own blood to make a noticeable difference when reinjected later would leave you weakened for about as long as it would take to recover from hypovolemia anyway.

#### Direct transfusions from NPC followers: qualified maybe.

Coercing followers into being your personal blood bags is not going to happen. 

Assuming a willing donor, this would require someone with medical knowhow, matching blood types, about an hour of sitting around for the transfusion, and would make the recipient violently ill for about a day. Mistakes in this process (mismatched blood types, certain immune reactions, etc.) can easily be fatal, so this is still a high-risk option.

A slightly inferior but much more accessible and safer alternative is saline infusions to partially substitute for missing blood volume. This is currently in the game.

#### Add 3D printers and have them print guns and armor and car parts: Yes but no.

This breaks down into two possibilities, which tl;dr, neither really works.

The first option is you find design files for 3D prints. Without the internet, finding design files for 3D prints would be exceedingly difficult. The absence of a reliable method to search files on deserted computers, coupled with the fact that many useful designs are considered contraband, makes this option highly improbable. The idea of discovering files on discarded USB sticks has already been stretched to its limits and cannot be reasonably expanded to include a new method of crafting items. Even if one were to find a location with 3D printers, such as a house or workplace, most of the design files would are going to be for unhelpful items like figurines, toys, and replacement parts for random appliances, rather than items you want to produce as a survivor. 

The second option is creating new designs from scratch or existing designs. This involves finding a functional 3D printer, securing the necessary software (including drivers and design software), sourcing filament, and then creating original designs. This process could take anywhere from days to weeks for simple items, and potentially years for complex items like functional firearms. In most cases, there are easier-to-find alternatives for anything you could create with a 3D printer. Again, if your goal is 3D printing bottle openers and anime figurines, this would be a way to do that, but for useful survival-y things, it's not a viable option.

Finally, even if one were to overcome these issues and make 3D printed items, the quality of these items is generally going to be inferior to those crafted from metal, wood, or fabric. The resources and effort required to produce items with a 3D printer outweigh the benefits, making it impractical for a post-apocalyptic world. 

### Electrical power transmission
This covers several sub-suggestions that do or do not work for various reasons.

#### Bring back a municipal power grid: not feasible to implement

This isn’t feasible for several reasons. First, the assumption is that the grid is wrecked. After a disaster of this scale, it frequently takes dedicated teams of technicians working overtime days to weeks of work to restore the grid to working function, and that’s with near-unlimited resources, no additional disasters happening, and specific restoration plans in place. For a survivor it would be simply impossible, and even for teams of survivors it would take a prohibitive amount of time to do, and it would be much easier to simply cobble together a rough point-to-point power transmission system.

Second, this would be extremely difficult to support in the game engine because once you surpass a certain scale you need to keep every connected electrical device loaded and periodically processed in order to keep track of power usage. The only way I could see this working is if you ran through a series of missions to reclaim a town, and as part of the missions some power generation plant was assembled, and the town was wired up for it. At that point we could hand-wave the power usage tracking because the faction would be running the plant, not the player.

#### Short-range power transmission (scale of a single building): Largely implemented

Currently you can hook up multiple vehicles with jumper cables so they can transmit power, and this even works if some of the vehicles aren’t in the immediate area.

As of the 0.G "Gaiman" release there exists "appliances" which can be placed and interacted with using the construction and menus brought up with 'e’xamine instead of going through the vehicle menus. These "appliances" automatically form power grids and can be connected through special wire connections which can run through existing walls (if those walls would already have wiring provisions). There are "appliances" for most objects you would expect to encounter, including backup generators, household lighting, fridges, common power tools, etc. There are also a variety of cables (jumper cables, extension cords, etc.) which can be used to connect appliances to vehicles or other appliances, allowing power flow in either direction.

In addition to "appliances", there may also be "facilities".  Again, under the hood, facilities are going to be related to vehicles (admittedly, stationary vehicles) but are going to be built via the construction menu and interacted with as collections of terrain and furniture.  Facilities will hopefully allow for medium sized, powered buildings. As of this writing, there is not yet any support for "facilities".

### User interface

#### Bring back points pool in character generation: there is a reason we moved away from it

The main reason would be that it was not a fun system. To quote Venera3:

> The point system turning every character into a Heavy Sleeper Truth Teller Wool Allergy Stimulant Psychosis \<and now insert the traits you want to take\> is just shit design

The point system did not encourage players to try new approaches or roleplay but instead supported attempts to 'break' it, find a way to trick the system, and try to find as much profit within existing points as possible. Which brings us to issue number two, which is...

Points are a bad tool for balance. How many points is a bad back trait worth? 3? 2? 2.5? 2.76? π? Numbers have difficulties representing the abstract influence of a single trait on your entire playthrough. 

Luckily for us, a smart person coded a smarter solution to this, named Survivor mode, which is, put simply, "let's put the character against multiple in-game simulations and see how it would behave". Instead of assuming "dense bones = two points", we apply dense bones mutation to the test character and calculate how many blows it can take against a zombie. Is it more than average? It will help you in your game, so be "strong". Such evaluation will ensure the system will always work without much maintenance burden.

Is it a perfect system? No, even now, when this text is written, it still has a lot of rough edges. Is it a good system? Yes, very. Has the pool system been removed completely? No, you can still turn it on when you create the world, and it won't be removed unless it stands in the way of another changes. Is it possible to make a good system out of a pool system? Probably, but no one did.

Nota Bene, having a version of survivor mode with restrictions, a-la "you can't get traits that would make your defense higher than X" is desirable and would cover the niche of players who feel the open pool is too much for them, and want some sort of restriction.

#### The ability to select MP3s to play while listening to music: Too complicated

CDDA runs across a wide variety of platforms, each of which has their own special code for finding files on the local filesystem.  They also each have their own way to play an MP3, and generally they have multiple ways to play an MP3.  Adding support for selecting MP3s to play from inside the game requires that CDDA present a way to browse the local filesystem to find the MP3 files and a way to play them.  This code would be different for each platform and for each MP3 player; supporting it would be a maintenance nightmare full of weird and obscure bugs.

It might be possible for a really talented developer to come up some way to specify MP3s and an MP3 player in a platform independent way that didn't involve dragging in a huge amount of strange libraries, but all of the attempts so far have been platform specific and required a huge amount of extra library code.

### Monsters

#### Separate limbs for monsters: not useful

The suggestion is usually along the lines of, “add limbs to monsters so they can be crippled”. The problem with this approach is you can add crippling attacks much more easily by just applying a crippling effect to the monster, there’s no need to track per-limb HP for monsters to make it happen. Also, a limb system for monsters would be much more complicated than the one for players, since monsters have a more variable number of limbs, so we couldn’t even just use the code from the player-based system.

#### Targeted attacks on monster body parts: not useful

A similar suggestion is to allow the player to target specific monster body parts in order to achieve specific effects such as stunning with headshots or slowing by hitting the legs. A much simpler system is to have the player declare the effect they want rather than something indirect like targeting a limb, then we can be more flexible about how the attack plays out based on the combination of player abilities, weapon used, and monster type.

For a better outline on what we DO want to do, see https://github.com/CleverRaven/Cataclysm-DDA/issues/1565

#### Using zombie parts: mod only, mostly no

From a game balance perspective zombies are very abundant and should only be used under limited lore-friendly circumstances in consumables, trivial early items or with lasting risks and drawbacks.

#### Conversations with monsters and other things: Partially implemented

This is partially implemented already with monsters, items, furniture, as the player character can have conversations with them via NPC-style dialogue. However, support for dialogue functions is limited; you can't trade with monsters, and a significant quantity of edge cases for oddball functions remain unsupported for non-NPC talkers. Conversations with vehicles are entirely unsupported, meaning you can't talk with your car or some super-intelligent AI inside it.

Work is being done to enable all these conversations.  It's not hard work at this point, it's just tedious bits that need to get done.

#### Multi-tile monsters: desired but difficult

There are substantial technical challenges to a monster that exists on more than 1 tile at the same time, mostly dealing with pathfinding.  As people come up with solutions to these challenges, multi-tile monsters will become more likely.

Some multi-tile monsters, such as giant snakes, have fewer technical challenges and might arrive sooner.

### Item and terrain additions

#### Add food-bearing plant X: based on viability

The setting of the game is New England, with a strong emphasis on the Massachusetts area, so for example this places the [Hardiness Zone](https://en.wikipedia.org/wiki/Hardiness_zone) to reference at 6 or 7, and other indicators of plant viability (rainfall, first frost) should likewise be based on the typical Massachusetts stats. The other major thing to keep in mind is that when you are adding a tree or other plant, the primary impact is that you are expecting it to grow and produce harvestable fruit (or other harvestables) under the prevailing weather conditions for the area. Greenhouse-only plants will mostly die off in the first winter if not before; plants that require other winterizing actions will likewise die or be unhealthy if not cared for, so there would be no expectation of these plants thriving and producing harvestable materials. Due to this, the emphasis should be on plants that *thrive* in this region, not just ones that it is *possible* to grow.  It is not out of the question that we add a system that handles this in a more nuanced way, including requiring intensive gardening interventions, but until such a system is present plants that do not naturally thrive in Massachusetts should not be added, or they should be added in a form that does not produce harvests.

#### Add this item/recipe/etc: Go ahead

We often get recommendations to add a particular item or recipe to the game.  As long as the suggestion meets the game's inclusion criteria, this is generally permitted.  Due to the huge volume of suggestions like this that we get, the standard advice for this sort of thing is "add it yourself". That doesn't mean your suggestion isn't good or you're a bad person: quite the opposite, it means you've probably got a good idea. Anyone suggesting this is probably a contributor themselves, and is suggesting you join our illustrious ranks. It is meant in a complimentary fashion.

That said, the sheer amount of well-meaning suggestions we get is so high that these simple content addition suggestions are shut down quickly. That's not meant personally, it's just that if we don't moderate it heavily we are quickly flooded in excited suggestions that drown out conversation about the things people are currently actively working on.

### Vehicle additions

#### Caterpillar tracks/tank treads: As soon as someone gets around to it.

The way treads would work is they’d be very tough and have a very large amount of traction, along with a somewhat incidental property of needing to take up several squares. Also if any section of a track were destroyed, it would render the entire unit non-working, and probably keep your vehicle from moving at all.

Things that need to be implemented before treads proper are:

* Traction system, where there’s a trade-off between amount of traction and amount of friction, so higher traction means you have more grip, especially on loose surfaces like mud or sand, but you have higher friction, so it reduces your top speed and fuel efficiency.  This is mostly implemented.
* A way to handle installing and destroying multi-tile components in vehicles.  This is partially implemented.

#### Walker legs: mod only, if someone can figure out how to make it work

This doesn’t really fit in the game, vehicle sized robotics balancing and walking around is too far fantasy scifi for the game.

Someone is going to have come up with physics formula to convert engine power to motive speed for walker legs before this can be implemented.  Vehicles are probably going to have multiple z-levels for most walker leg designs.

#### I should be able to run my APC on solar panels

A modern solar/electric car that can even approach continuous locomotion is massively under weight compared to a standard car, and even then tends to only be tested in highway conditions where stops and starts are nonexistent. Even if we crank up the efficiency of cells to their theoretical maximum (by roughly 5x, which would be a reasonable thing to do with “quantum cells” from labs), we’re possibly reaching enough power to operate a small car at a reasonable duty cycle, but its not remotely reasonable to run a heavy utility vehicle from solar, much less an armored vehicle.

#### There should be helicopter flight simulators in military bases so we can learn to fly helicopters: No but yes

With the introduction of helicopters, people are upset that they can't learn to fly them but have to start the game with the appropriate trait.  I've done research on the early aircraft pioneers, and teaching yourself to fly involves crashing, a lot, and hopefully non-fatally.

As a work-around for that, people have proposed that military bases have flight simulators.  This is kind of reasonable, in that room-sized flight simulators with haptic feedback are a thing in the real world, but it's not happening because those things require a level of power and maintenance infrastructure that just isn't present after the Cataclysm, plus they have cryptic user interfaces that mean some random survivor can't just walk up to them and start the "Flight Training 101" simulation.

What would be more reasonable and possible would be finding a high end personal computer with a flight controller rig and a high end flight simulator like MS Flight Simulator 8.   I've known a dozen or more private pilots in my life, and every one of them had a system like this so they could practice flying even when they couldn't afford to take their private plane up.  A high end PC is certainly something that your average survivor can set up, and you can run them off a scavenged gasoline engine set up as a small electrical generator.  So this is definitely a solution, as soon as someone gets around to writing it.

#### We should be able to modify helicopters and other aircraft: Qualified yes

Small modifications of aircraft are fine, but significant changes (adding frames, changing the engine, etc) basically mean that you have a new aircraft design and you need to run some test flights to make sure that your new design is airworthy. In the real world this is normally handled by very complicated flight modeling software which (up until the last decade or so) still required manual verification by wind tunnel and flight testing, both of which are obviously unavailable following the Cataclysm. Even if someone was able to locate the software it would be unusable for its intended purpose without formal education in aeronautics.

For this to happen, we need code to detect significant changes in aircraft, code to simulate the chance that your new design isn't airworthy, and code to make you fall out of the sky during your test flights.

As of this writing, a variety of 'simple parts' can be added or removed from helicopters at will without rendering them non-airworthy. This is controlled by a simple JSON flag and can be easily changed as needed, if a good argument can be made that the addition/removal of such a part would never seriously impact a helicopter.

#### We should be able to make autogyros/hot air balloons/blimps/submarines/simple airplanes: Yes

All of these have been suggested.  All of them would be possible if someone wrote a bunch of code to enable them.

#### We should be able to pilot fixed-wing/tiltrotor aircraft: No

The mechanics of the game simply do not support vehicles moving at the speeds needed to stay flying. Helicopters are already fast enough to break many of the game's assumptions. Aircraft can be even faster than that.

The game only simulates 10 z-levels up or down, which is about 40 meters above the surface. This is a very low altitude even for a helicopter. For an aircraft, this is suicidal.

#### We should be able to use fixed-wing/tiltrotor aircraft to travel the map: Yes!

As some sort of 'fast travel' system, yes, with the obvious restrictions. Obviously fixed-wing aircraft need a place to takeoff, and another place to land. Some time-based events (including the real calendar time) need to be advanced while 'flying' to the destination, etc.

This just needs someone to write a bunch of code to enable it.

#### Vehicles should be able to span multiple z-levels: Yes

Vehicles can already span multiple z-levels, with a ground vehicle moving along a ramp from z0 to z+1 being an example.  What they can't do is have two vehicle frames with the same (x,y) co-ordinates but different z co-ordinates.  This will hopefully get fixed (it's a huge amount of work inside the vehicle code), and people will be able to build double-decker bus deathmobiles.

### Gun Behavior

#### Hit multiple targets with a shotgun: Kind of yes, but really no.

For some examples of 12-gague 00-shot patterns, take a look at [this](http://www.theboxotruth.com/the-box-o-truth-20-buckshot-patterns/).

TL;DR at 45 yards, which is outside the maximum effective range of the round in the first place (I’ve seen anywhere from 25 yards to 35 yards claimed), the spread was between 27 and 33 inches. Even at this extreme range, the spread is still less than one in-game square, so you’re effectively never going to hit two targets standing side-by-side. What might happen is you get some kind of “graze” one one target, and the shot that didn’t hit the target will continue and possibly hit another target behind it.

#### Hit multiple targets with a very large, penetrating round: Kind of yes, with specific qualifications

Under the existing system, rounds that strike a creature apply all their kinetic energy (damage) to the creature and cease to exist. Explosive rounds get around this in-game by alternatively creating an explosion (handled differently) and otherwise sending out shrapnel (spawns new 'bullets' radiating outwards from the explosion).

There are certain very large-caliber rounds in the game which could penetrate a human body or shambling corpse and still have damaging or lethal amounts of kinetic energy after passing through, but even these rounds dissipate on hitting the first creature they encounter. (Note that some terrain and furniture can be penetrated with partial damage reduction, but this system does not have widespread support and is not extendable to creatures)

Adding this is desired, and it would be possible if someone wrote a bunch of code to enable it.

This would not be applicable to most rounds, including basically all rounds smaller than 50cal (0.5 inch, or ~12.7mm bullet diameter). Such small rounds simply don't have enough kinetic energy to wound a second person after passing through one body, even in highly optimistic scenarios.

### Environment

#### Bring back acid melting items: Grossly unrealistic

There’s no level of acid strength where it makes sense for it to dissolve large volumes of items on the ground, but isn’t invariably fatal on contact with a survivor. If someone has some actual facts about how acids work when in contact with large volumes of various substances, including flesh, that contradict my understanding of this, we can talk, but referring to the previous state of the game or other games isn’t going to get you anywhere.

The previous mechanic where it was sort of dangerous to the survivor but massively damaging to items was a game-ism, and it’s most likely not coming back. The popularity point of view seems to be a wash as well, because there seem to be at least as many people strongly in support of removing item-melting as there are who want it back.

#### Bring back acid rain: Partially

The old implementation of acid rain had a massive world consistency flaw, given the lethality of acid rain when exposed to it, and its frequency, nearly every animal on the planet should be dead after about a month. This includes most of the invasive creatures, because it would kill them too. There are two options for mitigating this issue. One is to make it incapacitating instead of lethal, in which case animals would be harmed by it, but not killed, so in general populations would be unaffected. The other is to make it a highly localized or rare phenomenon, and kill any creatures (and probably plants) in the area when it occurs, whether the player is there or not. Again, if it’s rare populations would recover over time and be mostly unaffected. With the second event though, you’d get cool trails of death scattered around in your game world.

## Failed Rationalizations

These aren’t suggestions, but arguments that comes up in support of suggestions or alterations to the game. They come up constantly so I’m putting a note about them here.

In the context of this game, failed rationalization means "if something was added before, it doesn't mean we should add something similar again" or "if someone wants to add something to the game, they should review it separately from the rest of the game, not relying on other parts of the game".
The game is open-source and community-built and changes itself all the time: Whales Cataclysm and DDA have a 10-year difference and ~40k merged changes from ~2k different contributors. The game changes all the time; we've got an entire setting revamped, which means there are a lot of items that were added a long time ago that can't be added anymore, and honestly shouldn't be: mininuke manhacks, SPIW weapons, healing royal jelly, superalloy dog harnesses, etc.

More examples:

#### “Seeing as we have nanobots and power armors…”, “We have teleportation, so it’s not unreasonable to have…”: Irrelevant

If you make this argument, you will not only not make the intended point, since the argument is nonsensical, but you will also damage your credibility with me personally, and with many of the other contributors as well. I am absolutely sick of reading this, and I am even more sick of responding to it, so I’ll just refer to this post from now on.

The supposed lack of “consistency” between super-science elements of the game and mundane elements of the game is intended. The setting of the world is current-day New England (America if you don’t recognize the region name), with isolated science fiction elements, such as super-science items that generally appear in “secret research labs”* or deployed with military units. The existence of super-science items does not imply that every aspect of daily life is imbued with elements of fantastical science.

\* Yes we do a fairly bad job of labs feeling like “secret research facilities”, but bear with us and/or make suggestions about making them feel more like that, that is the intent.

#### "Player can craft a X, so they should also be able to craft a Y"

This kind of argument is invalid for several reasons.

Just because there’s a contradiction i.e. “cordless drills and gunsmithing tools are equally hard to make, but we can only make cordless drills” doesn’t imply that the correct action is allowing crafting of gunsmithing tools, it’s equally likely that craftable cordless drills were a mistake.

This argument generally has a tenuous or non-existent relationship between the items in question. Frequently it’s an assertion that X is “complicated” and Y is either “simple” or “also complicated”, this is not sufficient for craftability of X to imply craftability of Y.

Hoisted from a [topic where it came up again](https://discourse.cataclysmdda.org/t/i-cant-see-the-difference-between-firearm-and-gunsmith-repair-kits).
