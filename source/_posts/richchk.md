---
title: RichChk - Python StarCraft map editor
date: 2024-10-12 19:33:47
tags: 
- StarCraft
- Python
- RichChk
- Blizzard
---

## Introduction

// find a way to tie this back into general software engineering practices: open source, data modeling, immutability, functional programming, typing, unit tests
// clearly outline the entirety of the problems that your tool is addressing
// what is your overall outcome you're looking to achieve?

In this article I will introduce [RichChk](https://github.com/sethmachine/richchk)--a Python library that allows directly editing StarCraft maps in order to create custom games.  Custom games can be thought of as mods built on top of the StarCraft engine that completely change how the game plays.  While this article is mostly related to StarCraft, my project demonstrates how to apply software engineering best practices such as data immutability, functional programming, static typing, unit tests, code linters, GitHub Actions/Workflows, etc.  Indeed, I believe that experienced software engineers and coders have a superpower that can be applied with multiplicative force to build high quality software that can immensely benefit others unrelated to usual business endeavors.  I will first give a brief overview of StarCraft and its rich history of custom games.  After that I will dive into RichChk: what it is, why I made it, and how it works.  Finally, I will provide a brief survey of other existing tools comparable to RichChk and how these compare and contrast.  

## StarCraft Custom Games
[StarCraft](https://us.shop.battle.net/en-us/product/starcraft) was first released by Blizzard Entertainment in 1998 as a science fiction [real-time strategy (RTS)](https://en.wikipedia.org/wiki/Real-time_strategy) game where players contend for the fate of the galaxy controlling 3 different races: Terran, Zerg, and Protoss.  Two key factors propelled StarCraft to massive popularity: (1) free online multiplayer with [Blizzard's revolutionary Battle.net](https://en.wikipedia.org/wiki/Battle.net) and (2) the ability players to create custom games within the StarCraft engine via [StarEdit](https://us.forums.blizzard.com/en/starcraft/t/public-staredit-download/457).  A typical smattering of lobbies of online multiplayer custom games is shown below:   

![StarCraft custom games lobbies in 2024](richchk/starcaft-custom-games-example.PNG)

The community eventually developed a more powerful map editor called [ScmDraft 2](http://www.stormcoast-fortress.net/cntt/software/scmdraft), which [Blizzard officially endorsed in June 2019](https://us.forums.blizzard.com/en/starcraft/t/staredit-deprecation-in-patch-1-23-0/223).  During the early 2000s, a massive online community grew dedicated exclusively playing these custom maps, which were often implementations of other video games ([Elder Scrolls Oblivion RPG](http://sc.nibbits.com/maps/project/78115/oblivion-rpg-2013)) or universes (e.g. Lord of the Rings StarCraft war maps).  Even [MOBA](https://en.wikipedia.org/wiki/Multiplayer_online_battle_arena) games like [Dota 2](https://www.dota2.com/) and [League of Legends](https://www.leagueoflegends.com/en-us/) can trace their ancestry to the original StarCraft custom map, [Aeon of Strife](https://starcraft.fandom.com/wiki/Aeon_of_Strife_(map)).  Fast forward to 2024, and while the custom games community is much smaller, there is still a large active group on various forums, such as [staredit.net](http://www.staredit.net/) and new custom games are being regularly produced and updated.  

### StarCraft Map Files

Each unique StarCraft custom game is contained entirely within a map file.  These map files are stored in a [binary MPQ archive file](https://encyclopedia.pub/entry/37738) with either a .SCM or .SCX file extension.  Contained within the MPQ archive, the map data itself is stored as a [CHK file](https://www.starcraftai.com/wiki/CHK_Format), which is a compact binary format that represents all the data required to play a single instance of a game of StarCraft 

Without 3rd party tools, the binary map file is not human-readable and can only be opened with StarEdit or ScmDraft  The map file contains everything needed for a custom game to function: terrain, unit/structure placements, locations data, etc., but also the business logic of the map which is expressed through Triggers.  Triggers can be thought of as a high level scripting language to control the StarCraft engine.  In some sense, maps can be thought of combining the frontend (how the map looks) and backend (how the map plays) into a single file.  This is ideal for peer to peer transmission and downloading of maps, especially when bandwidth was quite limited in the early days of online gaming.  However, this is not ideal from the perspective of creating and maintaining a custom game, as it makes it hard to decouple the visuals from the business logic.  

RichChk solves the problem of separating the map visuals from its business logic.  RichChk accomplishes this by allowing map makers edit all aspects of a StarCraft map in Python without ever opening a GUI map editor.  This allows for using traditional software engineering practices like version control, and decouples the map's visuals from its business logic.

## What is RichChk?

RichChk is a Python library that directly editing StarCraft's CHK files, and hence editing StarCraft map files in Python without ever needing to open a GUI program.  RichChk provides a semantically **rich** and human readable model of the **CHK** format, hence the naming.  As stated earlier, the traditional creation of StarCraft maps is done in a visual GUI program like StarEdit or ScmDraft.  GUI editors are ideal for visual aspects such as the terrain or unit placements.  However, the business logic of a map is done through Triggers.  This is where the map editors fall short: there is no way to organize or re-use Triggers as one might do for a modern codebase.  


![ScmDraft 2 Map Editor](richchk/scmdraft-open-example.png)

The above image depicts [ScmDraft 2](http://www.stormcoast-fortress.net/cntt/software/scmdraft), a GUI based program to visually create and edit StarCraft custom games.  RichChk offers the same exact functionality as the GUI editor, but with the advantage of being able to modify the map purely in Python.

[RichChk](https://github.com/sethmachine/richchk) is an open source Python library I developed for the purpose of creating, editing, and maintaining StarCraft custom maps.  RichChk provides human readable, fully typed, and [b]rich[/b] semantic model of the [StarCraft CHK format](http://www.staredit.net/wiki/index.php/Scenario.chk), which contains all the data for a StarCraft map/custom game.  Thus, why the library is called RichChk--a rich representation of the CHK format.  RichChk is powerful and unique in several aspects.  RichChk allows editing all aspects of a StarCraft map in Python and save directly to a StarCraft map file without ever opening a GUI map editor.  RichChk


RichChk is a Python library that semantically models the [StarCraft Brood War CHK format], allowing for editing all aspects of a map entirely in Python.

RichChk is a massive ergonomic and efficiency improvement for creating StarCraft custom maps.  This is because it has the ability to edit all sections of the CHK file in Python and save directly to an existing StarCraft map file (an MPQ archive typically with a .SCX/.SCM file extension).  There are existing ways to write triggers outside the StarEdit/ScmDraft GUI, but these are text based and must be copy and pasted back into the

RichChk allows for directly editing StarCraft's CHK files, a compact binary format that represents all the data required to play a single instance of a game of StarCraft.  Traditional creation of StarCraft maps is done in a visual GUI editor, which is ideal for visual aspects such as the terrain or unit placements.  However, the business logic of a map is done through Triggers, which can be thought of as a high level procedural scripting language on top of the StarCraft engine.  This is where the map editors fall short, as there is no way to organize or re-use Triggers as one might do for a modern codebase.

RichChk fills in this gap, by allowing all of a custom game's business logic to be written in Python and be translated directly to the binary CHK format that StarCraft reads.  This is a massive user experience improvement for mapmakers, allowing them to leverage all of the advantages of modern coding practices such as version control, re-using code, high modularity and maintainability, unit tests, semantic versioning, etc.

## Why RichChk?

I made RichChk to provide myself a powerful and flexible way to create and maintain custom game projects for StarCraft.  This can be viewed as a massive ergonomic and user experience improvement.  Unlike the visual editors, RichChk is 100% Python, and thus can be used to maintain any project like a modern codebase.  This includes version control and all the other advantages of using a well known programming language like Python.  The binary map file is not human readable or interpretable directly without using an external program.  

One of the difficulties of making complex games in StarCraft is working with the [Triggers engine](http://classic.battle.net/scc/faq/triggers.shtml), which arguably is the most important aspect of a custom game, as it contains all the business logic for the modifying the game behavior.  For any non-trivial map, creating, maintaining, and updating these systems quickly become unwieldy.  Consider the following trigger as shown in the ScmDraft editor:

![ScmDraft Trigger Example for Garrison](richchk/scmdraft-trigger-example-garrison.png)

The trigger consists of 3 parts, two of which are shown.  On the left are the conditions, which are the boolean statements that all must be satisfied for the trigger to execute.  On the right are the actions, which is what will happen if the conditions are true.  This single trigger is just one part of a much larger trigger system for "Garrisons" in this custom map: if a player controlled building comes under attack, defending units will automatically emerge to protect the structure.  This particular trigger handles the case where an opposing player captures an enemy structure.  Conditions and actions must be added one by one in the GUI from a dropdown menu.  While each trigger can be copied to help reduce boilerplate, there is no way to reference other triggers, variables, data structures, or external data to configure them.  For a Garrison system like this, there can be an upward of 4 triggers per player to model each state per structure on the map, which is roughly 32 triggers per player.  At this scale, it becomes a mind-numbing task to manually copy each trigger and replace the data so it works for each player.  This is also prone to human error in manually copying the data.  Imagine if there is a mistake or a future enhancement needs to happen!  

Now consider how this system could be built in RichChk using pure Python:

```python garrison_triggers.py
def generate_garrison_captured_for_player(
    garrison: Garrison, player: PlayerId, dc_timer: AllocatedDeathCounter
) -> RichTrigger:
    return RichTrigger(
        _conditions=[
            DeathsCondition(
                _group=dc_timer.player,
                _comparator=NumericComparator.EXACTLY,
                _amount=1,
                _unit=dc_timer.unit,
            ),
            BringCondition(
                _group=PlayerId.FOES,
                _comparator=NumericComparator.AT_LEAST,
                _amount=1,
                _unit=garrison.structure,
                _location=garrison.location,
            ),
            BringCondition(
                _group=player,
                _comparator=NumericComparator.AT_LEAST,
                _amount=1,
                _unit=UnitId.MEN,
                _location=garrison.location,
            ),
        ]
        + generate_condition_for_undefended_garrison_by_foes(garrison),
        _actions=[
            SetDeathsAction(
                _group=dc_timer.player,
                _unit=dc_timer.unit,
                _amount=0,
                _amount_modifier=AmountModifier.SET_TO,
            ),
            GiveUnitAction(
                _from_group=PlayerId.FOES,
                _to_group=player,
                _amount=1,
                _unit=garrison.structure,
                _location=garrison.location,
            ),
            ModifyUnitHitpointsAction(
                _group=player,
                _unit=garrison.structure,
                _amount=1,
                _percent=100,
                _location=garrison.location,
            ),
            DisplayTextMessageAction(
                _text=RichString(_value=_GARRISON_CAPTURED_MESSAGE)
            ),
            MinimapPingAction(_location=garrison.location),
            PreserveTrigger(),
        ],
        _players={player},
    )
```

This is the same trigger as early, but coded up in Python.  It is indeed verbose, but it is fully parameterized for each player via the `player: PlayerId` argument, and even supports different types of garrisons (structure type and unit type that comes out) via the `garrison: Garrison` data structure (a custom data structure used for convenience and unrelated to RichChk).  The `dc_timer: AllocatedDeathCounter` is used to increment state surrounding the garrison system.  With a simple for loop over players and garrisons, this can instantly generate all the triggers needed for allowing a player to capture an enemy's garrison structure.  If any modifications are needed, one simply edits this method and uses RichChk to create the new map.  E.g. playing a sound file to indicate the structure has been captured.  
 
 
## How RichChk works

RichChk implements the CHK format as fully detailed in [staredit.net's Scenario.chk wiki](http://www.staredit.net/wiki/index.php/Scenario.chk).  The CHK is extracted from a StarCraft MPQ file (.SCM or .SCX file extension) using [StormLib](http://www.zezula.net/en/mpq/stormlib.html), an open source C++ library for reading and writing MPQ archive files.  


The CHK file is then decoded, section by section, into Python dataclass equivalents via the [ChkIo#decode_chk_file](https://github.com/sethmachine/richchk/blob/master/src/richchk/io/chk/chk_io.py#L56C9-L56C24), which get stored as a list in a [DecodedChk](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/chk/decoded_chk.py#L12) object.  The actual decoding is a while loop operation that iterates over the bytes of the CHK file until no more sections are found, as shown in [ChkIo#_decode_chk_byte_stream](https://github.com/sethmachine/richchk/blob/master/src/richchk/io/chk/chk_io.py#L94).  For sections that are directly modeled by RichChk, these correspond to specific dataclasses like [DecodedTrgSection](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/chk/trig/decoded_trig_section.py#L140) for the [CHK TRG section](http://www.staredit.net/wiki/index.php/Scenario.chk#.22TRIG.22_-_Triggers).  Most sections are not currently modeled, and handled by a special [DecodedUnknownSection](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/chk/unknown/decoded_unknown_section.py), which is a wrapper around the unparsed bytes of that section.  These sections can be safely ignored until future contributions decide to model them, while still being able to edit the parsed sections.

The `DecodedChk` is still just a wrapper around the binary CHK format, and isn't editable even represented as Python objects.  Thus, the [RichChk](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/richchk/rich_chk.py#L24) is introduced to model a high level, editable representation of all CHK data (not to be confused with the library name!).  Just as the `DecodedChk` contains all the binary CHK sections of a map file, the `RichChk` contains all the corresponding rich CHK sections, or more specifically a list of [RichChkSection](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/richchk/rich_chk_section.py).  Compare the trigger models in [richchk.model.richchk.trig](https://github.com/sethmachine/richchk/tree/master/src/richchk/model/richchk/trig) to the lower level models in [richchk.model.chk.trig](https://github.com/sethmachine/richchk/tree/master/src/richchk/model/chk/trig).  The former has a plentiful taxonomy of classes and enums modeling how a human would understand and create triggers, while the latter is just a wrapper around the compact binary format triggers are stored in.

## Coding Best Practices

### Unit tests

### Cross platform support

### Static typing

### Immutability

### Functional programming

## Existing mapping tools

Also see: https://scmscx.com/map/gRdD9yb8 

RichChk diagram: https://lucid.app/lucidchart/1c68933a-2a30-4199-bd1e-e7ee77a178e5/edit?viewport_loc=-1682%2C2456%2C3022%2C1753%2C0_0&invitationId=inv_967bc678-1fc2-4980-8308-480e27f117f9 

The traditional way to edit maps and create custom games are using a visual GUI program.  The two most popular are StarEdit (Blizzard's own official editor, but now deprecated) and [ScmDraft 2](http://www.stormcoast-fortress.net/cntt/software/scmdraft).  These programs are a great way to create the visuals of a map: the terrain, precise unit placements, locations data (used to identify unique regions in the map), etc.  But they are not good for creating [Triggers](http://classic.battle.net/scc/faq/triggers.shtml), the business logic and "code" of a custom game.  They require using a painstaking visual UI which has no notion of higher level programming primitives like variables or referencing other trigger/map data.  Further, there is no way to organize or logically group triggers; they can only be viewed based on the player they execute for.  

ScmDraft 2 attempted to address shortcomings by providing a text based trigger editor, but this now requires external tools to properly write these triggers, and a manual way to copy and paste them into the map.  That also does not address the metaprogramming issue--once a map goes to any significant complexity, it becomes a Herculean task to maintain and update the map.  There is no way to easily search the triggers since ScmDraft 2 only provided a text based format to enter triggers, but all the same problems remain.  

[insert image of SCMDraft]
[image of ]

On top of ScmDraft 2, the StarCraft map making community has periodically released its own tools to improve creating triggers.  Some of the most popular tools include [LangUMS](https://github.com/LangUMS/langums), [TriGen](http://www.staredit.net/topic/17244/), [Oreo Triggers](http://www.staredit.net/topic/15156/), [ProTRG](http://www.staredit.net/topic/9581/), [LIT](http://www.staredit.net/topic/16432/), and [Macro Triggers](http://www.staredit.net/topic/2970/). Besides LangUMS, these tools act as text based trigger pre-processors, allowing mapmakers to create triggers in a high level text based format or even coding language (e.g. Python for TriGen, PHP for Oreo Triggers, Lua for LIT, etc.).  But, these tools are just pre-processors.  They cannot edit a StarCraft map directly and instead produce a text based output that must be manually copy and pasted into the ScmDraft 2 trigger editor.  Another drawback of these tools is that many of the authors have not made these tools open source.  This is problematic for a number of reasons: inevitably the authors may stop maintaining the projects, the community cannot themselves makes fixes or re-build the tools for new or different operating systems (e.g. macOS), and the tools may inevitably break as StarCraft receives any updates.  Yet another issue is that without open sourcing the code, others cannot learn, gatekeeping valuable knowledge of the internals of StarCraft map making and modding.  These are some my largest criticisms of existing mapping tools, and why I have made RichChk 100% open source and freely available.     

LangUMS is perhaps the most interesting and powerful trigger editing program available but even it has several critical drawbacks: it is its own high level language built on top of C++, and not an actual modern programming language, meaning users need to essentially learn a brand new language.  Because the engine is written in C++ (and completely unrelated to the actual LangUMS scripting language), it is not transparent how to improve or update the functionality and requires expert knowledge in learning two separate subjects: C++ and the LangUMS language.  Yet another issue is that LangUMS tries to solve two different problems at once: editing a StarCraft map and providing a higher level framework on top of StarCraft.  By conflating those together, it prevents map makers from having full transparency and control in how the actual triggers are manifested.  The project also only works on trigger data, but there is high value in being able to edit static data like unit settings at the same time (e.g. so triggers could reference static data).  LangUMS is not cross platform: it only works for Windows and it is unclear if it could be used on other operating systems like macOS or Linux.  Finally, it has been 7+ years since the author (the only contributor to the entire project) has updated contributions.  These all make it a highly unlikely tool to be ever adopted outside of an academic context.   

## Enter RichChk

To address the key shortcomings of [existing mapping tools](#existing-mapping-tools), I created RichChk.  As stated earlier, RichChk is a Python library that allows editing the CHK directly (and any/all the data contained within it, including triggers).  RichChk does not do anything that cannot be done in any of the existing mapping tools or ScmDraft 2, but it does provide a far more ergonomic and structured way to create, update, and manage complex custom games and mapping projects.  RichChk can be thought of a massive user experience (UX) improvement for map making.  This is accomplished by provide a rich semantic model of the CHK itself with full static typing of all StarCraft concepts in the library.  And because RichChk is not a pre-processor, but a model of the CHK format, this allows going straight from Python code to an update map file without ever opening a GUI program.  No more copy and pasting triggers!  This creates a fast and responsive iteration loop for developing a map: edit Python code, update the map, test in StarCraft, and repeat.  

Nonetheless, RichChk is not without its flaws.  It requires installation and knowledge of the Python programming language.  This is not a trivial requirement, but I contend that the ability to create custom maps and program are highly correlated skills, so any motivated mapmaker could learn enough Python to be productive.  Any complex mapping project would benefit immensely from moving triggers to a modern programming language and taking full advantage of version control, IDE highlighting, etc.  

Below is the quintessential "Hello world!" example, show how RichChk can be used at the most basic level.  This script displays "Hello world!" to all the players when the map is played in StarCraft.  

```python hello_world.py
from richchk.editor.richchk.rich_chk_editor import RichChkEditor
from richchk.editor.richchk.rich_trig_editor import RichTrigEditor
from richchk.io.mpq.starcraft_mpq_io_helper import StarCraftMpqIoHelper
from richchk.io.richchk.query.chk_query_util import ChkQueryUtil
from richchk.model.richchk.str.rich_string import RichString
from richchk.model.richchk.trig.actions.display_text_message_action import (
    DisplayTextMessageAction,
)
from richchk.model.richchk.trig.conditions.always_condition import AlwaysCondition
from richchk.model.richchk.trig.player_id import PlayerId
from richchk.model.richchk.trig.rich_trig_section import RichTrigSection
from richchk.model.richchk.trig.rich_trigger import RichTrigger

INPUT_MAP_FILE = "maps/base-map.scx"
OUTPUT_MAP_FILE = "generated-maps/hello-world-generated.scx"


def generate_display_hello_world_trigger() -> RichTrigger:
    return RichTrigger(
        _conditions=[AlwaysCondition()],
        _actions=[DisplayTextMessageAction(_text=RichString(_value="Hello world!"))],
        _players={PlayerId.ALL_PLAYERS},
    )


if __name__ == "__main__":
    mpqio = StarCraftMpqIoHelper.create_mpq_io()
    chk = mpqio.read_chk_from_mpq(INPUT_MAP_FILE)
    new_trig = RichTrigEditor().add_triggers(
        [generate_display_hello_world_trigger()],
        ChkQueryUtil.find_only_rich_section_in_chk(RichTrigSection, chk),
    )
    new_chk = RichChkEditor().replace_chk_section(new_trig, chk)
    mpqio.save_chk_to_mpq(
        new_chk, INPUT_MAP_FILE, OUTPUT_MAP_FILE
    )
```

This example highlights a few key traits of RichChk:

* Data and map file immutability: editing a map is done by taking an existing map and appending new data to it.  The original map file is never meant to be modified directly.  
* Separation of data and code: the class [RichTrigSection](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/richchk/trig/rich_trig_section.py#L140) only contains data for triggers, while [RichTrigEditor](https://github.com/sethmachine/richchk/blob/master/src/richchk/editor/richchk/rich_trig_editor.py) and under the hood [RichChkTrigTranscoder](https://github.com/sethmachine/richchk/blob/master/src/richchk/transcoder/richchk/transcoders/richchk_trig_transcoder.py#L35) are responsible for reading and writing the trigger data back to the CHK format
    * Similarly, the class [RichChk](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/richchk/rich_chk.py#L24) contains all the CHK data, but does not know how to edit or update the CHK data.  This is accomplished through separate helper classes like [StarCraftMpqIo](https://github.com/sethmachine/richchk/blob/master/src/richchk/io/mpq/starcraft_mpq_io.py#L15), [RichChkIo](https://github.com/sethmachine/richchk/blob/master/src/richchk/io/richchk/richchk_io.py), and [RichChkEditor](https://github.com/sethmachine/richchk/blob/master/src/richchk/editor/richchk/rich_chk_editor.py)
* Static typing and models for all parts of the StarCraft map: e.g. the [PlayerId](https://github.com/sethmachine/richchk/blob/master/src/richchk/model/richchk/trig/player_id.py#L9) enum provides all possible players that a trigger can affect or execute for. 