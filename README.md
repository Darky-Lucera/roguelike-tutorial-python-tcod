# Roguelike Tutorial: Python & tcod

> A step-by-step guide to building a classic roguelike in Python, that explains **why** each decision was made, not just **what** to type.

[![Read the tutorial](https://img.shields.io/badge/read%20the-tutorial-7e57c2?style=for-the-badge)](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/)

[![Built with Material for MkDocs](https://img.shields.io/badge/built%20with-Material%20for%20MkDocs-526CFE?logo=materialformkdocs&logoColor=white)](https://squidfunk.github.io/mkdocs-material/)
[![Python 3.12+](https://img.shields.io/badge/python-3.12%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![tcod](https://img.shields.io/badge/tcod-21.2%2B-2b5797)](https://python-tcod.readthedocs.io/)
[![Tutorial: CC BY 4.0](https://img.shields.io/badge/tutorial-CC%20BY%204.0-lightgrey)](https://creativecommons.org/licenses/by/4.0/)
[![Code: MIT](https://img.shields.io/badge/code-MIT-green)](https://opensource.org/licenses/MIT)

**👉 Read it here: <https://darky-lucera.github.io/roguelike-tutorial-python-tcod/>**

---

## What makes this tutorial different

Most roguelike tutorials show you *what* to type. This one leads with the *why*: every chapter opens with the design rationale (events → actions → engine, components, separation of concerns) before a single line of code. The goal is that by the end you don't just have a working game, you understand the architecture behind it well enough to extend or rewrite it on your own.

It targets **modern Python** (3.12+), the Pythonic **`tcod`** library, and **`numpy`**, with [`uv`](https://docs.astral.sh/uv/) for project setup.

## What you will build

A complete, playable roguelike featuring:

- 🗺️ Procedurally generated dungeons
- ⚔️ Turn-based combat with A\* pathfinding
- 🎒 Items and inventory
- ✨ Spells and targeting
- 🪜 Multiple dungeon levels with XP and level-ups
- 🛡️ Equipment (weapons and armor)
- 💾 Save and load

## Contents

| Part | Chapter |
| --- | --- |
| 0 | [Introduction and Setup](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-0.html) |
| 1 | [Drawing the @ and Moving It](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-1.html) |
| 2 | [Entities, the Map, and the Engine](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-2.html) |
| 3 | [Generating a Dungeon](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-3.html) |
| 4 | [Field of View](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-4.html) |
| 5 | [Enemies and the Turn System](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-5.html) |
| 6 | [Combat](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-6.html) |
| 7 | [The User Interface](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-7.html) |
| 8 | [Items and Inventory](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-8-intro.html) (full or split into 8a / 8b) |
| 9 | [Spells and Targeting](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-9.html) |
| 10 | [Save and Load](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-10.html) |
| 11 | [Dungeon Levels and Experience](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-11.html) |
| 12 | [Procedural Difficulty](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-12.html) |
| 13 | [Equipment](https://darky-lucera.github.io/roguelike-tutorial-python-tcod/part-13.html) |

Plus six appendices on damage formulas, combat effects, consumable scaling, advanced dungeon generation, and designing enemies and equipment with personality.

## Tech stack

| Tool | Purpose |
| --- | --- |
| [Python 3.12+](https://www.python.org/) | Language |
| [tcod](https://python-tcod.readthedocs.io/) | Console rendering, FOV, pathfinding, input |
| [numpy](https://numpy.org/) | Tile arrays and fast map operations |
| [uv](https://docs.astral.sh/uv/) | Project and dependency management (game code) |
| [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) | Static site generator for this tutorial |

## Running the site locally

This repository contains the tutorial site (Markdown + MkDocs), not the game itself, the game is built step by step inside the chapters.

```bash
# 1. Clone
git clone https://github.com/Darky-Lucera/roguelike-tutorial-python-tcod.git
cd roguelike-tutorial-python-tcod

# 2. (Recommended) create a virtual environment
python -m venv .venv
source .venv/bin/activate      # Linux / macOS
.venv\Scripts\activate         # Windows PowerShell

# 3. Install the docs toolchain
pip install -r requirements.txt

# 4. Live preview at http://127.0.0.1:8000
mkdocs serve

# 5. Build the static site into site/
mkdocs build
```

## Deployment

Every push to `master` triggers a [GitHub Actions workflow](.github/workflows/pages.yml) that builds the site with MkDocs and publishes it to GitHub Pages. No manual steps required.

## Repository structure

```text
roguelike-tutorial-python-tcod/
├── v3/                   # tutorial source (Markdown, images, assets)
│   ├── index.md          # landing page
│   ├── part-0.md … part-13.md
│   ├── append-1.md … append-6.md
│   └── images/
├── mkdocs.yml            # MkDocs Material configuration and navigation
├── requirements.txt      # docs build dependencies
├── .github/workflows/    # GitHub Pages deployment
└── LICENSE
```

## Acknowledgements

This tutorial stands on the shoulders of the work that came before it. The text, code, and architecture here are original, but the chapter progression and many design decisions were informed by three earlier tutorials:

- [RogueBasin's "Complete Roguelike Tutorial, using python3+tcod"](https://www.roguebasin.com/index.php/Complete_Roguelike_Tutorial,_using_python3+tcod), built on the classic `libtcod` API.
- TStand90's original Roguelike Tutorial (2019), an early Python + `libtcod` rewrite.
- TStand90's "[Yet Another Roguelike Tutorial](https://rogueliketutorials.com/)" (v2, 2020), the modern `tcod` rewrite.

## License

Dual-licensed (see [LICENSE](LICENSE)):

- **Tutorial content** (prose, explanations, diagrams): [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
- **Code examples**: [MIT](https://opensource.org/licenses/MIT)

Copyright © 2026 Carlos Aragonés (caragones).
