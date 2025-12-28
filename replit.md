# Discord Currency Bot (yapcord dealer)

## Overview

yapcord dealer is a feature-rich Discord bot that creates an interactive economy system for Discord servers. The bot enables server engagement through a comprehensive currency system where users earn by chatting, completing quests, playing games, and participating in social features. It includes a customizable shop, lootbox system, achievements, battle pass progression, VIP memberships, and a unique "Dismegle" random chat matching feature inspired by Omegle.

## User Preferences

Preferred communication style: Simple, everyday language.

## System Architecture

### Core Architecture Pattern

The application follows a **modular service-oriented architecture** with clear separation of concerns:

- **Entry Point (`src/index.ts`)**: Initializes the Discord.js client, manages bot lifecycle, coordinates event handling, and routes commands between services
- **Data Layer (`src/dataManager.ts`)**: Handles all data persistence operations with PostgreSQL storage for all data structures
- **Command Layer (`src/commands.ts`)**: Processes user commands (prefix-based using `?`), implements business logic for all bot features, handles interactive UI elements (buttons, select menus)
- **Type Definitions (`src/types.ts`)**: Provides comprehensive TypeScript interfaces ensuring type safety across the application
- **Database Migration (`src/migrate-to-db.ts`)**: One-time migration tool to transition from JSON storage to PostgreSQL
- **Database Schema (`shared/schema.ts`)**: Drizzle ORM schema definitions for PostgreSQL tables

### Data Storage Strategy

The application uses **PostgreSQL with Drizzle ORM** for all data persistence:

- **Primary Storage: PostgreSQL**
  - Schema fully defined in `shared/schema.ts` with 15+ tables
  - Drizzle Kit configuration in `drizzle.config.ts`
  - Advantages: Scalability, concurrent access, complex queries, data integrity
  - Database URL configured via environment variable
  - Single source of truth - all data flows through PostgreSQL

**Migration History**: Previously used JSON file storage but transitioned to PostgreSQL-only architecture (November 22, 2025) to eliminate data loss issues and provide reliable single source of truth.

**Hourly Backups** (added November 22, 2025):
- Complete data exports to JSON files every hour
- Backups stored in `data/backups/` directory with timestamps
- Files use format: `backup_YYYY-MM-DD_HH-MM-SS.json`
- Automatic cleanup: Keeps only the last 72 hourly backups (~3 days)
- **Persists after publishing**: Backups saved to project root, not `dist/`, so they survive republishing
- Immediate backup on bot startup, then every 60 minutes thereafter
- **Backup Recovery Commands**:
  - `?backups` - Shows all available backups with IST timestamps and file sizes
  - `?restore <filename>` - Restores data from a specific backup with confirmation prompt

### Economy System Design

The bot implements a **multi-tiered economy** with diverse earning mechanisms:

1. **Message-Based Earnings**: Configurable currency per message (default 100, range 1-1,000,000), multiplied by active boost multiplier - **ONLY SOURCE AFFECTED BY BOOSTS**
   - **Customizable with Duration Support**: Server owners can set base earnings with optional expiry
   - `?base set 500` → 500 currency/message permanently
   - `?base set 500 1d` → 500 currency/message for 24 hours, then auto-expires
   - Duration formats: `1s`, `30m`, `2h`, `1d` (seconds, minutes, hours, days)
   - When temporary base expires, automatically reverts to permanent base setting
2. **Passive Income**: Automatic 1 currency/second for all users (2/second with upgrade purchase) - **NOT AFFECTED BY BOOSTS**
3. **AFK Mode**: Enhanced passive earning at 5 currency/second when user activates AFK status - **NOT AFFECTED BY BOOSTS**
4. **Time-Based Rewards**: Daily (streaks supported), hourly, weekly claims with bonus rewards - **NOT AFFECTED BY BOOSTS**
5. **Activity Streak**: Progressive rewards for consecutive daily activity - **NOT AFFECTED BY BOOSTS**
6. **Boost System**: Single active boost per user (higher boosts automatically replace lower ones) - **APPLIES ONLY TO MESSAGE EARNINGS** (fixed November 19, 2025)
7. **Milestone Notifications**: Users receive automatic pings every 100 messages with balance and stats

**Boost Restriction Design** (implemented November 19, 2025):
- Boosts now **only multiply message-based earnings** (chat earnings)
- All other income sources (minigames, gifts, codes, passive income, time rewards) use raw currency values
- This prevents boost exploitation through gifting/codes and maintains game balance
- Internal implementation: `addCurrency()` applies boosts (chat only), `addCurrencyRaw()` for everything else

**Key Economic Features**:
- **Shop System**: Default items plus server-customizable items with one-time purchase controls
- **Inventory Management**: Track owned items, consumption system via `?use` command
- **Lootbox System**: Server administrators create custom lootboxes with configurable drop rates
- **Trading System**: Peer-to-peer trading with acceptance/rejection flow and trade locks
- **VIP Memberships**: Tiered VIP system with expiration tracking
- **Staff Shop System**: Role-based promotion purchases with cooldown enforcement (added November 19, 2025)
- **Token Shop System**: Rotating shop with 5 special tokens (Timeout, Nickname, Slowmode, Voice Mute, Role Color) that restock hourly with per-token hourly purchase limits (updated November 20, 2025)

### Command System Architecture

The bot uses a **prefix-based command system** (`?` prefix):

- **Fuzzy Matching**: Intelligent command and item name matching for user convenience (e.g., "2x boost" matches "2x Money Boost")
- **Permission Layers**: Server owner, manager delegation system, regular users
- **Interactive UI**: Extensive use of Discord embeds with pagination, buttons, and select menus
- **Response Caching**: Prevents duplicate responses within 5-second window

### Game and Social Features

**Gambling/Games** (expanded November 22, 2025 - 20 new minigames):
- **Core Games**: Coinflip, scratch cards, slots (3-reel weighted slot machine), blackjack (dealer vs player), lottery (ticket-based instant draw)
- **PvP Challenges**: Arena duels, rock-paper-scissors challenges with wagering
- **Adventure Games**: Trivia showdown, treasure hunts, heist missions, mining with level progression
- **New Minigames** (added November 22, 2025):
  - **Classic Strategy**: Rock Paper Scissors (`?rps`), Hi-Lo Card Game (`?hilo`), Horse Racing (`?horserace`)
  - **Luck-Based**: Envelope Opening (`?envelope`), Goldfish Game (`?goldfish`), Coin Pile (`?coinpile`)
  - **Risk/Reward**: Double Or Nothing (`?doubletrouble`), Daredevil (`?daredevil`), Bomb Squad (`?bombsquad`)
  - **Progressive**: King of the Hill (`?kingofhill`), Jackpot Machine (`?jackpot`), Slot Machine (`?slotmachine`)
  - **Number Games**: Quick Pick (`?quickpick`), Roulette (`?roulette`), Cash Cow (`?cashcow`)
  - **Pattern Matching**: Trivia Game (`?trivia`), Minesweeper (`?minesweeper`), Fortune Wheel (`?fortune`)
- **Balanced Payouts**: Default bets range from 1,000-5,000 currency with multipliers from 1.5x-36x based on difficulty/rarity
- All gambling/game payouts **do not apply boost multipliers** to maintain fair gameplay and prevent economy exploitation

**Progression Systems**:
- Achievement tracking with rewards
- Battle pass with XP and level progression
- Daily quest system with rotation
- Title system for user customization

**Social Features**:
- **Dismegle System**: Omegle-style random chat matching between users with interest-based matching, text and video chat modes, user blocking functionality
- Trading between users with item and currency exchange
- Leaderboard tracking for competitive engagement
- Gift/transfer system between users

### Administrative Controls

**Server Owner Capabilities**:
- **Customizable Message Base Earnings**: Set base earnings (1-1,000,000 currency/message) with optional temporary duration
  - Permanent: `?base set 500`
  - Temporary (expires automatically): `?base set 500 1d` for 24-hour duration
  - View status: `?base` shows current base and remaining time if temporary
- Dynamic pricing for boost items
- Enable/disable gambling features per server
- Server data reset functionality
- User banning/unbanning from economy
- Manager role delegation for administrative commands
- Staff shop configuration with `?setstaff` command

**Data Management** (updated November 19, 2025):
- Inventory clearing for specific users (now properly resets boost and passive income states)
- Trade lock enforcement
- Custom shop item creation
- Lootbox creation and configuration
- **Clear Inventory Fix**: `?clearinventory` now resets balance, inventory, boost multiplier (to 1x), and purchased items to prevent orphaned state bugs

**Staff Shop System** (added November 19, 2025):
- **Configuration**: 
  - `?setstaff <role>` - Designate staff role (server owner only)
  - `?setpromotionprice <amount>` - Set base promotion price (server owner only)
  - `?setpromotioncooldown <days>` - Set cooldown duration in days (server owner only)
- **Access Control**: Only users with staff role (or server owner) can access staff shop
- **Dynamic Pricing System**: 
  - Base price set by server owner (default: 50,000 currency)
  - Price doubles after each purchase: `basePrice * 2^totalPurchases`
  - Example: 50k → 100k → 200k → 400k → 800k
  - Resets to base price on server reset
- **Promotion System**: 
  - **Purchase**: `?buy promotion` adds promotion to inventory
  - **Activation**: `?use promotion` grants the next role and removes from inventory
  - **Cooldown**: Configurable days between purchases per user (default: 5 days)
  - **Effect**: Automatically assigns next higher role in server hierarchy
  - **Gift Command**: `?giftpromotion <@user>` allows server owner to gift promotions (no cooldown)
- **Role Progression**: Automatically identifies next higher role based on Discord role position hierarchy
- **Safety Checks**: Validates bot permissions before role assignment to prevent errors
- **Transparency**: `?staffshop` displays base price, total purchases, current price formula, and cooldown settings

### Technical Implementation Choices

**Technology Stack**:
- **Node.js with TypeScript**: Type safety, modern JavaScript features, strong IDE support
- **Discord.js v14**: Latest stable Discord API library with full feature support
- **Drizzle ORM**: Type-safe SQL query builder for PostgreSQL migration

**Design Patterns**:
- **Singleton DataManager**: Central data access point with caching
- **Command Handler Pattern**: Centralized command routing and validation
- **Event-Driven Architecture**: Leverages Discord.js event system for message processing
- **Lazy Loading**: Data loaded on-demand per guild/user combination

**Error Handling**:
- Graceful degradation for missing data
- User-friendly error messages with emoji indicators
- Silent failure prevention with comprehensive logging

## External Dependencies

### Primary Dependencies

- **Discord.js v14.25.0**: Core Discord API interaction library providing gateway connections, event handling, message processing, embed builders, and component interactions
- **Node.js**: Runtime environment (ES2022 target)
- **TypeScript v5.3.3+**: Primary development language for type safety

### Database Layer

- **PostgreSQL**: Relational database for production data storage (configured, not yet active)
- **Drizzle ORM v0.44.7**: Type-safe ORM for database operations and migrations
- **Drizzle Kit v0.31.7**: Database migration and schema management tool
- **pg v8.16.3**: PostgreSQL client for Node.js

### Utility Libraries

- **axios v1.13.2**: HTTP client for external API calls (trivia questions, external data fetching)

### Development Tools

- **@types/node**: TypeScript definitions for Node.js
- **@types/pg**: TypeScript definitions for PostgreSQL client

### Environment Variables Required

- `DISCORD_BOT_TOKEN`: Discord bot authentication token (required)
- `DATABASE_URL`: PostgreSQL connection string (optional, for database migration)

### External APIs

- **Discord API**: All bot functionality relies on Discord's REST API and Gateway API
- **Trivia API** (implicit): Axios suggests external API calls for trivia question fetching