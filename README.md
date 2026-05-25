# Sparkade Skills

> AI agent skill files for building games on the Sparkade platform.

These skills are designed to be fetched by AI agents during game development. They encode how to integrate the Sparkade platform in Phaser v4 games: the platform bridge API, required configuration, verified Phaser patterns, and UI correctness rules.

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
├── hidpi.md                      ← HiDPI plugin — mandatory, fixes blurry rendering
├── phaser-setup.md               ← Phaser v4 config & scene structure
├── patterns.md                   ← Copy-paste Phaser v4 API patterns
├── sparkade-integration.md       ← Platform bridge protocol, save system, IAP
└── ui.md                         ← In-game UI correctness patterns
```

## License

© 2026 Sparkade. Licensed under [CC BY-NC-ND 4.0](./LICENSE.md).  
For use on the Sparkade platform only.
