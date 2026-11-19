# claude-skill-scalping

A Claude Code skill for running and monitoring XAUUSD scalping on a **MetaTrader 5 demo account** through a MetaTrader MCP server (M1/M5/M15), with layered 0.50-lot entries, broker-side SL/TP and a recurring 3-minute monitor. Replies to the user are in Bahasa Melayu.

## Requirements

- [Claude Code](https://claude.com/claude-code) (CLI or desktop app)
- A MetaTrader MCP server connected to Claude Code, exposing tools such as `get_account_info`, `get_positions_by_symbol`, `get_candles_latest`, `get_symbol_price`, `place_market_order`, `modify_position`, `close_position`
- MetaTrader 5 running and logged in to a **demo** account (the skill refuses to trade if `account_type` is not `demo`)
- Optional: an `smc` analysis skill for multi-timeframe zone mapping

## Install

```bash
git clone https://github.com/aQie20/claude-skill-scalping.git
cp -r claude-skill-scalping/scalping ~/.claude/skills/
```

On Windows (PowerShell):

```powershell
git clone https://github.com/aQie20/claude-skill-scalping.git
Copy-Item -Recurse claude-skill-scalping\scalping $HOME\.claude\skills\
```

Restart Claude Code so the skill is picked up. Check the broker's gold symbol name (this skill uses `XAUUSD.vx`; change it in `SKILL.md` if yours differs, e.g. `XAUUSD`).

## Cara guna (How to use)

### 1. Mula sesi (start a session)

Taip dalam Claude Code:

```
/scalping analisis zon baru hari ini dan pantau trade
```

Claude akan:
1. Semak akaun ialah **demo** (jika bukan, ia berhenti).
2. Bina peta zon (H4 → H1 → M15): sokongan, rintangan, julat, setup BUY/SELL dengan harga tepat.
3. Cipta semakan berulang **setiap 3 minit** (cron) yang membawa peraturan + peta zon.

### 2. Setiap semakan (each tick)

Claude semak akaun → posisi → candle M1/M5/M15 → harga live, kemudian lapor 2–4 baris:
harga, P/L, status, dan tindakan (atau "Tiada entry" + sebab). Ia hanya masuk bila **semua** syarat dipenuhi:

- Pecahan sah: M5/M15 **tutup jelas** (≥ $0.5) di luar zon — bukan sumbu.
- Tunggu **retest** paras yang pecah; jangan kejar candle pecahan.
- Isyarat = candle M1 **tutup**: BUY = higher low + tutup atas high sebelumnya; SELL = lower high + tutup bawah low sebelumnya.
- SL atas/bawah high/low retest sebenar + 0.5; TP **sebelum** halangan seterusnya.
- R:R ≥ 1.0 dikira dari harga live (dan harga isi yang dijangka).
- Pasaran tenang: sumbu M1 lebih kecil dari SL. Pasaran choppy → langkau.

Selepas order terisi, SL/TP dipasang serta-merta; jika gagal, posisi ditutup.

### 3. Arahan biasa (common commands)

| Taip | Hasil |
|---|---|
| `/scalping pantau trade` | Mula / sambung pemantauan 3 minit |
| `skrg ada peluang buy?` | Analisis segera arah tertentu |
| `kenapa tak entry?` / `kenapa loss?` | Penjelasan dengan harga struktur sebenar + pelajaran |
| `tutup posisi` | Tutup posisi milik skill |
| `stop pantau` | Hentikan cron pemantauan |
| Ubah peraturan, cth. `maks risiko $300` | Arahan terkini menggantikan default |

### 4. Peraturan default (boleh diubah)

| Peraturan | Nilai |
|---|---|
| Lot | 0.50 setiap layer, 2–4 layer |
| Risiko | maks $500 setiap trade; SL > $5 dengan 2 layer → langkau |
| R:R minimum | 1:1 |
| Arah | jangan BUY & SELL serentak |
| Henti | 3 kalah berturut dalam satu sesi → berhenti & lapor |
| Trailing | ditangguh semasa ujian EA trailing (SL/TP tidak dialih) |

### 5. Jurnal

Setiap trade dicatat dalam fail jurnal (cth. `scalping-journal.md` dalam folder projek): masa, arah, layer, entry, SL, TP, thesis, keputusan, pelajaran. Pelajaran baharu turut dimasukkan semula ke dalam `SKILL.md`.

## Limitations

- Polling every 3 minutes misses fast signals that start and finish between checks; the skill reports these honestly instead of chasing them.
- Market-order fills can slip in fast markets; the skill re-checks the live price before ordering.
- The cron monitor is session-only (expires after 7 days or when the session ends). Broker-side SL/TP still protect open trades.
- Avoid trading around the broker's daily close (wide spread, thin liquidity).

## Disclaimer

Educational / demo use only. Not financial advice. The strategy has no backtested edge; a few days of demo results prove nothing.
