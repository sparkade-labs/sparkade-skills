# Sparkade Skills

> AI agent skill files for building games on the [Sparkade](https://sparkade.games) platform.

These skills are designed to be fetched by AI agents during game development. They encode Sparkade's standards for game architecture, platform integration, mobile UX, and genre-specific mechanics.

## Usage

Point your AI agent at the index skill first:

```
https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/INDEX.md
```

The index will instruct the agent which skills to fetch based on the game being built.

## Structure

```
INDEX.md                          ← Start here
core/
├── phaser-setup.md               ← Phaser 3.90.0 config & scene structure
├── sparkade-integration.md       ← Platform message protocol & touch input
└── polish.md                     ← Juice, VFX, audio, camera effects
genres/
└── roguelite/
    ├── SKILL.md                  ← Genre overview & four pillars
    ├── run-structure.md          ← Room state machine, HUD, depth scaling
    ├── combat.md                 ← Player, enemies, collision, damage
    ├── upgrades.md               ← Upgrade system & selection UI
    └── procedural.md             ← Enemy composition, spawning, bosses
```

## License

© 2026 Sparkade. Licensed under [CC BY-NC-ND 4.0](./LICENSE.md).  
For use on the Sparkade platform only.
