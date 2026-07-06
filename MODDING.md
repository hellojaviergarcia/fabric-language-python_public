# 📚 Fabric-Language-Python Modding

Welcome to the **comprehensive guide** on building Minecraft mods using `fabric-language-python`. This document covers **every feature, API, and pattern** the framework provides — from a simple "Hello World" item to JIT-compiled world generation, custom GUIs, networking, Mixins, and beyond.

---

## 📑 Table of Contents

**Foundations**
1. [Introduction & Architecture](#1-introduction--architecture)
2. [Getting Started](#2-getting-started)
3. [Project Structure](#3-project-structure)
4. [Assets, Textures & Translations](#4-assets-textures--translations)
5. [Entrypoints](#5-entrypoints)
6. [Java Interoperability (GraalPy)](#6-java-interoperability-graalpy)

**Content Registration**
7. [Items](#7-items)
8. [Blocks](#8-blocks)
9. [Block Entities](#9-block-entities)
10. [Entities (Custom Mobs)](#10-entities-custom-mobs)
11. [AI Goals](#11-ai-goals)
12. [Creative Tabs](#12-creative-tabs)

**Systems**
13. [Events & Callbacks](#13-events--callbacks)
14. [Commands](#14-commands)
15. [Networking (Client ↔ Server)](#15-networking-client--server)

**Client-Side**
16. [Client Rendering & Input](#16-client-rendering--input)
17. [GUI System](#17-gui-system)
18. [3D Entity Models](#18-3d-entity-models)

**Advanced Content**
19. [Status Effects](#19-status-effects)
20. [Custom Particles](#20-custom-particles)
21. [Sounds](#21-sounds)
22. [World Generation](#22-world-generation)
23. [Data Generation (Recipes, Loot, Enchantments)](#23-data-generation-recipes-loot-enchantments)
24. [Persistent Data Storage](#24-persistent-data-storage)

**Expert Topics**
25. [Advanced Mixin Support (Core Modding)](#25-advanced-mixin-support-core-modding)
26. [Performance & Optimization](#26-performance--optimization)
27. [Async Operations](#27-async-operations)
28. [Third-Party Mod Interop](#28-third-party-mod-interop)
29. [Native Python (NumPy, TensorFlow, etc.)](#29-native-python-numpy-tensorflow-etc)
30. [Dynamic Class Generation (ByteBuddy)](#30-dynamic-class-generation-bytebuddy)
31. [Multi-File Mod Organization](#31-multi-file-mod-organization)

**Reference**
32. [In-Game Developer Tools](#32-in-game-developer-tools)
33. [Testing & Debugging](#33-testing--debugging)
34. [Built-in Utilities](#34-built-in-utilities)
35. [Critical Best Practices & 26.2 Quirks](#35-critical-best-practices--262-quirks)
36. [Complete Example Mod](#36-complete-example-mod)
37. [API Quick Reference](#37-api-quick-reference)

---

## 1. Introduction & Architecture

`fabric-language-python` is a **Language Adapter** for Fabric that allows you to write Minecraft mods entirely in **Python 3**.

### What Makes It Special

| Feature | Description |
|---|---|
| **Dual API** | Use the high-level `minecraft_py` decorators OR the raw Minecraft Java API directly |
| **Hot Reload** | Modify Python code and run `/py reload` — no restart needed |
| **Full Mixin Support** | Python `@Mixin` decorators compile to real Java Mixin classes at build time |
| **JIT Compilation** | World generation math can be compiled directly to JVM bytecode |
| **Native Python** | Run NumPy, TensorFlow, etc. via the native CPython sidecar process |
| **Zero Boilerplate** | Create a mod with a single Python file — no Gradle needed for simple mods |
| **In-Game REPL** | Execute arbitrary Python in a live server with `/py eval` |

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Your Python Mod                    │
│   (@item, @block, @entity, events, commands, etc.)  │
├─────────────────────────────────────────────────────┤
│              minecraft_py (Python API)               │
│  items · blocks · entities · events · gui · network  │
│  worldgen · effects · models · ai · mixins · etc.    │
├─────────────────────────────────────────────────────┤
│           Java Bridge Layer (37 classes)              │
│                                                       │
│                                                       │
├─────────────────────────────────────────────────────┤
│        GraalPy 25.0.2 (JVM Python Runtime)           │
├─────────────────────────────────────────────────────┤
│         Fabric Loader + Minecraft 26.2               │
└─────────────────────────────────────────────────────┘
```

### Two Ways to Write Mods

**1. High-Level API (`minecraft_py`)** — Pythonic decorators and helper functions:
```python
from minecraft_py import item, events

@item("mymod", "ruby")
def ruby(): pass

@events.on_player_join
def welcome(handler, sender, server):
    print(f"Welcome, {sender.getName()}!")
```

**2. Raw Java API (Pure Bridge)** — Direct access to Minecraft/Fabric classes:
```python
from net.fabricmc.api import ModInitializer
from net.minecraft.item import Item
from net.minecraft.registry import Registry, Registries

class MyMod(ModInitializer):
    def onInitialize(self):
        Registry.register(Registries.ITEM, "mymod:ruby", Item(Item.Settings()))
```

Both approaches can be mixed freely within the same mod.

---

## 2. Getting Started

### Prerequisites

- **Java 25** (required for Minecraft 26.2)
- **Python 3.10+** (only needed for CLI tools and mixin compilation)
- **Minecraft 26.2** with Fabric Loader ≥ 0.16.9
- The `fabric-language-python` adapter JAR in your `mods/` folder

### Installation

1. Install [Fabric Loader](https://fabricmc.net/) for Minecraft 26.2
2. Drop the `fabric-language-python-1.0.0+python.25.0.2.jar` into your `.minecraft/mods/` folder
3. Create your mod (see below)

### Extracting the CLI Tool & Python Stubs

The `minecraft_py` Python package is bundled **inside** the framework JAR. To use the CLI tool and get IDE autocomplete, extract it first:

```bash
# 1. Extract the minecraft_py package from the JAR
#    (a JAR is just a ZIP file)
mkdir minecraft_py_sdk
cd minecraft_py_sdk

# On Linux/macOS:
unzip /path/to/fabric-language-python-1.0.0+python.25.0.2.jar "python_stdlib/minecraft_py/*"

# On Windows (PowerShell):
Expand-Archive -Path "path\to\fabric-language-python-1.0.0+python.25.0.2.jar" -DestinationPath . -Force

# 2. Add the extracted path to your PYTHONPATH
#    Linux/macOS:
export PYTHONPATH="$PWD/python_stdlib:$PYTHONPATH"
#    Windows:
set PYTHONPATH=%CD%\python_stdlib;%PYTHONPATH%

# 3. Now you can use the CLI tool:
python -m minecraft_py init my-first-mod --example

# 4. Generate IDE stubs for autocomplete:
python -m minecraft_py stubs
```

> [!TIP]
> **IDE Autocomplete Setup:** After running `python -m minecraft_py stubs`, copy the generated `.pyi` files into your mod project. In VS Code, add the stubs directory to `python.analysis.extraPaths` in your settings. In PyCharm, mark it as a "Sources Root".

> [!TIP]
> **Alternative — no extraction needed:** If you don't want to extract, you can simply create the files manually following the templates in the next section. The `minecraft_py` package is automatically available at runtime when your mod loads inside Minecraft.

### Quickstart: Folder Mod (No Gradle!)

The fastest way to start — create a mod folder directly in `.minecraft/mods/`:

```bash
# Using the CLI tool (after extracting, see above)
python -m minecraft_py init my-first-mod --example

# Or create the structure entirely by hand (see templates below)
```

This generates:
```
my-first-mod/
├── fabric.mod.json
├── pack.mcmeta
├── assets/my-first-mod/
├── data/my-first-mod/
└── main.py          ← Your mod code goes here
```

**`fabric.mod.json`:**
```json
{
  "schemaVersion": 1,
  "id": "my-first-mod",
  "version": "1.0.0",
  "name": "My First Mod",
  "description": "My first Python Minecraft mod!",
  "authors": ["YourName"],
  "environment": "*",
  "entrypoints": {
    "main": [{
      "adapter": "python",
      "value": "main.py"
    }]
  },
  "depends": {
    "fabric-language-python": ">=1.0.0"
  }
}
```

**`pack.mcmeta`:**
```json
{
  "pack": {
    "pack_format": 48,
    "description": "My First Mod resources"
  }
}
```

> [!IMPORTANT]
> **`pack_format: 48`** is the correct value for Minecraft 26.2. Using a wrong value may cause resource loading warnings or failures.

**`main.py`:**
```python
from net.fabricmc.api import ModInitializer
from org.slf4j import LoggerFactory

logger = LoggerFactory.getLogger("my-first-mod")

class MyMod(ModInitializer):
    def onInitialize(self):
        logger.info("Hello from Python!")

mod_instance = MyMod()
```

Drop the folder into `.minecraft/mods/` and launch. That's it!

### Quickstart: Gradle Mod (Advanced, with Mixins)

For mods that need Python Mixins or Gradle tooling:

```bash
# Using the scaffolding tool (after extracting the SDK)
python tools/create-fabric-python.py
```

Or set up manually — see [Project Structure](#3-project-structure).

---

## 3. Project Structure

> [!IMPORTANT]
> **Mod ID Rules:** Your mod ID (`"id"` in `fabric.mod.json`) must be **lowercase**, use only `a-z`, `0-9`, `-`, and `_`. No spaces, no uppercase. This ID is used everywhere: folder names, registry keys, translation keys, asset paths.

### Folder Mod (Simple)
```
my-mod/                          ← Drop directly into .minecraft/mods/
├── fabric.mod.json              ← Mod metadata + entrypoints
├── pack.mcmeta                  ← Resource pack metadata (pack_format: 46)
├── main.py                      ← Your Python mod code
├── assets/<mod_id>/
│   ├── textures/
│   │   ├── item/                ← Item textures (.png)
│   │   ├── block/               ← Block textures (.png)
│   │   └── entity/              ← Entity textures (.png)
│   ├── models/
│   │   ├── item/                ← Item model JSONs
│   │   └── block/               ← Block model JSONs
│   ├── blockstates/             ← Block state definitions
│   ├── sounds/                  ← Sound files (.ogg)
│   └── lang/
│       └── en_us.json           ← Translations
└── data/<mod_id>/
    ├── recipe/                  ← Crafting recipes
    ├── loot_table/              ← Loot tables
    ├── advancement/             ← Advancements
    └── tags/                    ← Tag definitions
```

### Gradle Mod (Advanced)
```
my-mod/
├── build.gradle
├── gradle.properties
├── settings.gradle
├── src/main/
│   ├── python/<mod_id>/
│   │   ├── __init__.py
│   │   └── main.py             ← Your Python mod code
│   └── resources/
│       ├── fabric.mod.json
│       ├── assets/<mod_id>/     ← (same asset structure as above)
│       └── data/<mod_id>/       ← (same data structure as above)
```

### Gradle `build.gradle` (for mod users)
```groovy
plugins {
    id 'fabric-loom' version '1.17.local' // Or your preferred version
}

repositories {
    flatDir {
        dirs 'libs'
    }
}

// Apply the local Python Mixin compiler (only if using @Mixin)
apply from: "python-mixins.gradle"

dependencies {
    minecraft "com.mojang:minecraft:26.2"
    implementation "net.fabricmc:fabric-loader:0.19.3"
    compileOnly "net.fabricmc.fabric-api:fabric-api:0.154.0+26.2"

    // The Python Language Adapter (from local libs folder)
    implementation "net.fabricmc:fabric-language-python:1.0.0+python.25.0.2"
}
```

---

## 4. Assets, Textures & Translations

Every item, block, and entity needs visual assets and translation strings to appear correctly in-game.

### 4.1 Texture Conventions

| Asset Type | Path | Size | Format |
|---|---|---|---|
| Item texture | `assets/<mod_id>/textures/item/<name>.png` | **16×16 px** (standard) | PNG, supports transparency |
| Block texture | `assets/<mod_id>/textures/block/<name>.png` | **16×16 px** (standard) | PNG, supports transparency |
| Entity texture | `assets/<mod_id>/textures/entity/<name>.png` | **64×64 px** (typical) | PNG, standard skin layout |
| GUI texture | `assets/<mod_id>/textures/gui/<name>.png` | Varies | PNG |
| Particle texture | `assets/<mod_id>/textures/particle/<name>.png` | **8×8 px** (typical) | PNG, supports transparency |

> [!TIP]
> **Recommended Tools:** Use [Aseprite](https://www.aseprite.org/) or [Piskel](https://www.piskelapp.com/) for pixel art textures. Use [BlockBench](https://www.blockbench.net/) for 3D entity models and their UV-mapped textures.

Textures can be **higher resolution** (32×32, 64×64, etc.) for HD texture packs, but must always be **powers of 2** and maintain the standard aspect ratio.

### 4.2 Model Files

**Item model** (`assets/<mod_id>/models/item/<name>.json`):
```json
{
  "parent": "minecraft:item/generated",
  "textures": {
    "layer0": "<mod_id>:item/<name>"
  }
}
```

**Handheld item model** (for tools/swords — renders tilted in hand):
```json
{
  "parent": "minecraft:item/handheld",
  "textures": {
    "layer0": "<mod_id>:item/<name>"
  }
}
```

**Block model** (`assets/<mod_id>/models/block/<name>.json`):
```json
{
  "parent": "minecraft:block/cube_all",
  "textures": {
    "all": "<mod_id>:block/<name>"
  }
}
```

**Block item model** (`assets/<mod_id>/models/item/<name>.json`) — inherits from block:
```json
{
  "parent": "<mod_id>:block/<name>"
}
```

**Blockstate definition** (`assets/<mod_id>/blockstates/<name>.json`):
```json
{
  "variants": {
    "": { "model": "<mod_id>:block/<name>" }
  }
}
```

### 4.3 Translations (Language Files)

Create `assets/<mod_id>/lang/en_us.json` with human-readable names for your content:

```json
{
  "item.<mod_id>.<item_name>": "Ruby",
  "item.<mod_id>.ruby_sword": "Ruby Sword",
  "item.<mod_id>.golden_berry": "Golden Berry",

  "block.<mod_id>.<block_name>": "Ruby Block",
  "block.<mod_id>.altar": "Altar",

  "entity.<mod_id>.<entity_name>": "Guardian",
  "entity.<mod_id>.zombie_king": "Zombie King",

  "itemGroup.<mod_id>.<tab_name>": "My Mod Items",

  "effect.<mod_id>.<effect_name>": "Ruby Weakness",

  "enchantment.<mod_id>.<enchant_name>": "Inferno",

  "subtitles.<mod_id>.<sound_name>": "Guardian roars"
}
```

**Translation key format rules:**

| Content Type | Key Format | Example |
|---|---|---|
| Items | `item.<mod_id>.<item_id>` | `item.mymod.ruby` → "Ruby" |
| Blocks | `block.<mod_id>.<block_id>` | `block.mymod.ruby_block` → "Ruby Block" |
| Entities | `entity.<mod_id>.<entity_id>` | `entity.mymod.guardian` → "Guardian" |
| Creative Tabs | `itemGroup.<mod_id>.<tab_id>` | `itemGroup.mymod.main` → "My Mod" |
| Status Effects | `effect.<mod_id>.<effect_id>` | `effect.mymod.wither_touch` → "Wither Touch" |
| Enchantments | `enchantment.<mod_id>.<enchant_id>` | `enchantment.mymod.inferno` → "Inferno" |
| Sound Subtitles | `subtitles.<mod_id>.<sound_id>` | `subtitles.mymod.boss_roar` → "Boss roars" |
| Container GUIs | `container.<mod_id>.<name>` | `container.mymod.grinder` → "Grinder" |

> [!WARNING]
> **Without translations,** items/blocks will show raw keys like `item.mymod.ruby` in-game. Always create at minimum an `en_us.json` file.

For additional languages, create more files: `es_es.json` (Spanish), `fr_fr.json` (French), `de_de.json` (German), `pt_br.json` (Brazilian Portuguese), `zh_cn.json` (Simplified Chinese), `ja_jp.json` (Japanese), etc.

### 4.4 Complete Asset Checklist

When adding a new **item** (`mymod:ruby`):
- [ ] `assets/mymod/textures/item/ruby.png` — 16×16 texture
- [ ] `assets/mymod/models/item/ruby.json` — Item model
- [ ] `assets/mymod/lang/en_us.json` — Add `"item.mymod.ruby": "Ruby"`

When adding a new **block** (`mymod:ruby_block`):
- [ ] `assets/mymod/textures/block/ruby_block.png` — 16×16 texture
- [ ] `assets/mymod/models/block/ruby_block.json` — Block model
- [ ] `assets/mymod/models/item/ruby_block.json` — Block item model (parent of block model)
- [ ] `assets/mymod/blockstates/ruby_block.json` — Blockstate definition
- [ ] `assets/mymod/lang/en_us.json` — Add `"block.mymod.ruby_block": "Ruby Block"`

When adding a new **entity** (`mymod:guardian`):
- [ ] `assets/mymod/textures/entity/guardian.png` — 64×64 entity texture
- [ ] `assets/mymod/lang/en_us.json` — Add `"entity.mymod.guardian": "Guardian"`

When adding a new **sound** (`mymod:guardian_roar`):
- [ ] `assets/mymod/sounds/guardian_roar.ogg` — OGG Vorbis audio file
- [ ] `assets/mymod/lang/en_us.json` — Add `"subtitles.mymod.guardian_roar": "Guardian roars"`

---

## 5. Entrypoints

Define your mod's entrypoints in `fabric.mod.json`. The adapter supports four resolution patterns:

### Pattern 1: Instance Variable (Recommended)
```json
{
  "entrypoints": {
    "main": [{ "adapter": "python", "value": "mymod.main::mod_instance" }]
  }
}
```
```python
# mymod/main.py
class MyMod(ModInitializer):
    def onInitialize(self):
        pass

mod_instance = MyMod()  # Adapter uses this object directly
```

### Pattern 2: Class Reference (Auto-instantiated)
```json
{ "value": "mymod.main::MyMod" }
```
The adapter calls `MyMod()` automatically.

### Pattern 3: Function Reference
```json
{ "value": "mymod.main::on_initialize" }
```
```python
def on_initialize():
    print("Hello!")
```
The adapter wraps the function as the expected Java functional interface.

### Pattern 4: Auto-Discovery
```json
{ "value": "main.py" }
```
The adapter scans the module for a variable named `mod_instance`, or any object implementing the required interface.

### Client Entrypoint
```json
{
  "entrypoints": {
    "main":   [{ "adapter": "python", "value": "mymod.main::mod_instance" }],
    "client": [{ "adapter": "python", "value": "mymod.client::client_instance" }]
  }
}
```
```python
# mymod/client.py
from net.fabricmc.api import ClientModInitializer

class MyClient(ClientModInitializer):
    def onInitializeClient(self):
        print("Client side ready!")

client_instance = MyClient()
```

---

## 6. Java Interoperability (GraalPy)

The adapter installs a custom import hook that lets you use standard Python `import` syntax for any Java class. Mojang names are automatically resolved to the correct runtime mappings.

### 5.1 Importing Java Classes
```python
# Standard Minecraft classes
from net.minecraft.item import Item
from net.minecraft.block import Block, AbstractBlock
from net.minecraft.entity.player import PlayerEntity
from net.minecraft.registry import Registry, Registries
from net.minecraft.resources import Identifier   # 26.2 name for ResourceLocation

# Fabric API classes
from net.fabricmc.api import ModInitializer
from net.fabricmc.fabric.api.event.lifecycle.v1 import ServerTickEvents

# Any Java class
from java.util import ArrayList, HashMap
from org.slf4j import LoggerFactory
```

### 5.2 Creating Java Objects
```python
from net.minecraft.resources import Identifier

my_id = Identifier.of("mymod", "ruby")     # Static factory method
my_id2 = Identifier.parse("mymod:ruby")    # Alternative
```

### 5.3 Implementing Java Interfaces

**Single-method interfaces** (functional interfaces) — use lambdas or functions:
```python
from net.fabricmc.fabric.api.event.lifecycle.v1 import ServerTickEvents

# Lambda works automatically for functional interfaces
ServerTickEvents.END_SERVER_TICK.register(lambda server: print("Tick!"))

# Named function also works
def on_tick(server):
    pass
ServerTickEvents.END_SERVER_TICK.register(on_tick)
```

**Multi-method interfaces** — extend the Java interface:
```python
import java
SomeInterface = java.type("com.example.SomeInterface")

class MyImpl(SomeInterface):
    def methodA(self):
        return "hello"
    def methodB(self, arg):
        pass
```

### 5.4 Java Arrays
```python
import java
StringArray = java.type("java.lang.String[]")
arr = StringArray(3)
arr[0] = "Hello"
arr[1] = "World"
```

### 5.5 Accessing Java `.class` Literals

> [!WARNING]
> Python's `class` is a reserved keyword. You **cannot** write `MyClass.class`.

```python
from net.minecraft.entity.player import PlayerEntity

# ✅ Correct — use getattr
player_class = getattr(PlayerEntity, "class")

# ✅ Also correct — use the built-in java_class() helper
player_class = java_class(PlayerEntity)

# ❌ WRONG — SyntaxError!
# player_class = PlayerEntity.class
```

### 5.6 Using `java.type()` for Dynamic Resolution
```python
import java

# Load a class by fully-qualified name
Vec3 = java.type("net.minecraft.world.phys.Vec3")
my_vec = Vec3(1.0, 2.0, 3.0)
```

---

## 7. Items

### 6.1 Simple Item (High-Level API)

The simplest way to register an item:

```python
from minecraft_py import item

@item("mymod", "ruby")
def ruby():
    pass
```

This single decorator registers the item in the game registry. You'll need to provide the texture at `assets/mymod/textures/item/ruby.png` and the model at `assets/mymod/models/item/ruby.json`.

**Standard item model JSON** (`assets/mymod/models/item/ruby.json`):
```json
{
  "parent": "minecraft:item/generated",
  "textures": {
    "layer0": "mymod:item/ruby"
  }
}
```

### 6.2 Items with Callbacks
```python
from minecraft_py.items import create_item

def on_right_click(level, player, hand):
    print(f"{player.getName()} used the magic wand!")

wand = create_item("mymod", "magic_wand", max_count=1, on_use=on_right_click)
```

### 6.3 Food Items
```python
from minecraft_py.items import create_food

# nutrition = hunger points restored, saturation = saturation modifier
golden_apple = create_food("mymod", "golden_berry",
    nutrition=6,
    saturation=1.2,
    on_use=lambda level, player, hand: print("Yummy!")
)
```

### 6.4 Tool Items (Swords, Pickaxes, Axes, Shovels, Hoes)
```python
from minecraft_py.items import create_sword, create_pickaxe, create_axe, create_shovel, create_hoe

# Materials: "WOOD", "STONE", "IRON", "GOLD", "DIAMOND", "NETHERITE"
ruby_sword   = create_sword("mymod", "ruby_sword", material="DIAMOND", damage=8, speed=-2.4)
ruby_pickaxe = create_pickaxe("mymod", "ruby_pickaxe", material="DIAMOND", damage=2, speed=-2.8)
ruby_axe     = create_axe("mymod", "ruby_axe", material="DIAMOND", damage=7, speed=-3.0)
ruby_shovel  = create_shovel("mymod", "ruby_shovel", material="DIAMOND", damage=2.5, speed=-3.0)
ruby_hoe     = create_hoe("mymod", "ruby_hoe", material="DIAMOND", damage=-3, speed=-1.0)

# Tools can also have on_use callbacks
def on_sword_use(level, player, hand):
    print("Sword ability activated!")

enchanted_sword = create_sword("mymod", "enchanted_sword",
    material="NETHERITE", damage=12, speed=-2.0,
    on_use=on_sword_use
)
```

### 6.5 Custom Item Data (Data Components)

Store custom string data on individual item stacks:

```python
from minecraft_py import MinecraftBridge

# Set custom data on an ItemStack
MinecraftBridge.setPythonData(item_stack, '{"charges": 5, "owner": "Steve"}')

# Read custom data from an ItemStack
data = MinecraftBridge.getPythonData(item_stack)  # Returns JSON string
```

### 6.6 Items via Raw Java API

For full control, use the Minecraft API directly:

```python
from net.minecraft.item import Item
from net.minecraft.registry import Registry, Registries
from net.minecraft.resources import Identifier, ResourceKey

item_id = Identifier.of("mymod", "ruby")
item_key = ResourceKey.create(Registries.ITEM.key(), item_id)

# 26.2 REQUIRES setId() before constructing the Item
props = Item.Settings().setId(item_key)
ruby = Item(props)

Registry.register(Registries.ITEM, item_id, ruby)
```

> [!WARNING]
> **Minecraft 26.2 Strict Instantiation:** You **must** call `.setId(key)` on `Item.Settings()` before passing it to the `Item` constructor. Omitting this causes `NullPointerException: Block id not set`.

---

## 8. Blocks

### 7.1 Simple Block (High-Level API)

```python
from minecraft_py import block

@block("mymod", "ruby_block", hardness=3.0)
def ruby_block():
    pass
```

This registers both the block and its corresponding `BlockItem` (so players can pick it up).

### 7.2 Blocks with Properties and Callbacks
```python
from minecraft_py.blocks import create_block

def on_right_click(level, player, pos):
    print(f"Block activated at {pos}!")

def on_step(level, entity, pos):
    print(f"Something stepped on the block!")

magic_block = create_block("mymod", "magic_block",
    hardness=2.0,
    on_use=on_right_click,
    on_step=on_step
)
```

### 7.3 Blocks via Raw Java API
```python
from net.minecraft.block import Block, AbstractBlock
from net.minecraft.item import BlockItem, Item
from net.minecraft.registry import Registry, Registries
from net.minecraft.resources import Identifier, ResourceKey

block_id = Identifier.of("mymod", "ruby_block")
block_key = ResourceKey.create(Registries.BLOCK.key(), block_id)

block_props = AbstractBlock.Settings.create().setId(block_key)
block_props.strength(3.0, 6.0)        # hardness, resistance
block_props.requiresCorrectToolForDrops()
block_props.lightLevel(lambda state: 7)  # Luminance 0-15

ruby_block = Block(block_props)
Registry.register(Registries.BLOCK, block_id, ruby_block)

# Also register the BlockItem
item_key = ResourceKey.create(Registries.ITEM.key(), block_id)
item_props = Item.Settings().setId(item_key)
ruby_block_item = BlockItem(ruby_block, item_props)
Registry.register(Registries.ITEM, block_id, ruby_block_item)
```

**Required asset files:**

`assets/mymod/blockstates/ruby_block.json`:
```json
{
  "variants": {
    "": { "model": "mymod:block/ruby_block" }
  }
}
```

`assets/mymod/models/block/ruby_block.json`:
```json
{
  "parent": "minecraft:block/cube_all",
  "textures": {
    "all": "mymod:block/ruby_block"
  }
}
```

`assets/mymod/models/item/ruby_block.json`:
```json
{
  "parent": "mymod:block/ruby_block"
}
```

### 7.4 Machine Blocks (with Block Entity)

For blocks that store data, have inventories, or tick every frame:

```python
from minecraft_py import MinecraftBridge

def on_machine_use(level, player, pos):
    print("Machine activated! Opening GUI...")

machine = MinecraftBridge.createMachineBlock("mymod", "grinder", 4.0, on_machine_use)
MinecraftBridge.registerBlock("mymod", "grinder", machine)
```

Machine blocks automatically support Block Entities — see [Block Entities](#8-block-entities).

---

## 9. Block Entities

Block entities add persistent data, inventory, ticking behavior, and GUI support to blocks.

### 8.1 Simple Block Entity (High-Level API)

```python
from minecraft_py.block_entities import SimpleBlockEntity, register_block_entity

class CrystalEntity(SimpleBlockEntity):
    """A block entity that stores custom data, auto-serialized to NBT."""
    
    def __init__(self):
        super().__init__()
        self.data = {"energy": 0, "max_energy": 1000}  # Auto-saved/loaded
    
    def tick(self, level, pos, state):
        """Called every game tick (20 times per second)."""
        self.data["energy"] = min(self.data["energy"] + 1, self.data["max_energy"])
        if self.data["energy"] >= 1000:
            print("Crystal fully charged!")

# Register the block entity type, binding it to your block(s)
crystal_be_type = register_block_entity("mymod", "crystal_entity", CrystalEntity, crystal_block)
```

### 8.2 Inventory Block Entity

```python
from minecraft_py.block_entities import InventoryBlockEntity, register_block_entity
from minecraft_py import MinecraftBridge

class FurnaceEntity(InventoryBlockEntity):
    """Block entity with item slot management."""
    inventory_size = 3  # Number of slots
    name = "Python Furnace"  # Display name for the GUI
    
    def __init__(self):
        super().__init__()
        self.data = {"cook_time": 0}
    
    def tick(self, level, pos, state):
        input_item = self.get_item(0)   # Slot 0: input
        fuel_item  = self.get_item(1)   # Slot 1: fuel
        
        if input_item and fuel_item:
            self.data["cook_time"] += 1
            if self.data["cook_time"] >= 200:
                self.set_item(2, "minecraft:iron_ingot", 1)  # Slot 2: output
                self.data["cook_time"] = 0

furnace_be_type = register_block_entity("mymod", "furnace", FurnaceEntity, furnace_block)
```

### 8.3 Block Entity via Raw Java API

```python
from net.minecraft.block.entity import BlockEntity, BlockEntityType
from net.minecraft.registry import Registry, Registries
from net.minecraft.resources import Identifier

class MyBlockEntity(BlockEntity):
    def __init__(self, pos, state):
        super().__init__(MY_BE_TYPE, pos, state)
        self.custom_data = 0

    def saveAdditional(self, output, registries):
        """26.2 uses ValueOutput instead of CompoundTag"""
        super().saveAdditional(output, registries)
        output.putInt("CustomData", self.custom_data)

    def loadAdditional(self, input, registries):
        """26.2 uses ValueInput instead of CompoundTag"""
        super().loadAdditional(input, registries)
        self.custom_data = input.getInt("CustomData")

MY_BE_TYPE = Registry.register(
    Registries.BLOCK_ENTITY_TYPE,
    Identifier.of("mymod", "my_be"),
    BlockEntityType.Builder.create(
        lambda pos, state: MyBlockEntity(pos, state),
        my_block
    ).build()
)
```

> [!IMPORTANT]
> **26.2 API Change:** `saveAdditional` and `loadAdditional` use `ValueOutput` and `ValueInput` instead of the traditional `CompoundTag`. The `minecraft_py.block_entities.SimpleBlockEntity` handles this automatically by serializing your `self.data` dict as JSON.

---

## 10. Entities (Custom Mobs)

### 9.1 Entity via High-Level API

```python
from minecraft_py import entity

@entity("mymod", "fire_golem", health=80.0, speed=0.25, damage=10.0)
class FireGolem:
    def register_goals(self, entity):
        """Called when the entity spawns to set up AI."""
        from minecraft_py.ai import (
            add_float_goal, add_melee_attack_goal,
            add_wander_goal, add_target_player_goal
        )
        add_float_goal(entity, priority=0)
        add_melee_attack_goal(entity, priority=1, speed_modifier=1.3)
        add_wander_goal(entity, priority=3, speed_modifier=0.8)
        add_target_player_goal(entity, priority=1)
    
    def tick(self, entity):
        """Called every game tick."""
        if entity.getHealth() < 40:
            entity.setGlowing(True)
    
    def get_texture(self):
        """Return the texture identifier."""
        return "mymod:textures/entity/fire_golem.png"
```

### 9.2 Entity via `register_entity()`

```python
from minecraft_py.entities import BaseEntity, register_entity
from minecraft_py.ai import (
    add_float_goal, add_melee_attack_goal, add_wander_goal,
    add_target_player_goal, create_custom_goal
)

class ZombieKing(BaseEntity):
    def register_goals(self, entity):
        add_float_goal(entity, priority=0)
        add_melee_attack_goal(entity, priority=1, speed_modifier=1.2)
        add_wander_goal(entity, priority=5, speed_modifier=0.8)
        add_target_player_goal(entity, priority=1, must_see=True)
    
    def tick(self, entity):
        # Custom tick logic
        hp_percent = entity.getHealth() / entity.getMaxHealth()
        if hp_percent < 0.3:
            # Enrage phase!
            entity.setAttributeBaseValue("attack_damage", 20.0)
            entity.setAttributeBaseValue("movement_speed", 0.4)
    
    def get_texture(self):
        return "mymod:textures/entity/zombie_king.png"
    
    def get_model(self):
        """Return a custom model item ID for 3D display."""
        return "mymod:zombie_king_model"

# Register with custom attributes
zombie_king_type = register_entity("mymod", "zombie_king", ZombieKing,
    health=100.0,
    speed=0.3,
    damage=12.0
)
```

### 9.3 Entity Lifecycle Callbacks

Your Python entity class can implement these methods:

| Method | When Called | Parameters |
|---|---|---|
| `register_goals(entity)` | When entity spawns | The Java `PythonEntity` instance |
| `tick(entity)` | Every game tick (20/sec) | The Java `PythonEntity` instance |
| `get_texture()` | When rendering | None (return texture path string) |
| `get_model()` | When setting up 3D model | None (return item ID string) |
| `setup_animation(model, walk_pos, walk_speed, age, y_rot, x_rot)` | Every render frame | Animation parameters |
| `_on_destroyed()` | When entity is removed | None |

### 9.4 PythonEntity Bridge Methods

The Java `PythonEntity` object passed to your callbacks exposes these bridge methods (safe from obfuscation):

```python
def tick(self, entity):
    # AI Goals (bypasses obfuscated goalSelector)
    entity.addPythonGoal(priority, goal)
    entity.addPythonTargetGoal(priority, goal)
    
    # Attributes
    entity.setAttributeBaseValue("max_health", 100.0)
    entity.setAttributeBaseValue("movement_speed", 0.3)
    entity.setAttributeBaseValue("attack_damage", 12.0)
    entity.setAttributeBaseValue("attack_speed", 4.0)
    entity.setAttributeBaseValue("armor", 10.0)
    entity.setAttributeBaseValue("armor_toughness", 2.0)
    entity.setAttributeBaseValue("knockback_resistance", 0.5)
    entity.setAttributeBaseValue("follow_range", 48.0)
    
    # NBT Data (persistent across saves)
    entity.setPythonNbtString("phase", "enraged")
    entity.setPythonNbtInt("kills", 5)
    entity.setPythonNbtDouble("power_level", 9000.1)
    entity.setPythonNbtBoolean("is_boss", True)
    
    phase = entity.getPythonNbtString("phase")
    kills = entity.getPythonNbtInt("kills")
    
    # Rendering
    entity.setTextureId("mymod:textures/entity/boss_enraged.png")
    entity.setEntityScale(2.0)
    entity.attach3DModel("mymod:boss_model")  # ItemDisplay passenger
    
    # Utility
    entity.sendMessageToNearby("The boss roars!", 50.0)
    entity.spawnParticlesAtSelf("minecraft:flame", 20)
    entity.playSoundAtSelf("minecraft:entity.ender_dragon.growl", 1.0, 0.5)
    
    # World access
    level = entity.getPythonLevel()
    pos = entity.getPythonBlockPos()
```

### 9.5 3D Entity Models (ItemDisplay)

Attach a 3D model to your entity using an ItemDisplay passenger:

```python
class BossEntity(BaseEntity):
    def register_goals(self, entity):
        # Attach a custom 3D model item as a visual
        entity.attach3DModel("mymod:boss_model_item")
        # The entity becomes invisible, and the ItemDisplay rides on top
```

> [!IMPORTANT]
> **UUID Duplication Fix:** The `attach3DModel()` method defers spawning the passenger entity until the first `tick()`. This prevents the `Unable to summon entity due to duplicate UUIDs` crash that occurs when spawning passengers during construction.

---

## 11. AI Goals

### 10.1 Built-in Goals
```python
from minecraft_py.ai import (
    add_float_goal,            # Swim in water
    add_melee_attack_goal,     # Attack target in melee
    add_wander_goal,           # Wander randomly
    add_target_player_goal,    # Target nearest player
)

def register_goals(self, entity):
    add_float_goal(entity, priority=0)
    add_melee_attack_goal(entity, priority=2, speed_modifier=1.2, following_target_even_if_not_seen=False)
    add_wander_goal(entity, priority=5, speed_modifier=1.0)
    add_target_player_goal(entity, priority=1, must_see=True)
```

**Priority** determines execution order — **lower numbers = higher priority**.

### 10.2 Custom AI Goals
```python
from minecraft_py.ai import create_custom_goal

class TeleportGoal:
    """Teleport to a random position when health is low."""
    
    def can_use(self):
        """Return True when this goal should activate."""
        return self.entity.getHealth() < 20
    
    def start(self):
        """Called when the goal activates."""
        import random
        x = self.entity.getX() + random.randint(-10, 10)
        z = self.entity.getZ() + random.randint(-10, 10)
        y = self.entity.getY()
        self.entity.teleportTo(x, y, z)
    
    def stop(self):
        """Called when the goal deactivates."""
        pass
    
    def tick(self):
        """Called every tick while active."""
        pass
    
    def can_continue(self):
        """Return True to keep the goal active."""
        return False  # One-shot teleport

# In register_goals:
def register_goals(self, entity):
    teleport_goal = create_custom_goal(TeleportGoal)
    entity.addPythonGoal(3, teleport_goal)
```

### 10.3 Using Native Minecraft Goals
```python
import java
from net.minecraft.world.entity.ai.goal import FloatGoal, RandomStrollGoal, LookAtPlayerGoal
from net.minecraft.world.entity.player import Player

def register_goals(self, entity):
    entity.addPythonGoal(0, FloatGoal(entity))
    entity.addPythonGoal(5, RandomStrollGoal(entity, 0.8))
    entity.addPythonGoal(6, LookAtPlayerGoal(entity, java_class(Player), 8.0))
```

---

## 12. Creative Tabs

Add your items to existing creative tabs or create custom ones.

### 12.1 Adding Items to Existing Tabs (Raw API)
```python
from net.fabricmc.fabric.api.itemgroup.v1 import ItemGroupEvents
from net.minecraft.world.item import CreativeModeTabs

def add_to_ingredients(entries):
    entries.accept(ruby)         # Your registered Item object
    entries.accept(ruby_block)   # Your registered BlockItem

# Add to the "Ingredients" tab
ItemGroupEvents.modifyEntriesEvent(CreativeModeTabs.INGREDIENTS).register(add_to_ingredients)

# Other vanilla tabs you can add to:
# CreativeModeTabs.BUILDING_BLOCKS
# CreativeModeTabs.COMBAT
# CreativeModeTabs.TOOLS_AND_UTILITIES
# CreativeModeTabs.FOOD_AND_DRINKS
# CreativeModeTabs.REDSTONE_BLOCKS
# CreativeModeTabs.NATURAL_BLOCKS
# CreativeModeTabs.FUNCTIONAL_BLOCKS
# CreativeModeTabs.SPAWN_EGGS
```

### 12.2 Creating a Custom Creative Tab (Raw API)
```python
from net.fabricmc.fabric.api.itemgroup.v1 import FabricItemGroup
from net.minecraft.world.item import ItemStack
from net.minecraft.registry import Registry, Registries
from net.minecraft.resources import Identifier, ResourceKey
from net.minecraft.network.chat import Component

# Create the tab
tab_id = Identifier.of("mymod", "main_tab")
tab_key = ResourceKey.create(Registries.CREATIVE_MODE_TAB.key(), tab_id)

my_tab = FabricItemGroup.builder() \
    .title(Component.translatable("itemGroup.mymod.main")) \
    .icon(lambda: ItemStack(ruby)) \
    .displayItems(lambda ctx, entries: [
        entries.accept(ruby),
        entries.accept(ruby_sword),
        entries.accept(ruby_block),
        entries.accept(golden_berry),
    ]) \
    .build()

Registry.register(Registries.CREATIVE_MODE_TAB, tab_id, my_tab)
```

> [!TIP]
> Don't forget the translation key! Add `"itemGroup.mymod.main": "My Mod Items"` to your `en_us.json`.

---

## 13. Events & Callbacks

### 11.1 High-Level Event Decorators
```python
from minecraft_py import events

@events.on_server_start
def on_start(server):
    """Called when the server finishes starting."""
    print(f"Server started! Players online: {server.getPlayerCount()}")

@events.on_server_stop
def on_stop(server):
    """Called when the server begins stopping."""
    print("Server shutting down...")

@events.on_server_tick
def on_tick(server):
    """Called at the end of every server tick (20/sec)."""
    pass

@events.on_player_join
def on_join(handler, sender, server):
    """Called when a player connects."""
    print(f"{sender.getName()} joined!")

@events.on_block_broken
def on_break(world, player, pos, state, block_entity):
    """Called before a block is broken. Return False to cancel."""
    print(f"{player.getName()} broke a block at {pos}")
    return True  # Allow the break

@events.on_attack
def on_attack(player, target, hand):
    """Called when a player attacks an entity. Return False to cancel."""
    print(f"{player.getName()} attacked {target}")
    return True  # Allow the attack
```

### 11.2 Universal Entity Hooks

Hook into **any** entity's tick — not just `PythonEntity` — via the built-in `UniversalEntityMixin`:

```python
from minecraft_py import hooks

@hooks.on_entity_tick
def on_any_entity_tick(entity):
    """Called on EVERY entity tick in the world."""
    entity_type = str(entity.getType().builtInRegistryHolder().key().location())
    if entity_type == "minecraft:zombie":
        # Make all zombies glow at night
        pass
```

> [!WARNING]
> Universal hooks run on **every entity every tick**. Keep the logic lightweight or use type checks to filter early.

### 11.3 Fabric API Events (Raw)
```python
from net.fabricmc.fabric.api.event.lifecycle.v1 import ServerTickEvents, ServerLifecycleEvents
from net.fabricmc.fabric.api.entity.event.v1 import ServerEntityCombatEvents
from net.fabricmc.fabric.api.networking.v1 import ServerPlayConnectionEvents

# Direct event registration
ServerTickEvents.END_SERVER_TICK.register(lambda server: None)
ServerLifecycleEvents.SERVER_STARTED.register(lambda server: print("Started!"))
ServerPlayConnectionEvents.JOIN.register(lambda handler, sender, server: print("Player joined"))
```

### 11.4 Polyglot Event Bus

A lightweight cross-context pub/sub system for custom events:

```python
from minecraft_py import bus

# Subscribe to a custom event channel
bus.on("mymod:boss_spawned", lambda data: print(f"Boss spawned: {data}"))

# Emit an event (async, thread-safe)
bus.emit("mymod:boss_spawned", {"boss": "ZombieKing", "x": 100, "z": 200})
```

---

## 14. Commands

### 12.1 Simple Commands (High-Level API)
```python
from minecraft_py import commands

def heal_command(level, player):
    """Heals the player to full health."""
    from minecraft_py.player import heal
    heal(player, 20.0)
    print(f"Healed {player.getName()}!")

commands.register("heal", heal_command)
# Now players can type /heal in-game
```

### 12.2 Brigadier Commands (Raw API)

For full command trees with arguments, use Brigadier directly:

```python
from net.fabricmc.fabric.api.command.v2 import CommandRegistrationCallback
from net.minecraft.server.command.CommandManager import literal, argument
from net.minecraft.command.argument import EntityArgumentType, IntegerArgumentType
from net.minecraft.text import Text

def execute_smite(context):
    targets = EntityArgumentType.getEntities(context, "targets")
    power = IntegerArgumentType.getInteger(context, "power")
    
    for target in targets:
        level = target.level()
        from minecraft_py.world import strike_lightning
        for _ in range(power):
            strike_lightning(level, target.blockPosition())
    
    source = context.getSource()
    source.sendSuccess(lambda: Text.literal(f"Smote {len(list(targets))} entities!"), True)
    return 1

def register_commands(dispatcher, registryAccess, environment):
    dispatcher.register(
        literal("smite")
            .requires(lambda source: source.hasPermission(2))  # OP level 2
            .then(
                argument("targets", EntityArgumentType.entities())
                .then(
                    argument("power", IntegerArgumentType.integer(1, 10))
                    .executes(execute_smite)
                )
            )
    )

# In your ModInitializer:
CommandRegistrationCallback.EVENT.register(register_commands)
```

---

## 15. Networking (Client ↔ Server)

The framework provides two packet systems: **JSON packets** (structured data) and **Binary packets** (raw bytes).

### 13.1 JSON Packets

**Server-side receiver:**
```python
from minecraft_py import network

@network.on_receive("mymod", "ability_request")
def handle_ability(player, data):
    """Receives a JSON dict from the client."""
    ability_id = data["ability_id"]
    print(f"{player.getName()} used ability: {ability_id}")
    
    # Send a response back to the client
    network.send_to_client(player, "mymod", "ability_response", {
        "success": True,
        "cooldown": 60
    })
```

**Client-side receiver:**
```python
@network.on_client_receive("mymod", "ability_response")
def handle_response(data):
    """Receives a JSON dict from the server."""
    if data["success"]:
        print(f"Ability succeeded! Cooldown: {data['cooldown']}s")
```

**Sending packets:**
```python
# Client → Server
network.send_to_server("mymod", "ability_request", {"ability_id": "fireball"})

# Server → Client
network.send_to_client(player, "mymod", "ability_response", {"success": True})
```

### 13.2 Binary Packets

For performance-critical data (positions, raw buffers):

```python
@network.on_receive_bytes("mymod", "position_sync")
def handle_bytes(player, raw_bytes):
    """Receives raw bytes from the client."""
    import struct
    x, y, z = struct.unpack("fff", raw_bytes)
    print(f"Position: ({x}, {y}, {z})")

@network.on_client_receive_bytes("mymod", "position_sync")
def handle_client_bytes(raw_bytes):
    pass

# Sending raw bytes
import struct
data = struct.pack("fff", 1.0, 2.0, 3.0)
network.send_bytes_to_server("mymod", "position_sync", data)
network.send_bytes_to_client(player, "mymod", "position_sync", data)
```

### 13.3 Supported Data Types in JSON Packets

The recursive serializer supports: `int`, `float`, `bool`, `str`, `list`, `dict`, `None`, `bytes`.

---

## 16. Client Rendering & Input

### 14.1 Custom Keybinds
```python
from minecraft_py.client import register_keybind

def on_ability_key():
    from minecraft_py.network import send_to_server
    send_to_server("mymod", "ability_request", {"ability_id": "fireball"})

# GLFW key codes: https://www.glfw.org/docs/latest/group__keys.html
register_keybind("mymod_ability", 71, "MyMod", on_ability_key)  # G key
```

### 14.2 Camera Effects
```python
from minecraft_py.client import set_camera_shake, set_camera_roll

# Earthquake effect (x, y, z offset)
set_camera_shake(0.5, 0.3, 0.1)

# Camera roll (degrees)
set_camera_roll(15.0)

# Reset
set_camera_shake(0, 0, 0)
set_camera_roll(0)
```

### 14.3 Post-Processing Shaders
```python
from minecraft_py.client import set_shader, clear_shader

# Apply a built-in post-processing shader
set_shader("minecraft:shaders/post/blur")

# Remove the shader
clear_shader()
```

**Custom GLSL shaders:**
```python
from minecraft_py.graphics import apply_post_effect, clear_post_effect

# Generate and apply a custom fragment shader
apply_post_effect("mymod_vignette", """
#version 150
uniform sampler2D DiffuseSampler;
in vec2 texCoord;
out vec4 fragColor;

void main() {
    vec4 color = texture(DiffuseSampler, texCoord);
    float dist = distance(texCoord, vec2(0.5, 0.5));
    float vignette = smoothstep(0.8, 0.3, dist);
    fragColor = color * vignette;
}
""")

# Remove it
clear_post_effect()
```

### 14.4 HUD Overlays
```python
from minecraft_py.gui import on_hud_render, GuiContext

@on_hud_render
def render_hud(draw_context, tick_delta):
    """Called every render frame."""
    ctx = GuiContext(draw_context)
    ctx.draw_text("❤ Health: 20/20", 10, 10, 0xFF0000)
    ctx.draw_rect(10, 25, 110, 35, 0x80000000)  # Background bar
    ctx.draw_rect(10, 25, 110, 35, 0xFFFF0000)  # Health bar
```

### 14.5 Virtual Resource Pack Files

Inject assets at runtime without physical files:

```python
from minecraft_py.client import add_resourcepack_file

# Inject a custom item model JSON
add_resourcepack_file(
    "assets/mymod/models/item/generated_item.json",
    '{"parent": "minecraft:item/generated", "textures": {"layer0": "mymod:item/generated_item"}}'
)
```

---

## 17. GUI System

### 15.1 Custom Screens (Client-Side)
```python
from minecraft_py.gui import Screen

class SettingsScreen(Screen):
    def on_init(self):
        """Called when the screen opens. Add widgets here."""
        self.add_button(
            x=self.width // 2 - 50, y=self.height // 2,
            width=100, height=20,
            text="Click Me!",
            on_click=lambda: print("Button clicked!")
        )
        self.add_text_field(
            id="name_input",
            x=self.width // 2 - 75, y=self.height // 2 - 30,
            width=150, height=20,
            placeholder="Enter your name..."
        )
    
    def on_render(self, context, mouse_x, mouse_y, partial_tick):
        """Called every render frame."""
        ctx = GuiContext(context)
        ctx.draw_text("My Settings", self.width // 2 - 30, 20, 0xFFFFFF)
    
    def on_close(self):
        """Called when the screen is closed."""
        name = self.get_text_field_value("name_input")
        print(f"Name entered: {name}")

# Open the screen (must be called from client thread)
screen = SettingsScreen("My Settings")
screen.open()
```

### 15.2 Server-Driven GUI (ServerWindow)

Create multiplayer-safe GUIs that work from the server side:

```python
from minecraft_py.gui import ServerWindow

def show_shop(player):
    """Open a shop GUI for a player (server-side)."""
    window = ServerWindow("Item Shop")
    
    window.add_button(50, 50, 120, 20, "Buy Diamond Sword (10 gold)", 
        on_click=lambda: buy_item(player, "diamond_sword"))
    
    window.add_button(50, 80, 120, 20, "Buy Golden Apple (5 gold)",
        on_click=lambda: buy_item(player, "golden_apple"))
    
    window.open(player)  # Serialized and sent over the network
```

### 15.3 Container Screens (Inventory GUIs)

Machine blocks with `PythonBlockEntity` automatically get chest-style container GUIs:

```python
class SmelteryEntity(InventoryBlockEntity):
    inventory_size = 9  # 9 slots = 1 row
    name = "Smeltery"   # Title shown in GUI
    
    def tick(self, level, pos, state):
        # Process items...
        pass
```

When a player right-clicks the machine block, the container GUI opens automatically with the specified number of slots plus the player's inventory.

### 15.4 GuiContext Drawing API

```python
from minecraft_py.gui import GuiContext

def render(draw_context, mouse_x, mouse_y, partial_tick):
    ctx = GuiContext(draw_context)
    
    # Rectangles
    ctx.draw_rect(x1, y1, x2, y2, color)           # Filled rectangle (ARGB)
    
    # Text
    ctx.draw_text(text, x, y, color)                 # Left-aligned text
    
    # Textures
    ctx.draw_texture(texture_id, x, y, w, h, u, v, region_w, region_h)
    ctx.draw_sprite(sprite_id, x, y, w, h)           # Sprite from atlas
```

---

## 18. 3D Entity Models

### 16.1 Programmatic Model Builder

Build entity models part-by-part in Python:

```python
from minecraft_py.models import ModelBuilder, CubeListBuilder

def create_robot_model():
    builder = ModelBuilder.create()
    root = builder.root()
    
    # Body
    body_cubes = CubeListBuilder.create()
    body_cubes.tex_offs(0, 16)
    body_cubes.add_box(-4.0, 0.0, -2.0, 8.0, 12.0, 4.0)
    body = root.add_child_with_cubes("body", body_cubes, 0.0, 12.0, 0.0)
    
    # Head
    head_cubes = CubeListBuilder.create()
    head_cubes.tex_offs(0, 0)
    head_cubes.add_box(-4.0, 0.0, -4.0, 8.0, 8.0, 8.0)
    head = body.add_child_with_cubes("head", head_cubes, 0.0, 12.0, 0.0)
    
    # Right arm
    r_arm_cubes = CubeListBuilder.create()
    r_arm_cubes.tex_offs(40, 16)
    r_arm_cubes.add_box(-3.0, -2.0, -2.0, 4.0, 12.0, 4.0)
    body.add_child_with_cubes("right_arm", r_arm_cubes, -5.0, 10.0, 0.0)
    
    # Left arm
    l_arm_cubes = CubeListBuilder.create()
    l_arm_cubes.tex_offs(40, 16)
    l_arm_cubes.add_box(-1.0, -2.0, -2.0, 4.0, 12.0, 4.0)
    body.add_child_with_cubes("left_arm", l_arm_cubes, 5.0, 10.0, 0.0)
    
    return builder.bake(texture_width=64, texture_height=64)
```

### 16.2 BlockBench Models

Load `.geo.json` models exported from [BlockBench](https://www.blockbench.net/):

```python
from minecraft_py.models import load_blockbench
import json

# Load the model JSON (bundled in your mod's resources)
with open("assets/mymod/models/entity/dragon.geo.json") as f:
    model_json = f.read()

load_blockbench("mymod:dragon", model_json)
```

### 16.3 Custom Animation

Override `setup_animation` in your entity class:

```python
import math

class AnimatedEntity(BaseEntity):
    def setup_animation(self, model, walk_pos, walk_speed, age, y_rot, x_rot):
        """Called every render frame to animate the model."""
        # Head tracking
        head = model.getPart("head")
        if head:
            head.xRot = x_rot * 0.017453292  # degrees to radians
            head.yRot = y_rot * 0.017453292
        
        # Arm swing while walking
        right_arm = model.getPart("right_arm")
        left_arm = model.getPart("left_arm")
        if right_arm and left_arm:
            right_arm.xRot = math.cos(walk_pos * 0.6662) * 1.4 * walk_speed
            left_arm.xRot = math.cos(walk_pos * 0.6662 + math.pi) * 1.4 * walk_speed
        
        # Idle breathing animation
        body = model.getPart("body")
        if body:
            body.yRot = math.sin(age * 0.05) * 0.05
```

---

## 19. Status Effects

```python
from minecraft_py.effects import register_status_effect
from minecraft_py import MinecraftBridge

def on_poison_tick(level, entity, amplifier):
    """Called every tick_interval ticks while the effect is active."""
    damage = 1.0 * (amplifier + 1)
    entity.hurt(entity.damageSources().magic(), damage)
    return True  # Return True to continue the effect

wither_touch = register_status_effect(
    "mymod", "wither_touch",
    category="harmful",      # "harmful", "beneficial", or "neutral"
    color=0x551A8B,          # Particle color (purple)
    tick_interval=20,        # Tick every 20 ticks (1 second)
    on_tick=on_poison_tick
)
```

**Applying effects to entities:**
```python
from minecraft_py import MinecraftBridge

# Apply a status effect to a player
MinecraftBridge.addEffect = ...  # Use the raw Java API:

from net.minecraft.world.effect import MobEffectInstance
entity.addEffect(MobEffectInstance(wither_touch, 200, 1))  # 10 seconds, level 2
```

---

## 20. Custom Particles

```python
from minecraft_py.effects import register_particle

class SparkleParticle:
    """A custom particle with Python-controlled physics."""
    
    def __init__(self, particle):
        """Called when the particle spawns."""
        particle.setLifetime(40)       # 2 seconds
        particle.setScale(0.5)
        particle.setColor(1.0, 0.8, 0.2)   # Golden color
        particle.setAlpha(1.0)
    
    def tick(self, particle):
        """Called every frame."""
        # Fade out over lifetime
        progress = particle.age / particle.lifetime
        particle.setAlpha(1.0 - progress)
        
        # Spiral motion
        import math
        particle.xd = math.sin(particle.age * 0.3) * 0.02
        particle.zd = math.cos(particle.age * 0.3) * 0.02
        particle.yd += 0.001  # Float upward

# Register the particle type
sparkle = register_particle("mymod", "sparkle",
    textures=["mymod:sparkle"],  # Texture atlas reference
    python_class=SparkleParticle
)
```

**Spawning particles:**
```python
from minecraft_py.world import spawn_particle

# Spawn 10 sparkle particles at position
spawn_particle(level, "sparkle", x, y, z, count=10, speed=0.1, mod_id="mymod")
```

---

## 21. Sounds

### 19.1 Registering Custom Sounds
```python
from minecraft_py.sounds import register_sound

# Register a custom sound event
boss_roar = register_sound("mymod", "boss_roar", subtitles="Boss roars")
```

You need to provide:
- **Sound file**: `assets/mymod/sounds/boss_roar.ogg` (OGG Vorbis format)
- **sounds.json** is generated automatically by the framework

### 19.2 Playing Sounds
```python
from minecraft_py.world import play_sound

# Play a sound at a position
play_sound(level, "boss_roar", x, y, z, volume=2.0, pitch=0.8)

# Play a vanilla sound
play_sound(level, "entity.ender_dragon.growl", x, y, z, volume=1.0, pitch=1.0)
```

---

## 22. World Generation

### 20.1 Virtual Datapacks

Inject any datapack JSON at runtime without physical files:

```python
from minecraft_py.worldgen import add_datapack_file

# Add any datapack file dynamically
add_datapack_file("data/mymod/worldgen/biome/crystal_caves.json", json.dumps({
    "temperature": 0.5,
    "downfall": 0.5,
    "effects": {
        "sky_color": 0x77AAFF,
        "fog_color": 0xC0D8FF,
        "water_color": 0x3F76E4,
        "water_fog_color": 0x050533
    },
    "spawners": {},
    "spawn_costs": {},
    "carvers": {},
    "features": []
}))
```

### 20.2 Custom Biomes
```python
from minecraft_py.worldgen import add_biome

add_biome("mymod", "crystal_caves", {
    "temperature": 0.8,
    "downfall": 0.4,
    "effects": {
        "sky_color": 0xFF69B4,
        "fog_color": 0xFFB6C1,
        "water_color": 0xFF1493,
        "water_fog_color": 0xC71585,
        "grass_color": 0x7CFC00,
        "foliage_color": 0x00FF00
    }
})
```

### 20.3 Custom Dimensions
```python
from minecraft_py.worldgen import add_dimension

add_dimension("mymod", "void_realm", {
    "type": {
        "ultrawarm": False,
        "natural": False,
        "piglin_safe": False,
        "respawn_anchor_works": False,
        "bed_works": False,
        "has_raids": False,
        "has_skylight": False,
        "has_ceiling": True,
        "coordinate_scale": 1.0,
        "ambient_light": 0.1,
        "logical_height": 256,
        "infiniburn": "#minecraft:infiniburn_overworld",
        "min_y": 0,
        "height": 256
    }
})
```

### 20.4 Custom Structures
```python
from minecraft_py.worldgen import add_structure

add_structure("mymod", "ancient_tower", {
    "type": "minecraft:jigsaw",
    "start_pool": "mymod:tower/start",
    "size": 7,
    "max_distance_from_center": 80,
    "biomes": "#minecraft:has_structure/village_plains",
    "step": "surface_structures"
})
```

### 20.5 Python Density Functions (Custom Terrain)

Define terrain generation math in Python:

```python
from minecraft_py.worldgen import density_function

@density_function(min_val=-1.0, max_val=1.0)
def wavy_terrain(x, y, z):
    """Custom density function for terrain generation."""
    import math
    return math.sin(x * 0.05) * math.cos(z * 0.05) * 0.5 + (64 - y) * 0.01
```

### 20.6 Ore Generation
```python
from minecraft_py import MinecraftBridge

# Add a placed feature to the overworld's ore generation
MinecraftBridge.addOreToOverworld("mymod", "ruby_ore_placed")
```

---

## 23. Data Generation (Recipes, Loot, Enchantments)

All data generation uses the virtual datapack system — no physical JSON files needed!

### 21.1 Crafting Recipes

**Shaped recipe:**
```python
from minecraft_py.datagen import add_shaped_recipe

add_shaped_recipe(
    mod_id="mymod",
    name="ruby_sword",
    pattern=[
        " R ",
        " R ",
        " S "
    ],
    keys={
        "R": "mymod:ruby",
        "S": "minecraft:stick"
    },
    result="mymod:ruby_sword",
    count=1
)
```

**Shapeless recipe:**
```python
from minecraft_py.datagen import add_shapeless_recipe

add_shapeless_recipe(
    mod_id="mymod",
    name="ruby_block_from_rubies",
    ingredients=["mymod:ruby"] * 9,  # 9 rubies
    result="mymod:ruby_block",
    count=1
)
```

**Raw JSON recipe (for any recipe type):**
```python
from minecraft_py import MinecraftBridge

MinecraftBridge.addRecipeJson("mymod:smelting/ruby_from_ore", json.dumps({
    "type": "minecraft:smelting",
    "ingredient": "mymod:ruby_ore",
    "result": {"id": "mymod:ruby"},
    "experience": 1.0,
    "cookingtime": 200
}))
```

### 21.2 Loot Tables

**Entity drops:**
```python
from minecraft_py.datagen import add_entity_loot

add_entity_loot("mymod:zombie_king", "mymod:ruby", count=3, chance=0.5)
```

**Block drops:**
```python
from minecraft_py.datagen import add_block_loot

add_block_loot("mymod:ruby_ore", "mymod:ruby", count=1)
```

**Raw JSON loot table:**
```python
from minecraft_py import MinecraftBridge

MinecraftBridge.addLootTableJson("mymod:entities/boss_loot", json.dumps({
    "type": "minecraft:entity",
    "pools": [{
        "rolls": 1,
        "entries": [{
            "type": "minecraft:item",
            "name": "mymod:boss_trophy",
            "weight": 1
        }]
    }]
}))
```

### 21.3 Enchantments (Data-Driven)
```python
from minecraft_py.effects import register_enchantment

fire_aspect_plus = register_enchantment(
    mod_id="mymod",
    name="inferno",
    max_level=3,
    min_cost_base=10,
    max_cost_base=50,
    supported_items="#minecraft:enchantable/weapon",
    weight=5,              # Rarity (lower = rarer)
    slots=["mainhand"]
)
```

**Checking enchantment level:**
```python
from minecraft_py.player import get_enchantment_level

level = get_enchantment_level(player, "mymod", "inferno")
if level > 0:
    print(f"Player has Inferno {level}!")
```

---

## 24. Persistent Data Storage

Save Python data that persists across server restarts:

```python
from minecraft_py.persistent import Data

# Create a data store (auto-saves every 5 minutes + on shutdown)
economy = Data("mymod_economy", default_data={
    "bank": {},
    "prices": {"ruby": 100, "diamond": 500}
})

# Load existing data (or use defaults)
economy.load()

# Modify data
economy.data["bank"]["Steve"] = 1500

# Save explicitly (also auto-saves periodically)
economy.save()
```

### Per-Player Data
```python
from minecraft_py.persistent import Data

scores = Data("mymod_scores")
scores.load()

@events.on_player_join
def on_join(handler, sender, server):
    name = str(sender.getName())
    if name not in scores.data:
        scores.data[name] = {"kills": 0, "deaths": 0}
        scores.save()
```

---

## 25. Advanced Mixin Support (Core Modding)

The framework **fully supports Mixins** — Python decorators are compiled to real Java Mixin classes at build time by the `mixin_generator.py` AST compiler.

> [!IMPORTANT]
> **Mixins require the Gradle build system.** The `python-mixins.gradle` script must be applied in your `build.gradle`, and you must run `./gradlew build` to generate the Java Mixin classes.

### 23.1 Setup

Add to your `build.gradle`:
```groovy
apply from: "python-mixins.gradle"
```

### 23.2 Simple Injection (`@Inject`)

```python
from minecraft_py.mixins import mixin

@mixin.inject(
    target_class="net.minecraft.entity.player.PlayerEntity",
    method="jump",
    at="HEAD"
)
def on_player_jump(self, info: "CallbackInfo"):
    """Called at the beginning of PlayerEntity.jump()"""
    print(f"Player jumped!")
```

### 23.3 Cancellable Injection
```python
@mixin.inject(
    target_class="net.minecraft.entity.player.PlayerEntity",
    method="attack",
    at="HEAD",
    cancellable=True
)
def on_player_attack(self, target: "net.minecraft.entity.Entity", info: "CallbackInfo"):
    """Cancel attacks on baby animals."""
    if target.isBaby():
        # Cancel the attack
        from minecraft_py import MinecraftBridge
        MinecraftBridge.cancelCallback(info)
        print("You can't attack babies!")
```

### 23.4 Injection Points

| Point | Description |
|---|---|
| `HEAD` | Beginning of the method |
| `TAIL` | End of the method (before final return) |
| `RETURN` | At every `return` statement |
| `INVOKE` | At a specific method call (requires `target`) |

### 23.5 Advanced: `@Redirect`

> [!IMPORTANT]
> **Type Hints are mandatory** for `@Redirect` because Java requires exact method signatures. Use Python type hint strings for Java type names.

```python
@mixin.redirect(
    target_class="net.minecraft.entity.player.PlayerEntity",
    method="tick",
    at_target="Lnet/minecraft/entity/Entity;move(Lnet/minecraft/entity/MovementType;Lnet/minecraft/util/math/Vec3d;)V"
)
def redirect_move(
    self,
    entity: "net.minecraft.entity.Entity",
    movement_type: "net.minecraft.entity.MovementType",
    vec3d: "net.minecraft.util.math.Vec3d"
) -> "void":
    """Intercept and modify entity movement."""
    print("Movement redirected!")
    # Call original or skip entirely
```

### 23.6 Build Pipeline

```
1. You write Python @Mixin decorators in src/main/python/
2. Run: ./gradlew build
3. mixin_generator.py parses Python AST
4. Generates Java .java files → build/generated/sources/mixins/
5. Java compiler compiles them alongside the framework
6. At runtime, generated code calls MixinBridge.invoke()
7. MixinBridge dispatches to your Python callback
```

---

## 26. Performance & Optimization

### 24.1 FastPath Bridge

For bulk operations that would be slow with individual polyglot calls:

```python
from minecraft_py.fastmath import (
    fill_blocks, replace_blocks, fill_sphere,
    get_entities_in_box, raycast, raycast_from_entity,
    get_positions, get_healths
)

# Fill a cuboid region (100x faster than setting blocks one-by-one)
changed = fill_blocks(level, x1, y1, z1, x2, y2, z2, "minecraft:stone")

# Replace specific blocks in a region
replaced = replace_blocks(level, x1, y1, z1, x2, y2, z2, "minecraft:dirt", "minecraft:grass_block")

# Fill a sphere
fill_sphere(level, center_x, center_y, center_z, radius=10, block_id="minecraft:glass")

# Get all entities in an area (AABB query)
entities = get_entities_in_box(level, x1, y1, z1, x2, y2, z2)

# Batch extract positions (returns flat double array [x1,y1,z1, x2,y2,z2, ...])
positions = get_positions(entities)

# Batch extract health values
healths = get_healths(entities)

# Raycast from an entity's eye position along their look vector
hit = raycast_from_entity(player, max_distance=50.0)

# Raycast from arbitrary position/direction
hit = raycast(level, source_entity, start_x, start_y, start_z, dir_x, dir_y, dir_z, max_distance=100.0)
# Returns: Entity (if hit entity), BlockPos (if hit block), or None
```

### 24.2 JIT-Compiled Math (`@jit_math`)

Compile Python math functions directly to JVM bytecode via ASM — zero polyglot overhead:

```python
from minecraft_py.jit import jit_math

@jit_math
def terrain_noise(x, y, z):
    """Compiled to raw JVM bytecode at decoration time."""
    import math
    return math.sin(x * 0.1) * math.cos(z * 0.1) + y * 0.01

# Returns a FastMathNode Java object
# terrain_noise.compute(x, y, z) runs at native Java speed
```

**Supported operations in `@jit_math`:**
- Arithmetic: `+`, `-`, `*`, `/`
- Math functions: `math.sin`, `math.cos`, `math.tan`, `math.sqrt`, `math.abs`
- Variables: `x`, `y`, `z`
- Constants: Any numeric literal

### 24.3 Thread-Safe Task Scheduling

Execute code on the main server thread safely:

```python
from minecraft_py.tasks import run_on_main_thread

def delayed_action():
    # This runs on the main thread, safe for Minecraft API calls
    print("Running on main thread!")

run_on_main_thread(delayed_action)
```

---

## 27. Async Operations

### 25.1 Asyncio Integration
```python
from minecraft_py.async_bridge import get_event_loop, await_java

import asyncio

async def load_chunks_around_player(player):
    """Asynchronously load chunks around a player."""
    from minecraft_py.async_bridge import get_chunk_async
    
    level = player.serverLevel()
    px, pz = int(player.getX()) >> 4, int(player.getZ()) >> 4
    
    tasks = []
    for dx in range(-3, 4):
        for dz in range(-3, 4):
            tasks.append(get_chunk_async(level, px + dx, pz + dz))
    
    chunks = await asyncio.gather(*tasks)
    print(f"Loaded {len(chunks)} chunks!")

# Schedule the coroutine
loop = get_event_loop()
asyncio.run_coroutine_threadsafe(load_chunks_around_player(player), loop)
```

### 25.2 Java CompletableFuture → Python
```python
from minecraft_py.async_bridge import await_java

async def example():
    java_future = some_java_method_returning_future()
    result = await await_java(java_future)
    print(f"Future resolved: {result}")
```

---

## 28. Third-Party Mod Interop

### 26.1 Check If a Mod Is Loaded
```python
from minecraft_py.mods import is_loaded, get_class

if is_loaded("sodium"):
    print("Sodium detected! Adjusting renderer...")

if is_loaded("iris"):
    ShaderPack = get_class("net.irisshaders.iris.api.v0.IrisApi")
    if ShaderPack:
        print("Iris shaders available!")
```

### 26.2 Accessing Other Mod APIs
```python
from minecraft_py.mods import get_class

# Access any class from any loaded mod
SomeModAPI = get_class("com.othermod.api.SomeModAPI")
if SomeModAPI:
    SomeModAPI.doSomething()
```

---

## 29. Native Python (NumPy, TensorFlow, etc.)

The framework includes a **native CPython sidecar process** for running code that requires C-extensions (which GraalPy doesn't support):

```python
from minecraft_py.native import run

# Execute code in native CPython (separate process)
result = run("""
import numpy as np

# Generate Perlin noise for terrain
def generate_heightmap(size):
    x = np.linspace(0, 4*np.pi, size)
    z = np.linspace(0, 4*np.pi, size)
    X, Z = np.meshgrid(x, z)
    heightmap = np.sin(X) * np.cos(Z) * 50 + 64
    return heightmap.tolist()

return generate_heightmap(16)
""")

# result is a Python list (deserialized from JSON)
heightmap = result
```

**With variables:**
```python
result = run("""
import numpy as np
center = np.array([x, y, z])
random_offsets = np.random.randn(count, 3) * spread
positions = (center + random_offsets).tolist()
return positions
""", variables={"x": 100, "y": 64, "z": 200, "count": 50, "spread": 10})
```

> [!WARNING]
> The sidecar communicates via TCP sockets with JSON serialization. It's designed for batch computation, not real-time per-tick operations.

---

## 30. Dynamic Class Generation (ByteBuddy)

For cases where you need to extend abstract Java classes that can't be directly subclassed via GraalPy:

```python
# The java_extend() global function creates a real Java subclass at runtime
# using ByteBuddy, with all method calls proxied back to Python

from net.minecraft.world.entity.ai.goal import Goal

class MyCustomGoal:
    def canUse(self):
        return True
    def start(self):
        print("Goal started!")
    def tick(self):
        pass

# Create a real Java Goal subclass backed by Python
java_goal = java_extend(Goal, MyCustomGoal())
entity.addPythonGoal(1, java_goal)
```

The `DynamicClassBridge` uses ByteBuddy to generate JVM bytecode for a subclass, then intercepts all method calls via `PythonInterceptor`, routing them to your Python object.

---

## 31. Multi-File Mod Organization

As your mod grows, you'll want to split code across multiple Python files.

### 31.1 Folder Mod (Multi-File)

For folder mods, place all your `.py` files in the mod folder and use relative imports:

```
my-big-mod/
├── fabric.mod.json
├── pack.mcmeta
├── main.py              ← Entrypoint: imports everything
├── items.py             ← Item definitions
├── blocks.py            ← Block definitions
├── entities.py          ← Entity definitions
├── commands.py          ← Command registration
├── events.py            ← Event handlers
├── gui.py               ← GUI screens
├── config.py            ← Shared constants
└── assets/mymod/        ← (assets as usual)
```

**`config.py`** — Shared constants:
```python
MOD_ID = "mymod"
MOD_NAME = "My Big Mod"
VERSION = "1.0.0"
```

**`items.py`** — Item definitions:
```python
from minecraft_py import item
from minecraft_py.items import create_sword, create_food
from config import MOD_ID

@item(MOD_ID, "ruby")
def ruby(): pass

ruby_sword = create_sword(MOD_ID, "ruby_sword", material="DIAMOND", damage=8, speed=-2.4)
golden_berry = create_food(MOD_ID, "golden_berry", nutrition=8, saturation=1.5)
```

**`main.py`** — Entrypoint that imports all modules:
```python
from net.fabricmc.api import ModInitializer
from org.slf4j import LoggerFactory
from config import MOD_ID

logger = LoggerFactory.getLogger(MOD_ID)

# Import all modules to trigger their registration
import items      # Registers all items
import blocks     # Registers all blocks
import entities   # Registers all entities
import commands   # Registers all commands
import events     # Registers all event handlers

class MyMod(ModInitializer):
    def onInitialize(self):
        logger.info(f"[{MOD_ID}] All modules loaded!")

mod_instance = MyMod()
```

> [!IMPORTANT]
> **Import order matters!** If `blocks.py` references items from `items.py`, import `items` first in `main.py`. Python executes imports in order.

### 31.2 Gradle Mod (Package Structure)

For Gradle mods, use a proper Python package:

```
src/main/python/mymod/
├── __init__.py          ← Package init (can be entrypoint)
├── main.py              ← ModInitializer
├── items/__init__.py    ← Item sub-package
├── blocks/__init__.py   ← Block sub-package
├── entities/__init__.py ← Entity sub-package
└── utils.py             ← Utility functions
```

**`mymod/__init__.py`:**
```python
# This runs when the package is imported
from . import items, blocks, entities
from .main import mod_instance
```

**`fabric.mod.json` entrypoint:**
```json
{ "adapter": "python", "value": "mymod::mod_instance" }
```

### 31.3 Tips for Large Mods

- **One file per system**: Keep items, blocks, entities, events, and commands in separate files
- **Use a shared `config.py`**: Put `MOD_ID`, version, and shared constants in one place
- **Lazy imports**: If a module is only needed in a callback, import it inside the function to avoid circular imports
- **Hot-reload friendly**: `/py reload` re-executes all modules, so avoid side effects in module-level code that can't be repeated safely

---

## 32. In-Game Developer Tools

### 29.1 Hot Reload
```
/py reload
```
Reloads all Python mods without restarting the server. Clears module caches, re-executes entrypoints, and re-binds callbacks. Callback maps are designed for hot-reload — items, blocks, and entities keep working with updated logic.

### 29.2 Interactive REPL
```
/py eval <code>
```
Execute arbitrary Python in the server context. The REPL injects `player`, `level`, and `server` variables:
```
/py eval player.setHealth(1.0)
/py eval MinecraftBridge.giveItem(player, "minecraft", "diamond", 64)
/py eval print(server.getPlayerCount())
```

### 29.3 In-Game Package Manager
```
/py pip
```
Opens a GUI for installing Python packages from PyPI directly in-game. Packages are installed via the native sidecar process.

### 29.4 Diagnostics
```
/py test
```
Runs the built-in test suite (7 test modules: blocks, items, events, entities, network, GUI, mixins) and reports results in chat.

---

## 33. Testing & Debugging

A comprehensive guide to testing your mod and diagnosing issues.

### 33.1 In-Game Testing Commands

Once your mod is loaded, use these commands to test content:

```
# Give yourself a custom item
/give @s mymod:ruby 64
/give @s mymod:ruby_sword
/give @s mymod:golden_berry 16

# Place a custom block
/setblock ~ ~ ~ mymod:ruby_block
/setblock ~ ~1 ~ mymod:altar

# Summon a custom entity
/summon mymod:guardian ~ ~ ~
/summon mymod:zombie_king ~ ~ ~

# Apply a custom effect
/effect give @s mymod:ruby_weakness 60 2

# Test custom commands
/stats
/heal

# Run the framework test suite
/py test

# Live Python execution (OP required)
/py eval player.getHealth()
/py eval from minecraft_py.player import give_item; give_item(player, "mymod", "ruby", 64)
```

### 33.2 Reading Logs

Minecraft logs are your primary debugging tool:

- **Game logs**: `.minecraft/logs/latest.log`
- **Crash reports**: `.minecraft/crash-reports/`

Python `print()` statements and `logger.info()` calls appear in the game log. Use `logger` for production, `print()` for quick debugging:

```python
from org.slf4j import LoggerFactory
logger = LoggerFactory.getLogger("mymod")

logger.info("This shows as [mymod/INFO] in logs")
logger.warn("Something looks wrong")
logger.error("Something broke!")
print("Quick debug output")  # Shows as [stdout] in logs
```

### 33.3 Common Errors & Solutions

| Error | Cause | Solution |
|---|---|---|
| `NullPointerException: Block id not set` | Forgot `.setId(key)` on Block/Item Settings | Add `props.setId(resource_key)` before construction |
| `SyntaxError: invalid syntax` on `.class` | Used `MyClass.class` in Python | Use `getattr(MyClass, "class")` or `java_class(MyClass)` |
| Item shows as `item.mymod.ruby` | Missing translation file | Create `assets/mymod/lang/en_us.json` |
| Block is invisible | Missing model/blockstate JSON | Create all 3 files: blockstate + block model + item model |
| Entity is invisible | Missing texture file | Ensure texture path matches `get_texture()` return value |
| `Unable to summon entity: duplicate UUIDs` | Spawning passengers during construction | Defer passenger spawning to first `tick()` |
| `NoSuchMethodError` | Method signature changed in 26.2 | Use reflection or check 26.2 API changes table |
| `ClassNotFoundException` | Wrong import path for 26.2 | Use `net.minecraft.resources.Identifier` not `ResourceLocation` |
| `RegistryFrozenException` | Registering content too late | Ensure all registration runs during `onInitialize()` |
| `/py reload` doesn't update blocks/items | Blocks/items are immutable after registration | Only callback logic (on_use, tick) updates on reload; registry entries are permanent |
| `PolyglotException` in entity tick | Python error in tick callback | Check logs for the Python traceback inside the PolyglotException |
| Textures show as purple/black checkerboard | Texture file not found or wrong path | Verify file exists at exact path and filename matches (case-sensitive!) |
| Sound doesn't play | Missing .ogg file or wrong format | Sounds must be `.ogg` (Ogg Vorbis). MP3/WAV won't work |

### 33.4 Debugging Python Exceptions

When Python code throws an exception inside GraalPy, you'll see a `PolyglotException` in the Java log. The actual Python traceback is embedded inside:

```
[Server thread/ERROR] PolyglotException: TypeError: unsupported operand type(s) for +: 'int' and 'str'
  at mymod/main.py:42  ← This is your Python file and line number
  at mymod/items.py:15
```

Look for the **file path and line number** after `at` to find your bug.

### 33.5 Hot-Reload Debugging Workflow

The fastest way to iterate on your mod:

1. Launch Minecraft with your mod
2. Edit your `.py` files in your code editor
3. Run `/py reload` in-game
4. Test the changes immediately
5. Check `.minecraft/logs/latest.log` if something went wrong
6. Repeat

> [!TIP]
> **Keep the log file open** in a terminal with `tail -f logs/latest.log` (Linux/macOS) or open it in a text editor with auto-refresh. This lets you see Python errors in real-time.

### 33.6 The `/py eval` REPL

The in-game REPL is incredibly powerful for debugging:

```
# Inspect an entity's properties
/py eval [str(e.getType()) for e in level.getEntities(None, player.getBoundingBox().inflate(10))]

# Check what items a player has
/py eval [str(player.getInventory().getItem(i)) for i in range(36) if not player.getInventory().getItem(i).isEmpty()]

# Test a function before adding it to your mod
/py eval from minecraft_py.world import strike_lightning; strike_lightning(level, player.blockPosition())

# Check if your mod's module is loaded
/py eval import sys; print([m for m in sys.modules if 'mymod' in m])
```

---

## 34. Built-in Utilities

### 30.1 `java_class(cls)`

Converts a Python class reference to a Java `Class<T>` object — required by methods with generic signatures:

```python
from net.minecraft.entity.mob import ZombieEntity

# Methods like getEntitiesByType need Class<T>, not a Python reference
zombies = world.getEntitiesByType(java_class(ZombieEntity), area, predicate)
```

### 30.2 `java_extend(base_class, python_obj)`

Creates a real Java subclass at runtime backed by a Python object (via ByteBuddy).

### 30.3 Player Utilities
```python
from minecraft_py.player import give_item, heal, get_enchantment_level

give_item(player, "mymod", "ruby", 10)
heal(player, 20.0)
level = get_enchantment_level(player, "mymod", "inferno")
```

### 30.4 World Utilities
```python
from minecraft_py.world import (
    set_block, get_block, create_explosion, strike_lightning,
    spawn_particle, play_sound, fill_blocks,
    get_entities_in_box, raycast, raycast_from_entity
)

set_block(level, pos, "mymod", "ruby_block")
block_id = get_block(level, pos)
create_explosion(level, pos, power=4.0)
strike_lightning(level, pos)
spawn_particle(level, "flame", x, y, z, count=20, speed=0.1, mod_id="minecraft")
play_sound(level, "entity.ender_dragon.growl", x, y, z, volume=1.0, pitch=0.5)
```

### 30.5 Message Utilities
```python
from minecraft_py import MinecraftBridge

MinecraftBridge.sendMessage(player, "Hello from Python!")
MinecraftBridge.sendTitle(player, "BOSS FIGHT", "Prepare yourself!", 10, 70, 20)
MinecraftBridge.sendActionBar(player, "Mana: 50/100")
```

### 30.6 The Healer (Auto-Fix Method Names)

When Minecraft updates rename methods, the healer uses fuzzy matching to find the correct method:

```python
from minecraft_py.healer import wrap

# Wraps a Java object with fuzzy method resolution
safe_player = wrap(player)
safe_player.someOldMethodName()  # Finds the closest matching method
```

### 30.7 IDE Stub Generation

Generate `.pyi` type stubs for autocomplete in your IDE:

```bash
python -m minecraft_py stubs
```

This generates type hints for `Player`, `Level`, `Entity`, `BlockPos`, and all `minecraft_py` modules.

---

## 35. Critical Best Practices & 26.2 Quirks

### 31.1 The Java Bridge Pattern

> [!CAUTION]
> **Minecraft fields are obfuscated in production.** Never access internal Minecraft fields directly from Python.

```python
# ❌ WRONG — crashes in production due to obfuscation
entity.goalSelector.addGoal(1, some_goal)

# ✅ CORRECT — use the bridge method
entity.addPythonGoal(1, some_goal)
```

Custom Java bridge classes (like `PythonEntity`) provide public methods that the Java compiler resolves at compile time, bypassing runtime obfuscation entirely.

### 31.2 Reserved Keyword: `class`

```python
# ❌ WRONG — SyntaxError
player_class = PlayerEntity.class

# ✅ CORRECT
player_class = getattr(PlayerEntity, "class")
# or
player_class = java_class(PlayerEntity)
```

### 31.3 Checking Python Methods from Java

> [!WARNING]
> `Value.hasMember("method_name")` returns `false` for methods defined on a Python class (vs instance). The framework uses `try-catch` for all Python method invocations instead.

If you're writing custom Java bridge classes, always use:
```java
try {
    pythonObject.invokeMember("tick", this);
} catch (UnsupportedOperationException | PolyglotException ignored) {}
```

### 31.4 Minecraft 26.2 Specifics

| Topic | 26.2 Behavior |
|---|---|
| **Obfuscation** | 26.2 is **natively unobfuscated** — no Mojang→Intermediary mapping needed |
| **ResourceLocation** | Renamed to `net.minecraft.resources.Identifier` |
| **Block/Item Registration** | `Properties.setId(ResourceKey)` is **mandatory** before construction |
| **BlockEntity NBT** | Uses `ValueOutput`/`ValueInput` instead of `CompoundTag` |
| **Java Version** | Requires **Java 25** |
| **Loom Obfuscation** | `fabric.loom.disableObfuscation=true` in `gradle.properties` |

### 31.5 Passenger Entity UUID Duplication

```python
# ❌ WRONG — crashes /summon with duplicate UUID
class Boss(BaseEntity):
    def __init__(self):
        passenger = ItemDisplay(...)
        level.addFreshEntity(passenger)  # DON'T spawn during construction!

# ✅ CORRECT — defer to first tick
class Boss(BaseEntity):
    def __init__(self):
        self.model_spawned = False
    
    def tick(self, entity):
        if not self.model_spawned:
            entity.attach3DModel("mymod:boss_model")
            self.model_spawned = True
```

### 31.6 Registry Freezing

Minecraft 26.2 freezes registries after initialization. The framework calls `MinecraftBridge.defrostRegistries()` automatically when registering content, using deep reflection to unfreeze `MappedRegistry` instances.

### 31.7 Volatile Method Signatures

Method signatures change across Minecraft snapshots. The framework uses Java Reflection to handle this:

```java
// Instead of hardcoding: entity.startRiding(vehicle, true)
// The framework searches for matching method signatures at runtime
for (Method m : Entity.class.getMethods()) {
    if (m.getName().equals("startRiding") && m.getParameterCount() == 2) {
        m.invoke(entity, vehicle, true);
    }
}
```

---

## 36. Complete Example Mod

Here's a complete mod demonstrating items, blocks, entities, events, commands, GUI, and networking:

```python
"""
My Complete Mod - A full-featured mod written entirely in Python.
File: main.py
"""
from net.fabricmc.api import ModInitializer
from org.slf4j import LoggerFactory
from minecraft_py import item, block, entity, events, commands
from minecraft_py.items import create_sword, create_food
from minecraft_py.blocks import create_block
from minecraft_py.entities import BaseEntity, register_entity
from minecraft_py.ai import add_float_goal, add_melee_attack_goal, add_wander_goal, add_target_player_goal
from minecraft_py.datagen import add_shaped_recipe, add_entity_loot
from minecraft_py.effects import register_status_effect
from minecraft_py.sounds import register_sound
from minecraft_py.persistent import Data
from minecraft_py.gui import on_hud_render, GuiContext, Screen
from minecraft_py.player import give_item, heal

MOD_ID = "mymod"
logger = LoggerFactory.getLogger(MOD_ID)

# ═══════════════════════════════════════════
# Items
# ═══════════════════════════════════════════
@item(MOD_ID, "ruby")
def ruby(): pass

def on_staff_use(level, player, hand):
    """Heals the player when used."""
    heal(player, 10.0)
    from minecraft_py.world import spawn_particle
    spawn_particle(level, "heart", player.getX(), player.getY() + 1, player.getZ(), count=5, speed=0.2)

healing_staff = create_sword(MOD_ID, "healing_staff", material="GOLD", damage=2, speed=-1.0, on_use=on_staff_use)

golden_berry = create_food(MOD_ID, "golden_berry", nutrition=8, saturation=1.5)

# ═══════════════════════════════════════════
# Blocks
# ═══════════════════════════════════════════
@block(MOD_ID, "ruby_block", hardness=5.0)
def ruby_block(): pass

def on_altar_use(level, player, pos):
    from minecraft_py.world import strike_lightning
    strike_lightning(level, pos)

altar = create_block(MOD_ID, "altar", hardness=50.0, on_use=on_altar_use)

# ═══════════════════════════════════════════
# Entities
# ═══════════════════════════════════════════
class Guardian(BaseEntity):
    def register_goals(self, entity):
        add_float_goal(entity, priority=0)
        add_melee_attack_goal(entity, priority=2, speed_modifier=1.4)
        add_wander_goal(entity, priority=5)
        add_target_player_goal(entity, priority=1)
    
    def tick(self, entity):
        if entity.getHealth() < entity.getMaxHealth() * 0.25:
            entity.setGlowing(True)
            entity.setAttributeBaseValue("movement_speed", 0.5)
    
    def get_texture(self):
        return f"{MOD_ID}:textures/entity/guardian.png"

guardian_type = register_entity(MOD_ID, "guardian", Guardian, health=60.0, speed=0.3, damage=8.0)

# ═══════════════════════════════════════════
# Recipes
# ═══════════════════════════════════════════
add_shaped_recipe(MOD_ID, "ruby_block", [" RRR", "RRR", "RRR"],
    {"R": f"{MOD_ID}:ruby"}, f"{MOD_ID}:ruby_block")

add_shaped_recipe(MOD_ID, "healing_staff", [" R ", " G ", " S "],
    {"R": f"{MOD_ID}:ruby", "G": "minecraft:gold_ingot", "S": "minecraft:stick"},
    f"{MOD_ID}:healing_staff")

# ═══════════════════════════════════════════
# Loot
# ═══════════════════════════════════════════
add_entity_loot(f"{MOD_ID}:guardian", f"{MOD_ID}:ruby", count=5, chance=0.8)

# ═══════════════════════════════════════════
# Sounds
# ═══════════════════════════════════════════
guardian_roar = register_sound(MOD_ID, "guardian_roar", subtitles="Guardian roars")

# ═══════════════════════════════════════════
# Effects
# ═══════════════════════════════════════════
def on_weakness_tick(level, entity, amplifier):
    entity.setAttributeBaseValue("movement_speed", max(0.05, 0.3 - 0.1 * amplifier))
    return True

ruby_weakness = register_status_effect(MOD_ID, "ruby_weakness",
    category="harmful", color=0xFF0000, tick_interval=10, on_tick=on_weakness_tick)

# ═══════════════════════════════════════════
# Persistent Data
# ═══════════════════════════════════════════
stats = Data("mymod_stats", default_data={"total_kills": 0})
stats.load()

# ═══════════════════════════════════════════
# Events
# ═══════════════════════════════════════════
@events.on_server_start
def on_start(server):
    logger.info(f"My Mod loaded! Total kills: {stats.data['total_kills']}")

@events.on_player_join
def on_join(handler, sender, server):
    from minecraft_py import MinecraftBridge
    MinecraftBridge.sendMessage(sender, "§6Welcome! §7This server runs §bPython mods§7!")

@events.on_attack
def on_attack(player, target, hand):
    stats.data["total_kills"] = stats.data.get("total_kills", 0) + 1
    return True

# ═══════════════════════════════════════════
# Commands
# ═══════════════════════════════════════════
def stats_command(level, player):
    from minecraft_py import MinecraftBridge
    MinecraftBridge.sendMessage(player, f"§6Total kills: §f{stats.data['total_kills']}")

commands.register("stats", stats_command)

# ═══════════════════════════════════════════
# HUD
# ═══════════════════════════════════════════
@on_hud_render
def render_kills_hud(draw_context, tick_delta):
    ctx = GuiContext(draw_context)
    ctx.draw_text(f"Kills: {stats.data.get('total_kills', 0)}", 10, 10, 0xFFD700)

# ═══════════════════════════════════════════
# Entry Point
# ═══════════════════════════════════════════
class MyMod(ModInitializer):
    def onInitialize(self):
        logger.info("[MyMod] Initialized successfully!")

mod_instance = MyMod()
```

---

## 37. API Quick Reference

### `minecraft_py` Top-Level Decorators

| Decorator | Description |
|---|---|
| `@item(mod_id, item_id, **kwargs)` | Register a simple item |
| `@block(mod_id, block_id, hardness=1.5, **kwargs)` | Register a block with BlockItem |
| `@entity(mod_id, name, health, speed, damage, **kwargs)` | Register a custom entity |

### `minecraft_py.items`

| Function | Description |
|---|---|
| `create_item(mod_id, item_id, max_count=64, on_use=None)` | Basic item |
| `create_food(mod_id, item_id, nutrition, saturation, on_use=None)` | Food item |
| `create_sword(mod_id, item_id, material, damage, speed, on_use=None)` | Sword tool |
| `create_pickaxe(mod_id, item_id, material, damage, speed, on_use=None)` | Pickaxe tool |
| `create_axe(mod_id, item_id, material, damage, speed, on_use=None)` | Axe tool |
| `create_shovel(mod_id, item_id, material, damage, speed, on_use=None)` | Shovel tool |
| `create_hoe(mod_id, item_id, material, damage, speed, on_use=None)` | Hoe tool |

### `minecraft_py.blocks`

| Function | Description |
|---|---|
| `create_block(mod_id, block_id, hardness, on_use=None, on_step=None)` | Block + BlockItem |

### `minecraft_py.entities`

| Function | Description |
|---|---|
| `register_entity(mod_id, name, python_class, health, speed, damage)` | Register entity type |

### `minecraft_py.ai`

| Function | Description |
|---|---|
| `add_float_goal(entity, priority)` | Swim in water |
| `add_melee_attack_goal(entity, priority, speed_modifier, following_target_even_if_not_seen)` | Melee combat |
| `add_wander_goal(entity, priority, speed_modifier)` | Random wandering |
| `add_target_player_goal(entity, priority, must_see)` | Target nearest player |
| `create_custom_goal(python_class)` | Custom AI goal |

### `minecraft_py.events`

| Decorator | Callback Signature |
|---|---|
| `@events.on_server_start` | `func(server)` |
| `@events.on_server_stop` | `func(server)` |
| `@events.on_server_tick` | `func(server)` |
| `@events.on_player_join` | `func(handler, sender, server)` |
| `@events.on_block_broken` | `func(world, player, pos, state, block_entity) → bool` |
| `@events.on_attack` | `func(player, target, hand) → bool` |

### `minecraft_py.network`

| Function | Description |
|---|---|
| `@network.on_receive(mod_id, packet_id)` | Server JSON receiver |
| `@network.on_client_receive(mod_id, packet_id)` | Client JSON receiver |
| `@network.on_receive_bytes(mod_id, packet_id)` | Server binary receiver |
| `@network.on_client_receive_bytes(mod_id, packet_id)` | Client binary receiver |
| `network.send_to_client(player, mod_id, packet_id, data)` | Server → Client |
| `network.send_to_server(mod_id, packet_id, data)` | Client → Server |
| `network.send_bytes_to_client(player, mod_id, packet_id, bytes)` | Server → Client (binary) |
| `network.send_bytes_to_server(mod_id, packet_id, bytes)` | Client → Server (binary) |

### `minecraft_py.gui`

| Class/Function | Description |
|---|---|
| `Screen` | Custom client-side screen (override `on_init`, `on_render`, `on_close`) |
| `ServerWindow` | Server-driven multiplayer GUI |
| `GuiContext` | Drawing API (`draw_rect`, `draw_text`, `draw_texture`, `draw_sprite`) |
| `@on_hud_render` | HUD overlay callback |

### `minecraft_py.worldgen`

| Function | Description |
|---|---|
| `add_datapack_file(path, json_str)` | Inject virtual datapack file |
| `add_biome(mod_id, name, data)` | Custom biome |
| `add_dimension(mod_id, name, data)` | Custom dimension |
| `add_structure(mod_id, name, data)` | Custom structure |
| `@density_function(min_val, max_val)` | Custom terrain density function |

### `minecraft_py.datagen`

| Function | Description |
|---|---|
| `add_shaped_recipe(mod_id, name, pattern, keys, result, count)` | Shaped crafting |
| `add_shapeless_recipe(mod_id, name, ingredients, result, count)` | Shapeless crafting |
| `add_entity_loot(entity_id, drop_id, count, chance)` | Entity drops |
| `add_block_loot(block_id, drop_id, count)` | Block drops |
| `add_loot_table(loot_id, json_data)` | Raw loot table |

### `minecraft_py.effects`

| Function | Description |
|---|---|
| `register_status_effect(mod_id, name, category, color, tick_interval, on_tick)` | Custom effect |
| `register_particle(mod_id, name, textures, python_class)` | Custom particle |
| `register_enchantment(mod_id, name, max_level, ...)` | Data-driven enchantment |

### `minecraft_py.fastmath`

| Function | Description |
|---|---|
| `fill_blocks(level, x1,y1,z1, x2,y2,z2, block_id)` | Bulk fill cuboid |
| `replace_blocks(level, x1,y1,z1, x2,y2,z2, old_id, new_id)` | Bulk replace |
| `fill_sphere(level, cx,cy,cz, radius, block_id)` | Fill sphere |
| `get_entities_in_box(level, x1,y1,z1, x2,y2,z2)` | AABB query |
| `raycast(level, entity, sx,sy,sz, dx,dy,dz, max_dist)` | Raycast |
| `raycast_from_entity(entity, max_distance)` | Raycast from entity view |
| `get_positions(entities)` | Batch position extraction |
| `get_healths(entities)` | Batch health extraction |

### `minecraft_py.persistent`

| Class | Description |
|---|---|
| `Data(namespace, default_data={})` | Auto-saving dict storage (`.load()`, `.save()`) |

### `minecraft_py.world`

| Function | Description |
|---|---|
| `set_block(level, pos, mod_id, block_name)` | Place block |
| `get_block(level, pos)` | Get block ID |
| `create_explosion(level, pos, power)` | Explosion |
| `strike_lightning(level, pos)` | Lightning bolt |
| `spawn_particle(level, name, x,y,z, count, speed, mod_id)` | Particles |
| `play_sound(level, name, x,y,z, volume, pitch)` | Play sound |

### `minecraft_py.player`

| Function | Description |
|---|---|
| `give_item(player, mod_id, item_name, count)` | Give items |
| `heal(player, amount)` | Heal player |
| `get_enchantment_level(player, mod_id, enchant_name)` | Check enchantment |

### `minecraft_py.commands`

| Function | Description |
|---|---|
| `commands.register(name, func)` | Register `/name` command (`func(level, player)`) |

### `minecraft_py.hooks`

| Decorator | Description |
|---|---|
| `@hooks.on_entity_tick` | Hook into ALL entity ticks (via `UniversalEntityMixin`) |

### `minecraft_py.bus`

| Function | Description |
|---|---|
| `bus.on(channel, callback)` | Subscribe to event channel |
| `bus.emit(channel, data)` | Broadcast to channel |

### `minecraft_py.mods`

| Function | Description |
|---|---|
| `mods.is_loaded(mod_id)` | Check if a mod is loaded |
| `mods.get_class(class_path)` | Get a Java class from any mod |

### `minecraft_py.native`

| Function | Description |
|---|---|
| `native.run(code, variables=None)` | Execute in native CPython sidecar |

### `minecraft_py.jit`

| Decorator | Description |
|---|---|
| `@jit_math` | Compile Python math → JVM bytecode |

---

> [!TIP]
> **Need help?** Use `/py eval` to test code live in-game, `/py reload` to iterate without restarts, and `python -m minecraft_py stubs` to get IDE autocomplete. Happy modding! 🐍⛏️
