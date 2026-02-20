# Cockatrice Codebase Guide

Cockatrice is a cross-platform, open-source virtual tabletop for Magic: The Gathering. C++20, Qt 5/6, Protocol Buffers 3.21+, CMake 3.10+. Version 2.11.0 ("Omenpath"). GPLv2 license.

Three applications share 12 libraries:
- **Cockatrice** - Desktop game client (GUI)
- **Servatrice** - Multiplayer game server
- **Oracle** - Card database importer (Scryfall)

---

## Directory Map

| Directory | Purpose |
|-----------|---------|
| `cockatrice/` | Desktop client application |
| `servatrice/` | Game server application |
| `oracle/` | Card database import tool |
| `webclient/` | Web-based client (TypeScript) |
| `libcockatrice_card/` | Card models, database, parsers |
| `libcockatrice_deck_list/` | Deck data structure, serialization, undo/redo |
| `libcockatrice_protocol/` | Protobuf definitions (150+ .proto files) |
| `libcockatrice_network/` | Client/server networking, server-side game logic |
| `libcockatrice_models/` | Qt Model/View classes for UI |
| `libcockatrice_settings/` | User preferences, card preferences |
| `libcockatrice_filters/` | Card filtering and search syntax |
| `libcockatrice_interfaces/` | Abstract interface definitions |
| `libcockatrice_utility/` | Shared utilities (CardRef, etc.) |
| `libcockatrice_rng/` | SFMT Mersenne Twister RNG |
| `tests/` | Google Test suite |
| `cmake/` | CMake modules, templates, packaging |
| `.ci/` | CI scripts (compile, release, lint) |
| `.github/workflows/` | GitHub Actions CI/CD (8 workflows) |
| `doc/` | XSD schemas (card DB v3/v4, deck format) |
| `vcpkg/` | Git submodule: Microsoft vcpkg |

---

## Build System

```bash
mkdir build && cd build
cmake .. -DWITH_SERVER=1 -DTEST=1
cmake --build .
```

| Flag | Default | Purpose |
|------|---------|---------|
| `WITH_CLIENT` | ON | Build Cockatrice client |
| `WITH_SERVER` | OFF | Build Servatrice server |
| `WITH_ORACLE` | ON | Build Oracle tool |
| `TEST` | OFF | Build Google Test suite |
| `FORCE_USE_QT5` | OFF | Skip Qt6, use Qt5 |
| `USE_VCPKG` | auto (ON on Windows) | Use vcpkg for dependencies |
| `UPDATE_TRANSLATIONS` | OFF | Regenerate .ts translation files |

Dependencies: Qt, Protobuf, OpenSSL, zlib, liblzma (optional), MySQL client (server only). Managed via `vcpkg.json`.

---

## Library Dependency Graph

```
Applications
├── cockatrice  → card, deck_list, network, models, settings, filters, rng
├── servatrice  → network, protocol, card, deck_list, rng
└── oracle      → card, models, settings

Libraries
├── libcockatrice_network   → protocol, card, deck_list
├── libcockatrice_models    → card, deck_list
├── libcockatrice_filters   → card
├── libcockatrice_card      → interfaces
├── libcockatrice_settings  → interfaces
├── libcockatrice_deck_list → utility
├── libcockatrice_protocol  → (protobuf)
├── libcockatrice_rng       → (standalone)
├── libcockatrice_interfaces→ (standalone)
└── libcockatrice_utility   → (standalone)
```

---

## Cockatrice Client

### Entry Point and Globals

| File | Key Contents |
|------|-------------|
| `cockatrice/src/main.cpp` | App init: RNG, theme, sound, translations, card DB, MainWindow |
| `cockatrice/src/main.h` | Global pointers: `translator`, `rng`, `soundEngine`, `trayIcon`, `themeManager` |

### Window and Tab Architecture

| Class | File | Purpose |
|-------|------|---------|
| `MainWindow` | `cockatrice/src/interface/window_main.h` | Root QMainWindow: menus, toolbar, status bar |
| `TabSupervisor` | `cockatrice/src/interface/tab_supervisor.h` | QTabWidget managing all tabs |
| `Tab` (abstract) | `cockatrice/src/interface/widgets/tabs/tab.h` | Base for all tabs (extends QMainWindow) |

Tab types in `cockatrice/src/interface/widgets/tabs/`:

| Tab | Purpose |
|-----|---------|
| `TabGame` | Active game play area |
| `TabRoom` | Game room lobby |
| `TabDeckEditor` | Classic deck editor |
| `TabDeckEditorVisual` | Visual deck editor (VDE) |
| `TabDeckStorage` / `TabVisualDeckStorage` | Deck management |
| `TabHome` | Landing page |
| `TabServer` | Server browser |
| `TabAccount` | User profile |
| `TabAdmin` | Moderation tools |
| `TabReplays` | Game replay viewer |
| `TabLog` | Event/error log |
| `TabMessage` | Direct messages |
| `TabEdhRec` / `TabArchidekt` | External deck recommendations |

### Game Subsystem

Core classes in `cockatrice/src/game/`:

| Class | File | Purpose |
|-------|------|---------|
| `AbstractGame` | `game/abstract_game.h` | Game controller: state, events, players |
| `GameState` | `game/game_state.h` | Phase, active player, timing |
| `GameEventHandler` | `game/game_event_handler.h` | Processes protobuf game events |
| `PlayerManager` | `game/player/player_manager.h` | Player collection management |
| `GameScene` | `game/game_scene.h` | QGraphicsScene for board rendering |
| `GameView` | `game/game_view.h` | QGraphicsView viewport |

Player and card classes:

| Class | Location | Purpose |
|-------|----------|---------|
| `Player` | `game/player/player.h` | Player state, zones, menus |
| `CardItem` | `game/board/card_item.h` | QGraphicsItem for a card (counters, tapped, attacking, etc.) |
| `CardDragItem` | `game/board/card_drag_item.h` | Drag-and-drop support |

Zone architecture (logic and graphics separated):

| Zone Class | Location | Purpose |
|------------|----------|---------|
| `CardZone` (abstract) | `game/zones/card_zone.h` | Graphics base for zones |
| `CardZoneLogic` | `game/zones/card_zone_logic.h` | Business logic base |
| `HandZone` | `game/zones/hand_zone.h` | Player's hand |
| `TableZone` | `game/zones/table_zone.h` | Battlefield |
| `StackZone` | `game/zones/stack_zone.h` | Spell stack |
| `PileZone` | `game/zones/pile_zone.h` | Library, graveyard, exile |
| `ViewZone` | `game/zones/view_zone.h` | Temporary reveal views |

### Deck Editor

Classic editor in `cockatrice/src/interface/widgets/deck_editor/`:
- `DeckEditorDeckDockWidget` - Deck tree view
- `DeckEditorCardDatabaseDockWidget` - Card browser
- `DeckEditorFilterDockWidget` - Advanced filters
- `DeckEditorCardInfoDockWidget` - Card details
- `DeckEditorPrintingSelectorDockWidget` - Printing selection

Visual Deck Editor (VDE) in `cockatrice/src/interface/widgets/tabs/`:
- `TabDeckEditorVisual` - Main VDE tab
- `VisualDeckEditorWidget` - Drag-drop deck editing
- `VisualDatabaseDisplayWidget` - Visual card browser
- `DeckAnalyticsWidget` - Mana curve, stats
- `VisualDeckEditorSampleHandWidget` - Sample hand simulator
- `DeckStateManager` - Tracks modifications

### Card Picture Loading

| Class | File | Purpose |
|-------|------|---------|
| `CardPictureLoader` | `interface/card_picture_loader/card_picture_loader.h` | Singleton image manager |
| `CardPictureLoaderWorker` | (same dir) | Background thread fetcher |
| `CardPictureLoaderStatusBar` | (same dir) | Loading progress UI |

Pipeline: request -> QPixmapCache check -> disk check -> network fetch -> cache -> emit `pixmapLoaded()`

### Settings, Shortcuts, Themes

| Class | File | Purpose |
|-------|------|---------|
| `SettingsCache` | `client/settings/cache_settings.h` | Singleton settings manager |
| `ShortcutsSettings` | `client/settings/shortcuts_settings.h` | 19 groups, 150+ shortcuts |
| `ThemeManager` | `interface/theme_manager.h` | Zone backgrounds, theme switching |
| `SoundEngine` | `client/sound_engine.h` | QMediaPlayer audio, theme packs |

---

## Servatrice Server

### Entry Point

| File | Purpose |
|------|---------|
| `servatrice/src/main.cpp` | Load config, init DB, start TCP/WebSocket listeners |
| `servatrice/src/servatrice.h` | Main `Servatrice` class (extends `Server`) |

### Architecture

| Class | File | Purpose |
|-------|------|---------|
| `Servatrice` | `servatrice/src/servatrice.h` | Server instance, room management |
| `Servatrice_GameServer` | `servatrice/src/servatrice.h` | TCP listener (port 4747) |
| `Servatrice_WebsocketGameServer` | `servatrice/src/servatrice.h` | WebSocket listener (port 4748) |
| `Servatrice_ConnectionPool` | `servatrice/src/servatrice_connection_pool.h` | Thread pool for connections |
| `Servatrice_DatabaseInterface` | `servatrice/src/servatrice_database_interface.h` | MySQL/MariaDB wrapper (schema v34) |

### Server-Side Game Objects

Located in `libcockatrice_network/libcockatrice/network/server/remote/game/`:

| Class | Purpose |
|-------|---------|
| `Server_Game` | Game instance: players, state, replay recording, rules |
| `Server_Card` | Authoritative card state (position, face-down, counters) |
| `Server_CardZone` | Zone container (hand, library, battlefield, etc.) |
| `Server_Arrow` | Targeting/attachment arrows |
| `Server_Counter` | Life, poison, custom counters |
| `Server_AbstractParticipant` | Base for players and spectators |

### Configuration and Database

- Config: `servatrice/servatrice.ini.example` (ports, DB connection, auth, logging)
- Schema: `servatrice/servatrice.sql` (users, decks, games, replays, bans, warnings)
- SMTP: `servatrice/src/smtp/` (account activation, password reset emails)

---

## Oracle Tool

| File | Purpose |
|------|---------|
| `oracle/src/main.cpp` | Entry point |
| `oracle/src/oracleimporter.h` | Scryfall API fetch, card parsing, set priorities |
| `oracle/src/oraclewizard.h` | Wizard UI for guided import |

Set priority: Primary (core/expansion) > Secondary (commander) > Reprint (promo) > Lowest (non-English) > Other (funny/vanguard).

Output: CockatriceXml4 format (`.xml`), loaded by `CardDatabaseLoader`.

---

## Protocol (Protobuf)

Definitions in `libcockatrice_protocol/libcockatrice/protocol/pb/`.

### Message Hierarchy

Top-level: `IslMessage` (in `isl_message.proto`) wraps all communication:

| Payload | Direction | Purpose |
|---------|-----------|---------|
| `CommandContainer` | Client -> Server | Session, room, game, admin commands |
| `Response` | Server -> Client | Command results with ResponseCode |
| `SessionEvent` | Server -> Client | Login, server messages, disconnect |
| `GameEventContainer` | Server -> Client | In-game events |
| `RoomEvent` | Server -> Client | Room/lobby events |

### Command Categories

- **Session**: Login, Register, Activate, ForgotPassword, ListUsers
- **Room**: JoinRoom, LeaveRoom, ListGames, CreateGame, JoinGame
- **Game**: MoveCard, DrawCards, CreateToken, Tap/Untap, AddCounter, SetCardAttribute, Mulligan
- **Admin**: KickFromGame, BanUser, AdjustMod

### Key Event Types (`event_*.proto`)

Game: `MoveCard`, `DrawCards`, `CreateToken`, `SetCardAttr`, `CreateCounter`, `CreateArrow`, `RollDie`, `GameSay`, `PlayerPropertiesChanged`, `GameStateChanged`

Room: `UserJoined`, `UserLeft`, `ListGames`, `RoomSay`

Session: `ServerIdentification`, `ServerMessage`, `ConnectionClosed`

### ServerInfo Structures (`serverinfo_*.proto`)

`ServerInfo_Card`, `ServerInfo_Player`, `ServerInfo_Zone`, `ServerInfo_Counter`, `ServerInfo_Game`, `ServerInfo_User`, `ServerInfo_Arrow`, `ServerInfo_Replay`

---

## Key Data Models

### Card System (`libcockatrice_card/libcockatrice/card/`)

| Class | File | Purpose |
|-------|------|---------|
| `CardInfo` | `card_info.h` | Full card: name, text, properties (QVariantHash), printings, relations, UI attrs |
| `CardSet` | `card_set.h` | Set metadata: code, name, release date, priority |
| `PrintingInfo` | `printing_info.h` | Card variant in a set: collector number, art, flavor |
| `CardRelation` | `card_relation.h` | Links between related cards (DFC, companions) |
| `CardDatabase` | `database/card_database.h` | Thread-safe index: by name, by simplified name. Mutex-protected |
| `CardDatabaseLoader` | `database/card_database_loader.h` | Discovers and loads .xml files |
| `CardDatabaseQuerier` | `database/card_database_querier.h` | Query interface: by name, fuzzy, by set, filtered |
| `CockatriceXml4Parser` | `database/cockatrice_xml4_parser.h` | Current XML format parser |
| `CockatriceXml3Parser` | `database/cockatrice_xml3_parser.h` | Legacy XML format parser |

### Deck System (`libcockatrice_deck_list/libcockatrice/deck_list/`)

| Class | File | Purpose |
|-------|------|---------|
| `DeckList` | `deck_list.h` | Top-level deck: metadata, tree root, sideboard plans |
| `DecklistNodeTree` | `deck_list_node_tree.h` | Tree: zones -> groups -> cards |
| `InnerDecklistNode` | (same) | Interior tree node |
| `DecklistCardNode` | (same) | Leaf: card name + quantity |
| `SideboardPlan` | `sideboard_plan.h` | Named sideboard strategy (cards in/out) |
| `DeckListHistoryManager` | `deck_list_history_manager.h` | Undo/redo (Memento pattern) |

Tree structure: Root -> Zone nodes (Main Deck, Sideboard) -> Group nodes (Creatures, Instants) -> Card nodes

### Qt Models (`libcockatrice_models/libcockatrice/models/`)

`CardDatabaseModel`, `CardSearchModel`, `CardSetModel`, `DeckListModel`, `TokenDisplayModel`

---

## Coding Patterns

| Pattern | Convention |
|---------|-----------|
| Class names | CamelCase: `CardItem`, `TabDeckEditor`, `PlayerManager` |
| Method names | camelCase: `getZone()`, `setAttacking()`, `loadFromFile()` |
| File names | snake_case: `card_item.h`, `game_scene.cpp` |
| Member vars | camelCase (no prefix) |
| Constants | ALL_CAPS or camelCase depending on scope |
| Logging | `Q_LOGGING_CATEGORY` macros, `qCInfo`/`qCWarning`/`qCDebug` |
| Memory | Qt parent-child ownership, `QPointer`, `deleteLater()` |
| Signals/slots | Pervasive Qt signal/slot with `Q_OBJECT` macro |
| Include guards | `#ifndef CLASSNAME_H` / `#define CLASSNAME_H` |

Design patterns used: Singleton (SettingsCache, CardDatabase), Model-View (Qt MVC), Observer (signals/slots), Command (protobuf commands), Memento (deck undo/redo), Factory (parsers, client types), Thread Pool (Servatrice connection pools).

---

## Testing

- Framework: Google Test (auto-downloaded if missing)
- Location: `tests/`
- Build: `cmake .. -DTEST=1`
- Tests: card database, deck loading, clipboard import, filter expressions, password hashing, age formatting

---

## Client Connection States

```
Disconnected -> Connecting -> GettingPasswordSalt -> LoggingIn -> LoggedIn
                           -> Registering -> Activating -> LoggedIn
```

Offline play uses `LocalClient` + `LocalServer` (in-process, no network).

---

## Docker (Server Only)

```bash
docker-compose up  # Starts Servatrice + MySQL
```

Config in `docker-compose.yml`, base image Ubuntu 24.04, builds only server.
