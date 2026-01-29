# wp-gutenberg-claude-skill

Complete Gutenberg block development skill covering block.json, dynamic blocks, InnerBlocks, block variations, patterns, transforms, @wordpress/scripts, and server-side rendering.

## Installation

### Claude Code Plugin Marketplace

```bash
/plugin install https://github.com/xonack/wp-gutenberg-claude-skill
```

### Manual Installation

Copy the skill file to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/wp-gutenberg
cp skills/wp-gutenberg/SKILL.md ~/.claude/skills/wp-gutenberg/SKILL.md
```

## Usage

Once installed, the skill activates automatically when working on relevant WordPress tasks. You can also invoke it directly:

```
/wp-gutenberg
```

## Structure

```
wp-gutenberg-claude-skill/
├── README.md
├── LICENSE
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── wp-gutenberg/
        └── SKILL.md
```

## Contributing

PRs welcome. Please follow the [Agent Skills specification](https://agentskills.io/specification) for SKILL.md format.

## License

MIT
