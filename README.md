"""
Timmy's Concept Scorer - v1.3  "CALM EDITION"

What's new vs v1.2
  1. MARKET WEATHER: reads BTC, ETH and SOL 24h moves (Binance public data, CoinGecko as backup).
     Red markets mean thin trading and more room for bots, so the rules get STRICTER:
         GREEN / NEUTRAL / RED / DEEP RED  ->  higher liquidity floor, higher score bar,
         fewer paper trades allowed. On a DEEP RED day the tool says: stand down.
  2. DRAWDOWN-FROM-PEAK: rejects coins already far below their 48h high (spike-then-bleed,
     like the REA chart you sent).
  3. BOT / WALLET-PATTERN CHECK from the last 300 trades: a few wallets driving the volume,
     or wallets doing hundreds of buy-sell round trips with ~zero net (volume-bot behaviour).
  4. A CALM report: counts the traps you did NOT walk into, explains the weather in plain
     English, and ends with permission to do nothing. Skipping is a valid trade.

Run:
    python concept_scorer_v1_3.py                     live scan + spot checks
    python concept_scorer_v1_3.py --recheck           update paper trades with current prices
    python concept_scorer_v1_3.py --demo              fake data, no internet needed
    python concept_scorer_v1_3.py --demo --weather=RED   (GREEN / NEUTRAL / RED / DEEP_RED)

Optional environment variables:  SOLANA_RPC_URL, RUGCHECK_API_KEY

This is research + paper-trading software, not financial advice. WATCH does NOT mean safe.
GeckoTerminal's free API is rate-limited, so the spot checks pause between calls on purpose.
"""
import csv
import json
import os
import sys
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone

DEX = "https://api.dexscreener.com"
GECKO = "https://api.geckoterminal.com/api/v2"
RUG = "https://api.rugcheck.xyz/v1"
BINANCE = "https://api.binance.com/api/v3/ticker/24hr"
COINGECKO = "https://api.coingecko.com/api/v3/simple/price"
RPC_URL = os.environ.get("SOLANA_RPC_URL", "https://api.mainnet-beta.solana.com")
RUG_KEY = os.environ.get("RUGCHECK_API_KEY")

LOG_FILE = "concept_log_v1_3.csv"
PAPER_FILE = "paper_trades.csv"      # same file as v1.2, so your old paper trades carry over

# ---- tunable rules (change as YOUR data teaches you) ----
BASE_MIN_LIQUIDITY = 15_000
MIN_LIQ_TO_MCAP = 0.05
MIN_AGE_HOURS = 0.5
VOL_MCAP_WARN, VOL_MCAP_FAIL = 10, 30
COLLAPSE_1H, COLLAPSE_24H = -25, -50
MAX_TOP10_PCT, WARN_TOP10_PCT = 35, 25
DRAWDOWN_FAIL, DRAWDOWN_WARN = 60, 40           # % below 48h peak
TOP_WALLET_FAIL, TOP_WALLET_WARN = 35, 20       # % of recent trade volume from ONE wallet
BOT_VOLUME_FAIL = 30                            # % of volume from round-trip wallets
DISTINCT_FAIL, DISTINCT_WARN = 0.15, 0.30       # unique wallets / trades
SPOT_CHECK_TOP_N = 5
GECKO_PAUSE = 6.5                               # seconds between GeckoTerminal calls
WEIGHTS = {"narrative": 1.0, "meme": 1.0, "community": 1.5, "timing": 1.0, "value": 1.0}

# weather -> how strict today's rules are
WEATHER_RULES = {
    "GREEN":    {"liq_mult": 1.0,  "min_score": 50, "max_watch": 3},
    "NEUTRAL":  {"liq_mult": 1.0,  "min_score": 55, "max_watch": 2},
    "RED":      {"liq_mult": 1.5,  "min_score": 65, "max_watch": 1},
    "DEEP_RED": {"liq_mult": 2.0,  "min_score": 75, "max_watch": 0},
    "UNKNOWN":  {"liq_mult": 1.25, "min_score": 60, "max_watch": 2},
}


# ======================= helpers =======================
def get_json(url, headers=None, retry_429=True):
    h = {"User-Agent": "concept-scorer/1.3", "Accept": "application/json"}
    if headers:
        h.update(headers)
    try:
        with urllib.request.urlopen(urllib.request.Request(url, headers=h), timeout=20) as r:
            return json.load(r)
    except urllib.error.HTTPError as e:
        if e.code == 429 and retry_429:
            print("  (rate limited - pausing 30s, then trying once more)")
            time.sleep(30)
            return get_json(url, headers, retry_429=False)
        raise


def rpc(method, params):
    body = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    req = urllib.request.Request(RPC_URL, data=body, headers={
        "Content-Type": "application/json", "User-Agent": "concept-scorer/1.3"})
    with urllib.request.urlopen(req, timeout=20) as r:
        out = json.load(r)
    if "error" in out:
        raise RuntimeError(out["error"])
    return out["result"]


def to_float(x):
    try:
        return float(x) if x is not None else None
    except (TypeError, ValueError):
        return None


def clamp(x, lo=0, hi=5):
    return max(lo, min(hi, x))


# ======================= market weather =======================
def regime_from(avg):
    if avg is None:
        return "UNKNOWN"
    if avg <= -5:
        return "DEEP_RED"
    if avg <= -2:
        return "RED"
    if avg >= 2:
        return "GREEN"
    return "NEUTRAL"


def market_weather(demo=False, demo_regime="NEUTRAL"):
    changes = {}
    if demo:
        v = {"GREEN": 3.0, "NEUTRAL": 0.2, "RED": -3.5, "DEEP_RED": -6.0}.get(demo_regime, 0.2)
        changes = {"BTC": v + 0.4, "ETH": v - 0.3, "SOL": v - 0.1}
    else:
        for name, sym in {"BTC": "BTCUSDT", "ETH": "ETHUSDT", "SOL": "SOLUSDT"}.items():
            try:
                changes[name] = to_float(get_json(f"{BINANCE}?symbol={sym}").get("priceChangePercent"))
            except Exception:
                pass
        if len([c for c in changes.values() if c is not None]) < 3:
            try:
                d = get_json(f"{COINGECKO}?ids=bitcoin,ethereum,solana&vs_currencies=usd&include_24hr_change=true")
                for n, cid in {"BTC": "bitcoin", "ETH": "ethereum", "SOL": "solana"}.items():
                    if changes.get(n) is None:
                        changes[n] = to_float((d.get(cid) or {}).get("usd_24h_change"))
            except Exception:
                pass
    vals = [c for c in changes.values() if c is not None]
    avg = sum(vals) / len(vals) if vals else None
    return {"regime": regime_from(avg), "avg": avg, "changes": changes}


# ======================= data collection =======================
def gecko_to_pair(pool):
    a = pool.get("attributes") or {}
    rel = ((pool.get("relationships") or {}).get("base_token") or {}).get("data") or {}
    token_id = rel.get("id", "")
    if "_" not in token_id:
        return None
    addr = token_id.split("_", 1)[1]
    name = (a.get("name") or "").split(" / ")[0] or "?"
    created_ms = None
    if a.get("pool_created_at"):
        dt = datetime.fromisoformat(a["pool_created_at"].replace("Z", "+00:00"))
        created_ms = dt.timestamp() * 1000
    reserve = to_float(a.get("reserve_in_usd"))
    tx = (a.get("transactions") or {}).get("h1") or {}
    pc = a.get("price_change_percentage") or {}
    vol = a.get("volume_usd") or {}
    return {
        "chainId": "solana", "_source": "gecko", "pairAddress": a.get("address"),
        "baseToken": {"address": addr, "name": name, "symbol": name},
        "liquidity": {"usd": reserve} if reserve is not None else {},
        "marketCap": to_float(a.get("market_cap_usd")) or to_float(a.get("fdv_usd")),
        "priceUsd": a.get("base_token_price_usd"),
        "txns": {"h1": {"buys": tx.get("buys", 0), "sells": tx.get("sells", 0)}},
        "priceChange": {"h1": to_float(pc.get("h1")) or 0, "h24": to_float(pc.get("h24"))},
        "volume": {"h24": to_float(vol.get("h24"))},
        "pairCreatedAt": created_ms, "info": {},
    }


def fetch_live(limit=30):
    profiles, gecko_pairs = {}, {}
    try:
        for p in get_json(f"{DEX}/token-profiles/latest/v1"):
            if p.get("chainId") == "solana":
                profiles[p["tokenAddress"]] = p
    except Exception as e:
        print("DEX Screener profiles feed failed:", e)
    try:
        time.sleep(1)
        for pool in get_json(f"{GECKO}/networks/solana/new_pools").get("data", []):
            pair = gecko_to_pair(pool)
            if pair:
                gecko_pairs[pair["baseToken"]["address"]] = pair
    except Exception as e:
        print("GeckoTerminal feed failed:", e)

    addrs = list(dict.fromkeys(list(profiles)[:limit] + list(gecko_pairs)[:limit]))
    best = {}
    for i in range(0, len(addrs), 30):
        try:
            time.sleep(1)
            data = get_json(f"{DEX}/latest/dex/tokens/{','.join(addrs[i:i + 30])}")
        except Exception as e:
            print("DEX Screener token lookup failed:", e)
            continue
        for p in data.get("pairs") or []:
            if p.get("chainId") != "solana":
                continue
            a = p["baseToken"]["address"]
            liq = (p.get("liquidity") or {}).get("usd") or 0
            p["_source"] = "dexscreener"
            if a not in best or liq > best[a][0]:
                best[a] = (liq, p)
    pairs = [best[a][1] if a in best else gecko_pairs[a] for a in addrs if a in best or a in gecko_pairs]
    return pairs, profiles


def fetch_prices(addresses):
    prices = {}
    addresses = list(dict.fromkeys(addresses))
    for i in range(0, len(addresses), 30):
        time.sleep(1)
        data = get_json(f"{DEX}/latest/dex/tokens/{','.join(addresses[i:i + 30])}")
        best = {}
        for p in data.get("pairs") or []:
            liq = (p.get("liquidity") or {}).get("usd") or 0
            a = p["baseToken"]["address"]
            if a not in best or liq > best[a][0]:
                best[a] = (liq, p)
        for a, (_, p) in best.items():
            prices[a] = to_float(p.get("priceUsd"))
    return prices


# ======================= gate + scoring =======================
def liquidity_of(p):
    liq = (p.get("liquidity") or {}).get("usd")
    return None if liq is None else float(liq)


def age_hours(p):
    created = p.get("pairCreatedAt")
    return (time.time() * 1000 - created) / 3.6e6 if created else None


def safety_gate(p, liq_floor=BASE_MIN_LIQUIDITY):
    """Returns (passed, hard_fail_reasons, warnings)."""
    fails, warns = [], []
    liq = liquidity_of(p)
    mcap = p.get("marketCap") or p.get("fdv") or 0
    h1 = (p.get("txns") or {}).get("h1") or {}
    buys, sells = h1.get("buys", 0), h1.get("sells", 0)
    age = age_hours(p)
    pc = p.get("priceChange") or {}
    ch1, ch24 = to_float(pc.get("h1")), to_float(pc.get("h24"))
    vol24 = to_float((p.get("volume") or {}).get("h24"))

    if liq is None:
        fails.append("liquidity UNKNOWN (no data)")
    elif liq < liq_floor:
        fails.append(f"liquidity ${liq:,.0f} below today's ${liq_floor:,.0f} floor")
    if liq is not None and mcap and liq / mcap < MIN_LIQ_TO_MCAP:
        fails.append("liquidity under 5% of market cap")
    if buys > 20 and sells == 0:
        fails.append("many buys but zero sells (possible honeypot)")
    if age is not None and age < MIN_AGE_HOURS:
        fails.append("too new (under 30 min)")
    if ch1 is not None and ch1 <= COLLAPSE_1H:
        fails.append(f"price collapsing: {ch1:.0f}% in 1h")
    if ch24 is not None and ch24 <= COLLAPSE_24H:
        fails.append(f"price collapsed: {ch24:.0f}% in 24h")
    if vol24 and mcap:
        ratio = vol24 / mcap
        if ratio > VOL_MCAP_FAIL:
            fails.append(f"volume is {ratio:.0f}x market cap (churn / wash-trading hint)")
        elif ratio > VOL_MCAP_WARN:
            warns.append(f"volume is {ratio:.0f}x market cap (unusually high churn)")
    return (len(fails) == 0), fails, warns


def score_concept(p, profile):
    """Crude rule-based proxies. Later: let an AI model rate narrative and memeability."""
    base = p.get("baseToken") or {}
    info = p.get("info") or {}
    socials = info.get("socials") or []
    has_site = bool(info.get("websites"))
    has_image = bool(info.get("imageUrl"))
    desc = ((profile or {}).get("description") or "").strip()
    h1 = (p.get("txns") or {}).get("h1") or {}
    buys, sells = h1.get("buys", 0), h1.get("sells", 0)
    change_h1 = to_float((p.get("priceChange") or {}).get("h1")) or 0
    age = age_hours(p)
    age = 9999 if age is None else age

    narrative = (0 if not desc else (2 if len(desc) > 160 else 4)) + (1 if has_site else 0)
    meme = (2 if len(base.get("name", "")) <= 14 else 0) + (1 if len(base.get("symbol", "")) <= 6 else 0)
    meme += 2 if has_image else 0
    community = min(len(socials), 2)
    community += 1 if buys + sells >= 100 else 0
    community += 1 if (buys + sells) and buys / (buys + sells) >= 0.5 else 0
    community += 1 if len(socials) >= 2 else 0
    timing = (3 if 1 <= age <= 48 else (2 if age <= 336 else 0)) + (2 if 0 <= change_h1 <= 100 else 0)
    value = (3 if has_site else 0) + (2 if len(socials) >= 2 else 0)

    scores = {k: clamp(v) for k, v in {"narrative": narrative, "meme": meme, "community": community,
                                        "timing": timing, "value": value}.items()}
    total = sum(scores[k] * WEIGHTS[k] for k in scores)
    return scores, round(100 * total / (5 * sum(WEIGHTS.values())))


# ======================= deep spot checks =======================
def concentration(amounts):
    total = sum(amounts)
    if total <= 0:
        return None, None
    s = sorted(amounts, reverse=True)
    return 100 * sum(s[:10]) / total, 100 * sum(s[1:11]) / total


def check_rpc(mint):
    info = rpc("getAccountInfo", [mint, {"encoding": "jsonParsed"}])
    pinfo = ((((info or {}).get("value") or {}).get("data") or {}).get("parsed") or {}).get("info") or {}
    if not pinfo:
        raise RuntimeError("not a readable token mint")
    out = {"mint_authority": pinfo.get("mintAuthority"), "freeze_authority": pinfo.get("freezeAuthority"),
           "top10_pct": None, "top10_excl_biggest_pct": None}
    try:
        accts = rpc("getTokenLargestAccounts", [mint]).get("value") or []
        out["top10_pct"], out["top10_excl_biggest_pct"] = concentration(
            [to_float(a.get("uiAmount")) or 0 for a in accts])
    except Exception:
        pass
    return out


def check_rugcheck(mint):
    headers = {"X-API-KEY": RUG_KEY} if RUG_KEY else None
    rep = get_json(f"{RUG}/tokens/{mint}/report", headers)
    risks = [((r or {}).get("name", "?"), (r or {}).get("level", "?")) for r in rep.get("risks") or []]
    locked = None
    for m in rep.get("markets") or []:
        v = to_float((((m or {}).get("lp")) or {}).get("lpLockedPct"))
        if v is not None:
            locked = v if locked is None else max(locked, v)
    return {"risks": risks, "lp_locked_pct": locked,
            "mint_authority": rep.get("mintAuthority"), "freeze_authority": rep.get("freezeAuthority")}


def analyse_trades(rows):
    """rows = [(wallet, 'buy'/'sell', usd)]. Looks for a few wallets driving everything
    and for wallets that trade constantly with near-zero net (volume-bot behaviour)."""
    if not rows:
        return {"n": 0}
    by, total = {}, 0.0
    for wallet, kind, usd in rows:
        d = by.setdefault(wallet or "?", {"n": 0, "vol": 0.0, "buy": 0.0, "sell": 0.0})
        d["n"] += 1
        d["vol"] += usd
        d["buy" if kind == "buy" else "sell"] += usd
        total += usd
    top_wallet = max(by.items(), key=lambda kv: kv[1]["vol"])
    bots = [w for w, d in by.items()
            if d["n"] >= 15 and d["vol"] > 0 and abs(d["buy"] - d["sell"]) / d["vol"] < 0.15]
    bot_vol = sum(by[w]["vol"] for w in bots)
    return {"n": len(rows), "unique": len(by), "distinct_ratio": len(by) / len(rows),
            "top_wallet": top_wallet[0], "top_wallet_share": 100 * top_wallet[1]["vol"] / total if total else 0,
            "bot_wallets": bots, "bot_share": 100 * bot_vol / total if total else 0}


def check_pool(pool_address):
    """48h drawdown from peak (OHLCV) + recent-trades pattern. Each part can fail alone."""
    out = {"drawdown_pct": None, "trades": {"n": 0}}
    try:
        time.sleep(GECKO_PAUSE)
        d = get_json(f"{GECKO}/networks/solana/pools/{pool_address}/ohlcv/hour?aggregate=1&limit=48&currency=usd")
        candles = ((d.get("data") or {}).get("attributes") or {}).get("ohlcv_list") or []
        if candles:
            peak = max(c[2] for c in candles)
            last = sorted(candles, key=lambda c: c[0])[-1][4]
            out["drawdown_pct"] = 100 * (peak - last) / peak if peak else None
    except Exception as e:
        print(f"  (chart check failed: {e})")
    try:
        time.sleep(GECKO_PAUSE)
        d = get_json(f"{GECKO}/networks/solana/pools/{pool_address}/trades")
        rows = []
        for t in d.get("data") or []:
            a = t.get("attributes") or {}
            rows.append((a.get("tx_from_address"), a.get("kind"), to_float(a.get("volume_in_usd")) or 0))
        out["trades"] = analyse_trades(rows)
    except Exception as e:
        print(f"  (trades check failed: {e})")
    return out


def judge(rpc_data, rug_data, pool_data):
    """Combine all sources: REJECT / WATCH / UNVERIFIED."""
    fails, warns = [], []
    if rpc_data is None and rug_data is None:
        return "UNVERIFIED", ["could not reach Solana RPC or RugCheck"], []

    src = rpc_data or rug_data
    if src.get("mint_authority"):
        fails.append("mint authority NOT revoked (dev can print more supply)")
    if src.get("freeze_authority"):
        fails.append("freeze authority NOT revoked (dev can freeze your tokens)")

    if rpc_data and rpc_data.get("top10_excl_biggest_pct") is not None:
        t = rpc_data["top10_excl_biggest_pct"]
        if t > MAX_TOP10_PCT:
            fails.append(f"top-10 holders own {t:.0f}% (excluding biggest account)")
        elif t > WARN_TOP10_PCT:
            warns.append(f"top-10 holders own {t:.0f}% (watch concentration)")
    elif rpc_data:
        warns.append("holder concentration could not be read")

    if rug_data:
        bad = [n for n, lvl in rug_data["risks"] if lvl in ("danger", "critical")]
        meh = [n for n, lvl in rug_data["risks"] if lvl == "warn"]
        if bad:
            fails.append("RugCheck danger flags: " + ", ".join(bad[:3]))
        if meh:
            warns.append("RugCheck warnings: " + ", ".join(meh[:3]))
        lp = rug_data.get("lp_locked_pct")
        if lp is not None and lp < 50:
            warns.append(f"only {lp:.0f}% of liquidity locked")
    else:
        warns.append("RugCheck unavailable - check rugcheck.xyz by hand")

    pd = pool_data or {}
    dd = pd.get("drawdown_pct")
    if dd is None:
        warns.append("48h chart check unavailable")
    elif dd > DRAWDOWN_FAIL:
        fails.append(f"already {dd:.0f}% below its 48h peak (spike-then-bleed pattern)")
    elif dd > DRAWDOWN_WARN:
        warns.append(f"{dd:.0f}% below its 48h peak")

    tr = pd.get("trades") or {"n": 0}
    if tr.get("n", 0) >= 30:
        if tr["distinct_ratio"] < DISTINCT_FAIL:
            fails.append(f"only {tr['unique']} wallets made the last {tr['n']} trades (a few wallets dominate)")
        elif tr["distinct_ratio"] < DISTINCT_WARN:
            warns.append(f"only {tr['unique']} wallets in the last {tr['n']} trades")
        if tr["top_wallet_share"] > TOP_WALLET_FAIL:
            fails.append(f"one wallet drives {tr['top_wallet_share']:.0f}% of recent volume")
        elif tr["top_wallet_share"] > TOP_WALLET_WARN:
            warns.append(f"one wallet drives {tr['top_wallet_share']:.0f}% of recent volume")
        if tr["bot_share"] > BOT_VOLUME_FAIL:
            fails.append(f"bot-like round-tripping wallets drive {tr['bot_share']:.0f}% of volume")
        elif tr["bot_wallets"]:
            warns.append(f"{len(tr['bot_wallets'])} wallet(s) trade like volume bots")
    else:
        warns.append("trade-pattern check unavailable (too few trades to read)")

    if fails:
        return "REJECT", fails, warns
    if rpc_data is None:
        return "UNVERIFIED", ["Solana RPC unavailable - verify authorities on Solscan by hand"] + warns, []
    return "WATCH", [], warns


def spot_check(p, demo=False):
    mint = p["baseToken"]["address"]
    if demo:
        return judge(*demo_checks(mint))
    rpc_data = rug_data = pool_data = None
    try:
        rpc_data = check_rpc(mint)
    except Exception as e:
        print(f"  (RPC check failed for {mint[:6]}...: {e})")
    time.sleep(1)
    try:
        rug_data = check_rugcheck(mint)
    except Exception as e:
        print(f"  (RugCheck failed for {mint[:6]}...: {e})")
    if p.get("pairAddress"):
        pool_data = check_pool(p["pairAddress"])
    return judge(rpc_data, rug_data, pool_data)


def recommendation(verdict, price, fails, warns, rules):
    if verdict == "WATCH":
        extra = f" Cautions: {'; '.join(warns)}." if warns else ""
        return ("A calm look, paper only. Log entry at current price"
                f"{f' (${price})' if price else ''}, stop -30%, sell half at 2x, out by 24h." + extra)
    if verdict == "BENCH":
        return "Passed everything, but today's weather allows no more paper trades. Check again tomorrow."
    if verdict == "REJECT":
        return "Skip. " + "; ".join(fails)
    return "Not verified. " + "; ".join(fails) + " Do not trade until checked by hand."


# ======================= demo data =======================
def fetch_demo():
    now_ms = time.time() * 1000

    def mk(name, sym, liq, mcap, age_h, b, s, ch1, ch24, vol24, socials, site, desc):
        return ({
            "chainId": "solana", "_source": "demo", "priceUsd": "0.00012", "pairAddress": f"POOL{sym}",
            "baseToken": {"address": f"DEMO{sym}", "name": name, "symbol": sym},
            "liquidity": {} if liq is None else {"usd": liq}, "marketCap": mcap,
            "txns": {"h1": {"buys": b, "sells": s}},
            "priceChange": {"h1": ch1, "h24": ch24}, "volume": {"h24": vol24},
            "pairCreatedAt": now_ms - age_h * 3.6e6,
            "info": {"imageUrl": "x", "socials": [{"type": t} for t in socials],
                     "websites": [{"url": "x"}] if site else []},
        }, {"description": desc})

    rows = [
        mk("Dublin Rain Cat", "DRAIN", 60_000, 400_000, 8, 220, 150, 35, 60, 500_000, ["twitter", "telegram"], True,
           "A cat that only sleeps in the rain."),
        mk("Calm Heron", "CALM", 55_000, 450_000, 14, 120, 100, 12, 25, 300_000, ["twitter", "telegram"], True,
           "A heron that waits for the market to settle."),
        mk("Ghost Dog Walker", "GHOST", 50_000, 250_000, 30, 90, 70, 5, 20, 300_000, ["twitter", "telegram"], True,
           "Dog walking for ghosts after midnight."),
        mk("Botty McBotface", "BOTTY", 45_000, 300_000, 18, 160, 140, 10, 30, 400_000, ["twitter", "telegram"], True,
           "A bot that trades with itself."),
        mk("Spike Frog", "SPIKE", 40_000, 280_000, 20, 130, 110, 4, 10, 350_000, ["twitter", "telegram"], True,
           "Jumped once, never again."),
        mk("Quiet Owl", "OWL", 52_000, 280_000, 20, 60, 50, 8, 15, 90_000, ["twitter", "telegram"], True,
           "An owl that audits other animals."),
        mk("Wash Frog", "WASH", 20_000, 40_000, 11, 300, 280, 10, 20, 1_900_000, ["twitter"], True, "Frog."),
        mk("Falling Knife", "DUMP", 18_000, 60_000, 12, 50, 400, -35, -88, 150_000, ["twitter"], True, "Fell."),
        mk("Unknown Liq Coin", "UNKN", None, 500_000, 6, 90, 70, 15, 30, 100_000, ["twitter"], True, "Mystery."),
    ]
    return [r[0] for r in rows], {r[0]["baseToken"]["address"]: r[1] for r in rows}


def demo_checks(mint):
    import random
    rnd = random.Random(mint)
    good = {"mint_authority": None, "freeze_authority": None, "top10_pct": 40, "top10_excl_biggest_pct": 18}
    rug = {"risks": [("Mutable metadata", "warn")], "lp_locked_pct": 100,
           "mint_authority": None, "freeze_authority": None}
    healthy = [(f"W{i % 140}", "buy" if rnd.random() < 0.55 else "sell", rnd.uniform(5, 80)) for i in range(300)]
    pool = {"drawdown_pct": 20.0, "trades": analyse_trades(healthy)}
    if mint.endswith("GHOST"):
        return dict(good, mint_authority="SoMeDeVwAlLeT"), rug, pool
    if mint.endswith("OWL"):
        return None, None, None
    if mint.endswith("BOTTY"):
        rows = [("BOT1", "buy" if i % 2 == 0 else "sell", 100.0) for i in range(150)]
        rows += [(f"X{i % 120}", "buy", 10.0) for i in range(150)]
        return good, rug, {"drawdown_pct": 15.0, "trades": analyse_trades(rows)}
    if mint.endswith("SPIKE"):
        return good, rug, {"drawdown_pct": 83.0, "trades": analyse_trades(healthy)}
    return good, rug, pool


# ======================= paper trades =======================
PAPER_COLS = ["logged_at", "symbol", "address", "entry_price_usd", "entry_mcap",
              "checked_at", "price_now_usd", "change_pct", "note"]


def log_paper_trade(p, mint):
    new = not os.path.exists(PAPER_FILE)
    with open(PAPER_FILE, "a", newline="") as f:
        w = csv.writer(f)
        if new:
            w.writerow(PAPER_COLS)
        w.writerow([datetime.now(timezone.utc).isoformat(timespec="minutes"), p["baseToken"]["symbol"], mint,
                    p.get("priceUsd") or "", p.get("marketCap") or "", "", "", "", ""])


def recheck(price_fn=fetch_prices):
    if not os.path.exists(PAPER_FILE):
        print("No paper_trades.csv yet. Run a scan first.")
        return
    with open(PAPER_FILE, newline="") as f:
        rows = list(csv.DictReader(f))
    prices = price_fn([r["address"] for r in rows])
    now = datetime.now(timezone.utc)
    print(f"\n{'SYMBOL':<11}{'AGE':<8}{'ENTRY':<14}{'NOW':<14}CHANGE")
    for r in rows:
        entry, cur = to_float(r["entry_price_usd"]), prices.get(r["address"])
        age_h = (now - datetime.fromisoformat(r["logged_at"])).total_seconds() / 3600
        if entry and cur:
            chg = 100 * (cur - entry) / entry
            r.update(checked_at=now.isoformat(timespec="minutes"), price_now_usd=cur,
                     change_pct=f"{chg:.1f}", note="final-ish (24h+)" if age_h >= 24 else "so far")
            print(f"{r['symbol'][:10]:<11}{age_h:>4.0f}h   {entry:<14.8g}{cur:<14.8g}{chg:+.1f}%")
        else:
            r["note"] = "no price found (pool dead or removed?)"
            print(f"{r['symbol'][:10]:<11}{age_h:>4.0f}h   price unavailable (pool may be dead - count it as a loss)")
    with open(PAPER_FILE, "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=PAPER_COLS)
        w.writeheader()
        w.writerows(rows)
    print(f"\nUpdated {PAPER_FILE}. After 30+ rows, look at which scores and flags the winners shared.")


# ======================= the calm report =======================
CALM_LINES = [
    "Cash is a position. You don't owe the market a trade.",
    "Missing a pump costs you nothing. Joining a rug costs real money.",
    "Eat something and drink some water before any decision. Tired brains chase.",
    "Write it in the journal first. Then decide.",
    "The coins you skipped today can't hurt you tomorrow.",
    "Boring and steady beats exciting and broke.",
]

WEATHER_TEXT = {
    "GREEN": "Green skies. More people are trading, so there's more room for real buyers. Rules are normal.",
    "NEUTRAL": "Calm and steady. Nothing unusual from the big coins. Rules are normal.",
    "RED": ("Red skies. When the big coins fall, real traders go quiet and bots fill the gap, "
            "so today's rules are STRICTER."),
    "DEEP_RED": ("Deep red. The big coins are falling hard and real trading dries up. Bots and dumps "
                 "dominate. Today is a stand-down day: look, learn, log, don't trade."),
    "UNKNOWN": "Couldn't read the big coins, so I'm being a little extra careful today.",
}


def trap_category(reasons):
    text = " ".join(reasons).lower()
    if "liquidity" in text:
        return "thin or unknown liquidity"
    if "collapse" in text or "peak" in text or "dump" in text:
        return "already collapsing or dumped"
    if "volume" in text or "wallet" in text or "bot" in text or "wash" in text:
        return "wash-trading or bot-driven volume"
    if "authority" in text or "rugcheck" in text or "top-10" in text:
        return "dev could still rug it"
    if "too new" in text:
        return "too new to judge"
    return "other red flags"


def print_calm_report(results, weather, rules, watch_logged):
    print("\n" + "=" * 52)
    print("  THE CALM PART")
    print("=" * 52)
    avoided = [r for r in results if r["verdict"] in ("SKIP", "REJECT")]
    print(f"Coins scanned: {len(results)}")
    print(f"Traps you did NOT walk into: {len(avoided)}")
    cats = {}
    for r in avoided:
        c = trap_category(r["fails"])
        cats[c] = cats.get(c, 0) + 1
    for c, n in sorted(cats.items(), key=lambda kv: -kv[1]):
        print(f"   - {n} x {c}")
    print(f"Worth a calm look today: {watch_logged}")
    if weather["regime"] == "DEEP_RED":
        print("\nStand-down day. There is nothing here you need to chase.")
    elif watch_logged == 0:
        print("\nNothing earned a place today, and that's a good result. Doing nothing is a skill.")
    print("\n" + CALM_LINES[datetime.now().toordinal() % len(CALM_LINES)])


# ======================= main =======================
def main():
    if "--recheck" in sys.argv:
        recheck()
        return
    demo = "--demo" in sys.argv
    demo_regime = "NEUTRAL"
    for a in sys.argv:
        if a.startswith("--weather="):
            demo_regime = a.split("=", 1)[1].upper()

    weather = market_weather(demo, demo_regime)
    rules = WEATHER_RULES[weather["regime"]]
    liq_floor = BASE_MIN_LIQUIDITY * rules["liq_mult"]

    print("=" * 52)
    print("  TIMMY'S CONCEPT SCORER v1.3 - CALM EDITION")
    print("=" * 52)
    ch = weather["changes"]
    moves = ", ".join(f"{k} {v:+.1f}%" for k, v in ch.items() if v is not None) or "no data"
    avg = f"{weather['avg']:+.1f}%" if weather["avg"] is not None else "n/a"
    print(f"Market weather: {weather['regime'].replace('_', ' ')}  ({moves}; average {avg} over 24h)")
    print(WEATHER_TEXT[weather["regime"]])
    print(f"Today's rules: liquidity floor ${liq_floor:,.0f}, score bar {rules['min_score']}, "
          f"max {rules['max_watch']} paper trade(s).\n")

    pairs, profiles = fetch_demo() if demo else fetch_live()
    if not pairs:
        print("No pairs found. Try again in a minute.")
        return

    results = []
    for p in pairs:
        mint = p["baseToken"]["address"]
        passed, fails, warns = safety_gate(p, liq_floor)
        scores, total = score_concept(p, profiles.get(mint))
        results.append({"p": p, "mint": mint, "gate": passed, "fails": fails, "warns": warns,
                        "scores": scores, "total": total, "verdict": "SKIP" if not passed else "PENDING",
                        "rec": ""})
    results.sort(key=lambda r: (not r["gate"], -r["total"]))

    cands = [r for r in results if r["gate"] and r["total"] >= rules["min_score"]][:SPOT_CHECK_TOP_N]
    print(f"{len(results)} coins scanned, {sum(r['gate'] for r in results)} passed the gate, "
          f"{len(cands)} get the full spot check (this can take a minute - the free APIs need pauses)...")
    watch_logged = 0
    for r in cands:
        v, fails, warns = spot_check(r["p"], demo)
        r["verdict"], r["fails"], r["warns"] = v, fails, warns + r["warns"]
        if v == "WATCH":
            if watch_logged >= rules["max_watch"]:
                r["verdict"] = "BENCH"
            else:
                log_paper_trade(r["p"], r["mint"])
                watch_logged += 1
        r["rec"] = recommendation(r["verdict"], r["p"].get("priceUsd"), fails, r["warns"], rules)
    for r in results:
        if r["verdict"] == "PENDING":
            r["verdict"] = "UNCHECKED"
            r["rec"] = f"Passed the gate, but under today's score bar ({rules['min_score']}) or past the spot-check cut."
        elif r["verdict"] == "SKIP":
            r["rec"] = "Skip. " + "; ".join(r["fails"])

    order = {"WATCH": 0, "BENCH": 1, "UNVERIFIED": 2, "UNCHECKED": 3, "REJECT": 4, "SKIP": 5}
    results.sort(key=lambda r: (order[r["verdict"]], -r["total"]))

    print(f"\n{'SYMBOL':<10}{'VERDICT':<12}{'SCORE':<7}N M C T V   what to know")
    for r in results:
        s = r["scores"]
        letters = " ".join(str(s[k]) for k in ("narrative", "meme", "community", "timing", "value"))
        print(f"{r['p']['baseToken']['symbol'][:9]:<10}{r['verdict']:<12}{r['total']:<7}{letters}   {r['rec'][:100]}")

    print_calm_report(results, weather, rules, watch_logged)
    if watch_logged:
        top = next(r for r in results if r["verdict"] == "WATCH")
        print(f"\nBest survivor: {top['p']['baseToken']['symbol']} (score {top['total']}). Mint: {top['mint']}")
        print(f"Paper trade logged to {PAPER_FILE}. Run with --recheck tomorrow to see how it did.")
    print("WATCH is not SAFE. Paper trade first, and only ever risk money you can lose.")

    new = not os.path.exists(LOG_FILE)
    with open(LOG_FILE, "a", newline="") as f:
        w = csv.writer(f)
        if new:
            w.writerow(["checked_at", "weather", "symbol", "address", "source", "verdict", "reasons", "warnings",
                        "narrative", "meme", "community", "timing", "value", "total"])
        now = datetime.now(timezone.utc).isoformat(timespec="minutes")
        for r in results:
            s = r["scores"]
            w.writerow([now, weather["regime"], r["p"]["baseToken"]["symbol"], r["mint"],
                        r["p"].get("_source", ""), r["verdict"], "; ".join(r["fails"]), "; ".join(r["warns"]),
                        s["narrative"], s["meme"], s["community"], s["timing"], s["value"], r["total"]])
    print(f"Logged {len(results)} rows to {LOG_FILE}.")


if __name__ == "__main__":
    main()
