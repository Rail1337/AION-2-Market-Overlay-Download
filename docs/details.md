# Details

The longer explanations behind the [README](../README.md).

## Screen areas and equipment

The areas are stored as fractions of the game window, so they follow the window when it moves. Without a setup the
program uses built-in areas that were measured on real screenshots at **21:9 (2560x1080)** and **16:9 (2560x1440)**;
the one nearest to the shape of your window is taken, at any resolution of that shape. The settings window tells you
if your window has another shape (16:10, 4:3 ...); then run the setup once. After a setup your own areas count, also
the values for `regions` in `config.json` (they are only read after a setup).

**Equipment:** weapons, armour and accessories have a long detail panel. For them the box sits lower (at least at
`overlay.tall_min_y`, default 0.61 of the window height), so it does not cover the stats; materials get it right below
the short panel.

## Settings window

- Status line: game found, market open, which item.
- **Server**: pick region and server. The default is Kaisinel (Europe). The list comes from the site.
- **Show the info box**, **Show the price graph**, **Show the flip advice**, **Only while the game is in front** (the box is hidden while another program is in front of
  the game). If the box never shows up in the game but the status line says the market is open, the game window may
  not report itself as being in front: switch this off.
- **Reload data** fetches the item list and the prices again.

Everything is saved automatically in `config.json` next to the program. More values can be edited there (all with
checks; a wrong value falls back to the default and is noted in `logs/overlay.log`):

| Key | Meaning | Default |
|---|---|---|
| `server_id` | the server (see `/api/servers` on the site) | 11304 (Kaisinel, Europe) |
| `game_window_title` | the window to watch | `AION2` |
| `poll_ms` | how often the screen is read | 250 |
| `name_confirmations` | equal readings in a row before an item counts | 2 |
| `closed_confirmations` | looks without the market title before it counts as closed | 3 |
| `hold_polls` | how long an unmatched name is held before "No market data" | 8 |
| `match_threshold`, `match_margin`, `unknown_below` | how sure a name match must be | 0.86, 0.04, 0.6 |
| `price_cache_s` | how long a price is reused | 300 |
| `overlay.offset` | move the box by a fraction of the window | [0, 0] |
| `overlay.tall_min_y` | how low the box goes for equipment (fraction of the window height) | 0.61 |
| `show_graph` | the price graph below the box | true |
| `show_flip` | the flip advice in the box | true |
| `flip.listing_fee`, `flip.sale_fee`, `flip.min_profit` | the market's fees and the profit that makes a good deal | 0.02, 0.10, 0.05 |
| `rarity_colors` | own name colours, e.g. `{"rare": "#4cd964"}` for common, rare, epic, unique, heroic, special | {} |
| `overlay.opacity` | see-through level of the box | 0.92 |

## How it works

1. The game window is found by its title and its drawing area is measured.
2. Two small areas are cut out of it (market title, item name), enlarged for the game's small text, and read with
   Windows' own text recognition (offline, about 10 ms).
3. The name is matched against the 836 items of the site. The match is careful on purpose: many names are almost
   alike ("... Gloves" and "... Greaves"), so a name is only accepted if it clearly beats every other item. In a test
   with 2861 deliberate misreadings it never picked a wrong item. A name it does not know shows "No market data".
4. Prices come from `https://aion2.exchange/api` (public, read-only, no key). The grade of the item is read from the
   detail panel (the first word under the name), the graph comes from the site's `sparklines` endpoint (one more
   request per selected item, reused for five minutes like the prices). They are a snapshot (the site collects
   every few minutes, the sales statistics once a day), so the box is a reference, not a live price.

The program keeps the number of requests low: the item list is fetched once a day, prices are reused for five minutes
and only the selected item is asked for.

## The flip advice

Buying an item cheap and selling it again costs two fees on this market: 2 % of the price when you list it and 10 % off
when it sells, so only 88 % of a sale is left. The box shows two lines:

```
Flip: buy under 300,758
Resell ~358,860 · now +10.9 % after fees
```

- **Resell** is the typical lowest price of the last 24 hours (the median of the hourly cheapest prices), the price you
  could expect when listing the item again.
- **Buy under** is the highest price that still leaves 5 % profit after the fees: `resell x 0.88 / 1.05`. It is green
  when the cheapest listing now is under it, plain when it still leaves a little, red when buying at it would lose
  money. "now +10.9 %" is the profit if you bought at the cheapest price now and sold at the resell price.
- It is a rule of thumb: the price can move, a sale can take time, and the listing fee is gone if the item does not
  sell. Fees and goal can be changed in `config.json` (`flip`).

## Kina chests (Quna market)

The market also sells **Kina chests** for Quna ("Kina Chest: 10,000,000 Kina" and so on). The program treats each of
them as an item of its own, priced in Quna, with everything an item has: the price and the number of listings, the
change over 24 hours and the small price graph with its axes. Instead of the flip advice (there is nothing to flip:
it is an exchange of Kina for Quna, which costs the fees whatever the price) the box says whether **now is a good
moment** to sell.

```
Kina Chest: 10,000,000 Kina
Now 3,898 Quna · 43 listed
−3.1 % (24 h) · 2,565 Kina per Quna
Exchange: a good moment · +4.6 %
~3,430 Quna after 12 % fees · usual 3,725
```

- The numbers come from the site's Quna market (`/api/quna`), the price history from `/api/quna/history`. The site has
  only recorded the chests since 13 October, so the graph says "since 13 Oct" until a week of history is there, and
  the flip advice appears once there are six hourly prices from the last 24 hours.
- **Exchange line:** the price of the chest now against its usual price, the median of the last 24 hours. Green when it
  is 2 % or more better than usual, red when it is 2 % or more worse, plain in between. The second line shows what is
  left of the price after the two fees (2 % for listing, 10 % when it sells, 12 % together) and the usual price. The
  fees are lost in every case, the line only tells whether the moment is better or worse than usual. The band and
  the fees are in `config.json` (`flip.exchange_band`, `flip.listing_fee`, `flip.sale_fee`).
- There are no sales statistics for chests, so the "Sold" lines are left out.
- The text recognition sometimes loses the digits in the title at the normal size ("Kina Chest: Kina"). When a name
  finds no item but is close to one, the picture is read again at other sizes (4x, 2.5x, 1.5x, 8x) until one finds the
  amount; which size works differs from chest to chest (the 2 million chest comes out at 2.5x, not at 4x). Every extra
  read is written to the log.

## Privacy

The program reads the screen only in memory and never saves a picture. The log never contains picture content. Settings, the
cached item list and the log stay on your computer; the only requests go to aion2.exchange for the item list and prices.
