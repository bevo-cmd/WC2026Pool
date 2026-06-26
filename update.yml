#!/usr/bin/env node
/**
 * 2026 World Cup Pool (tiered) — scoring job.
 *
 * Reads:   data/teams.json    (48 nations: name, iso flag code, group, feed name, tier 1/2/3)
 *          data/entries.json  (every entry: participant + one team)
 * Fetches: openfootball public results feed (no API key)
 * Writes:  data/standings.json (fully computed leaderboard + roster the page renders)
 *
 * Scoring per entry (based on the entry's single team):
 *   Advancement points (cumulative, PRE-multiplier):
 *     advance from group 5, win R32 +5, win R16 +10, win QF +10, win SF +15, win Final +20  (max 65)
 *   These are multiplied by the team's tier:  Tier1 x1, Tier2 x2, Tier3 x3
 *   Flat bonuses (NOT multiplied):
 *     Group winner (finishes 1st in group): +2
 *     Giant Slayer (Tier 3 team knocks out a Tier 1 team in a KO): +10 each time
 *   Entry total = advancement x tier + group bonus + giant-slayer bonus
 *
 * A knockout won on penalties counts as a win. Group winner decided by points, then GD, then GF.
 */

import fs from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const ROOT = path.resolve(__dirname, "..");
const DATA = path.join(ROOT, "data");
const FEED_URL = "https://raw.githubusercontent.com/openfootball/worldcup.json/master/2026/worldcup.json";

// cumulative advancement points by furthest stage reached (pre-multiplier)
const BASE = { group: 0, r32: 5, r16: 10, qf: 20, sf: 30, runner: 45, champion: 65 };
const STAGE_LABEL = { group: "Group stage", r32: "Round of 32", r16: "Round of 16", qf: "Quarter-final", sf: "Semi-final", runner: "Runner-up", champion: "Champion" };

const readJson = (p) => JSON.parse(fs.readFileSync(p, "utf8"));

function roundKey(round = "") {
  const r = round.toLowerCase();
  if (r.includes("round of 32")) return "r32";
  if (r.includes("round of 16")) return "r16";
  if (r.includes("quarter")) return "qf";
  if (r.includes("semi")) return "sf";
  if (r.includes("third")) return "thirdmatch";
  if (r.includes("final")) return "final";
  return "group";
}
// 0 = team1 wins, 1 = team2 wins, 'draw', or null (not decided)
function winnerIdx(score, ko) {
  if (!score || !Array.isArray(score.ft)) return null;
  let a = score.ft;
  if (ko && a[0] === a[1] && Array.isArray(score.et)) a = score.et;
  if (ko && a[0] === a[1] && Array.isArray(score.p)) a = score.p;
  if (a[0] === a[1]) return ko ? null : "draw";
  return a[0] > a[1] ? 0 : 1;
}

async function main() {
  const teams = readJson(path.join(DATA, "teams.json"));
  const entries = readJson(path.join(DATA, "entries.json"));
  const byFeed = new Map(), byId = new Map();
  teams.forEach((t) => { byFeed.set(t.feedName, t.id); byId.set(t.id, t); });
  const tierOf = (id) => byId.get(id)?.tier || 2;

  console.log("Fetching results feed…");
  const res = await fetch(FEED_URL, { headers: { "user-agent": "wc-pool" } });
  if (!res.ok) throw new Error(`Feed fetch failed: ${res.status}`);
  const feed = await res.json();
  const M = feed.matches || [];

  // group tables -> group winners (points, then GD, then GF)
  const gt = {};
  for (const m of M) {
    if (!m.group) continue;
    const g = m.group; gt[g] ??= {};
    const a = byFeed.get(m.team1), b = byFeed.get(m.team2);
    [a, b].forEach((id) => { if (id && !gt[g][id]) gt[g][id] = { pts: 0, gd: 0, gf: 0 }; });
    if (m.score && Array.isArray(m.score.ft) && a && b) {
      const [x, y] = m.score.ft;
      gt[g][a].gf += x; gt[g][b].gf += y; gt[g][a].gd += x - y; gt[g][b].gd += y - x;
      if (x > y) gt[g][a].pts += 3; else if (y > x) gt[g][b].pts += 3; else { gt[g][a].pts++; gt[g][b].pts++; }
    }
  }
  const groupWinner = {};
  for (const g in gt) {
    const ranked = Object.entries(gt[g]).sort((p, q) => q[1].pts - p[1].pts || q[1].gd - p[1].gd || q[1].gf - p[1].gf);
    if (ranked.length) groupWinner[ranked[0][0]] = true;
  }

  // furthest stage + giant slayer count
  const st = {};
  teams.forEach((t) => { st[t.id] = { rounds: new Set(), finalRes: null, giant: 0 }; });
  for (const m of M) {
    const k = roundKey(m.round);
    if (k === "group") continue;
    const a = byFeed.get(m.team1), b = byFeed.get(m.team2);
    const stg = k === "thirdmatch" ? "sf" : k;
    if (a) st[a].rounds.add(stg);
    if (b) st[b].rounds.add(stg);
    const w = winnerIdx(m.score, true);
    if (w !== null && w !== "draw") {
      const wi = w === 0 ? a : b, li = w === 0 ? b : a;
      if (k === "final") { if (wi) st[wi].finalRes = "win"; if (li) st[li].finalRes = "loss"; }
      if (wi && li && tierOf(wi) === 3 && tierOf(li) === 1) st[wi].giant++;
    }
  }
  const stageFor = (s) => {
    const r = s.rounds;
    if (r.has("final")) return s.finalRes === "win" ? "champion" : "runner";
    if (r.has("sf")) return "sf";
    if (r.has("qf")) return "qf";
    if (r.has("r16")) return "r16";
    if (r.has("r32")) return "r32";
    return "group";
  };

  // per-team computed scores
  const teamOut = teams.map((t) => {
    const s = st[t.id], stage = stageFor(s), mult = t.tier;
    const base = BASE[stage] * mult;
    const gw = groupWinner[t.id] ? 2 : 0;
    const gs = mult === 3 ? s.giant * 10 : 0;
    return { id: t.id, name: t.name, iso: t.iso, flag: t.flag, group: t.group, tier: t.tier,
      stage, stageLabel: STAGE_LABEL[stage], base, gw, gs, giant: s.giant, total: base + gw + gs };
  });
  const teamMap = Object.fromEntries(teamOut.map((t) => [t.id, t]));

  // ranked entries (standard competition ranking)
  const rankedEntries = entries
    .map((e) => ({ id: e.id, participant: e.participant, teamId: e.team, total: teamMap[e.team].total }))
    .sort((a, b) => b.total - a.total || teamMap[a.teamId].name.localeCompare(teamMap[b.teamId].name));
  let lastTotal = null, lastRank = 0;
  rankedEntries.forEach((e, i) => { if (e.total !== lastTotal) { lastRank = i + 1; lastTotal = e.total; } e.rank = lastRank; });

  // roster (participants in original order, with their teams and a sum)
  const order = [], byP = {};
  entries.forEach((e) => { if (!byP[e.participant]) { byP[e.participant] = []; order.push(e.participant); } byP[e.participant].push(e.team); });
  const roster = order.map((p) => {
    const teamIds = byP[p];
    const sum = teamIds.reduce((s, id) => s + teamMap[id].total, 0);
    return { participant: p, teamIds, sum };
  });

  const out = {
    updatedAt: new Date().toISOString(),
    source: "openfootball/worldcup.json (2026)",
    scoring: { advance: BASE, multipliers: { 1: 1, 2: 2, 3: 3 }, groupWinner: 2, giantSlayer: 10 },
    teams: teamOut,
    entries: rankedEntries,
    roster,
  };
  fs.writeFileSync(path.join(DATA, "standings.json"), JSON.stringify(out, null, 2));
  const top = rankedEntries[0];
  console.log(`standings.json written — ${rankedEntries.length} entries. Leader: ${teamMap[top.teamId].name} (${top.participant}) ${top.total} pts`);
}

main().catch((e) => { console.error(e); process.exit(1); });
