+++
title = "💻 Software"
menu = "main"
+++

# Security Stuff

## Recent

Working on a prototype for usig LLMs for automatic exploit generation in EVM:

* [quimera](https://github.com/gustavo-grieco/quimera): data-driven exploit generation for Ethereum smart contracts using LLMs and Foundry.

![quimera screenshot](https://i.imgur.com/kZZiNTr.png "300px")

Worked on some of the most influential smart contract fuzzers for EVM:

* Led the development of [echidna](https://github.com/crytic/echidna).
* Provided assistance on the [medusa](https://github.com/crytic/medusa) development.

![echidna screenshot](https://i.imgur.com/saFWti4.png "300px")

Also have some experience with symbolic execution:

* Regularly contribute [improvements and fixes to hevm](https://github.com/argotorg/hevm/pulls?q=is%3Apr+author%3Agustavo-grieco), the symbolic execution engine for the EVM that powers Echidna.
* In the past, added [small features and fixed bugs in Manticore](https://github.com/trailofbits/manticore/pulls?q=is%3Apr+author%3Agustavo-grieco).

Recently combined some of these tools (Echidna, hevm and Certora) in a [formal verification effort for the ABDK Math 64.64 library](https://github.com/gustavo-grieco/abdk-math-64.64-verification).

## Old

* Led the development of [QuickFuzz](https://github.com/CIFASIS/QuickFuzz)
* Created and developed [VDiscover](https://github.com/CIFASIS/VDiscover).

# Fun Stuff

I'm a detective-story geek who created [mystery-o-matic.com](https://mystery-o-matic.com), a website where you can solve a daily murder mystery in 5 minutes. All [the code and website](https://github.com/mystery-o-matic/mystery-o-matic.github.io/) are open-source!

## Retro gaming

I'm part of the [ScummVM](https://www.scummvm.org/) team, volunteering as the main developer for the reimplementation of the following engines/games:

### Private engine
Point and click game engine developed by [Brooklyn Multimedia](https://www.mobygames.com/company/2861/brooklyn-multimedia/) for a single game:
* [Private Eye (1996)](https://www.mobygames.com/game/7117/private-eye/)

![private eye screenshot 1](https://www.scummvm.org/data/screenshots/private/private-eye/private-eye_win_en_1_1_full.png "350px") ![private eye screenshot 2](https://www.scummvm.org/data/screenshots/private/private-eye/private-eye_win_en_1_4_full.png "350px")

### Hypno engine
Game engine developed by Hypnotix used for point and click and rail shooters:
* [Wetlands (1995)](https://www.mobygames.com/game/862/wetlands/)
* [Marvel Comics Spider-Man: The Sinister Six (1996)](https://www.mobygames.com/game/34907/marvel-comics-spider-man-the-sinister-six/)
* [Soldier Boyz (1997)](https://www.mobygames.com/game/8383/soldier-boyz/)

![wetlands screenshot](https://www.scummvm.org/data/screenshots/hypno/wetlands/wetlands_dos_en_1_2_full.png "350px") ![spider man sinister six screenshot](https://www.scummvm.org/data/screenshots/hypno/sinistersix/sinistersix_dos_de_1_5_full.png "350px")

### Freescape engine
One of the first 3D game engines developed by Incentive Software for a number of games:
* [Driller/Space Station Oblivion (1987)](https://www.mobygames.com/game/4933/space-station-oblivion/)
* [Dark Side (1988)](https://www.mobygames.com/game/21802/dark-side/)
* [Total Eclipse (1988)](https://www.mobygames.com/game/6712/total-eclipse/) / [Total Eclipse 2 (1989)](https://www.mobygames.com/game/71411/total-eclipse-special-edition/)
*  [Castle Master (1990)](https://www.mobygames.com/game/2155/castle-master/) / [Castle Master 2 (1990)](https://www.mobygames.com/game/2169/castle-master-castle-master-ii-the-crypt/)
* [3D Construction Kit / Virtual Reality Kit (1991)](https://www.mobygames.com/game/391/virtual-reality-studio/)

![total eclipse screenshot](https://www.scummvm.org/data/screenshots/freescape/totaleclipse/totaleclipse_dos_en_1_2_full.png "350px") ![castle master screenshot](https://www.scummvm.org/data/screenshots/freescape/castlemaster/castlemaster_dos_en_1_1_full.png "350px")

### Colony engine
One of the first game engines to offer real-time 3D graphics with free movement, created by [David Alan Smith](https://davidasmith.medium.com/the-colony-a-memoir-d46a0e08ec60) and published by Mindscape for a single game:
* [The Colony (1988)](https://www.mobygames.com/game/3489/the-colony/)

### EEM engine
Point and click detective game engine developed by [Stormfront Studios](https://www.mobygames.com/company/395/stormfront-studios/) and published by EA*Kids for two games:
* [Eagle Eye Mysteries (1993)](https://www.mobygames.com/game/14171/eagle-eye-mysteries/)
* [Eagle Eye Mysteries in London (1994)](https://www.mobygames.com/game/19256/eagle-eye-mysteries-in-london/)

### SCUMM engine: Rebel Assault games
Reimplementation of the FMV rail shooters developed by [LucasArts](https://www.mobygames.com/company/72/lucasfilm-games/), as part of the existing SCUMM engine:
* [Star Wars: Rebel Assault (1993)](https://www.mobygames.com/game/272/star-wars-rebel-assault/)
* [Star Wars: Rebel Assault II - The Hidden Empire (1995)](https://www.mobygames.com/game/5800/star-wars-rebel-assault-ii-the-hidden-empire/)

## Source code reconstruction

Outside of ScummVM, I completed a full [source code reconstruction](https://github.com/neuromancer/my-teacher-is-an-alien-re) of [Bruce Coville's My Teacher Is an Alien (1997)](https://www.mobygames.com/game/157029/bruce-covilles-my-teacher-is-an-alien/), a point and click adventure developed by [7th Level](https://www.mobygames.com/company/793/7th-level-inc/) and Byron Preiss Multimedia. The reconstructed code is bug-for-bug faithful: it produces code identical to the original at the CPU instruction level when compiled with the same tools, verified using [binary-comp](https://github.com/neuromancer/binary-comp), a tool I created to compare binaries function by function.

![my teacher is an alien screenshot 1](https://github.com/user-attachments/assets/a0ac2b7e-36ea-4e8a-b6ad-d5e4988e2a83 "350px") ![my teacher is an alien screenshot 2](https://github.com/user-attachments/assets/48145419-1bd6-4d46-8938-dbad3834d920 "350px")

I'm also reconstructing the source of the two [Wing Commander](https://www.mobygames.com/game-group/wing-commander-series) games developed by [Origin Systems](https://en.wikipedia.org/wiki/Origin_Systems), as shipped in *Wing Commander: The Kilrathi Saga (1996)*. Both projects rebuild the original Win32 executables with the era's Microsoft Visual C++ toolchain, and both ship a native SDL2 port for Windows, Linux and macOS that runs on either Kilrathi Saga or original DOS game data, with an optional OpenGL renderer that draws space objects at output resolution:

* [wc1-re](https://github.com/neuromancer/wc1-re): reconstruction of [Wing Commander (1990)](https://www.mobygames.com/game/3/wing-commander/). All 1,472 identified functions are accounted for.
* [wc2-re](https://github.com/neuromancer/wc2-re): reconstruction of [Wing Commander II: Vengeance of the Kilrathi (1991)](https://www.mobygames.com/game/823/wing-commander-ii-vengeance-of-the-kilrathi/), still in progress.

![wing commander cockpit combat screenshot](https://raw.githubusercontent.com/neuromancer/wc1-re/main/screenshots/cockpit-combat.png "350px") ![wing commander tiger's claw hangar screenshot](https://raw.githubusercontent.com/neuromancer/wc1-re/main/screenshots/tigers-claw-hangar.png "350px")

![wing commander ii cockpit navigation screenshot](https://raw.githubusercontent.com/neuromancer/wc2-re/main/screenshots/cockpit-navigation.png "350px") ![wing commander ii external flight screenshot](https://raw.githubusercontent.com/neuromancer/wc2-re/main/screenshots/external-flight.png "350px")
