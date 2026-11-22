<script lang="ts">
  // PoE2's hit pipeline for the Calcs skill as a top-down tree: one bar per
  // step, coloured by damage type, with ribbons showing how each type carries
  // on, converts or grows into the next step. Numbers come from damage_flow,
  // which reads PoB's own CALCS env.
  import { untrack } from "svelte";
  import { engine, type DamageFlow, type DamageFlowSource, type DamageFlowStep, type DamageType } from "$lib/engine.svelte";
  import { build } from "$lib/state/build.svelte";
  import { m } from "$lib/paraglide/messages";

  type Ready = Extract<DamageFlow, { supported: true }>;
  type StepId = DamageFlowStep["id"];
  type Link = { from: DamageType | null; to: DamageType; amount: number };
  type Segment = { type: DamageType; x0: number; x1: number; value: number };
  type Row = { step: DamageFlowStep; total: number; prevTotal: number | null; links: Link[]; idle: boolean };

  const TYPES: DamageType[] = ["Physical", "Lightning", "Cold", "Fire", "Chaos"];
  const TYPE_VAR: Record<DamageType, string> = {
    Physical: "var(--c-physical)",
    Lightning: "var(--c-lightning)",
    Cold: "var(--c-cold)",
    Fire: "var(--c-fire)",
    Chaos: "var(--c-chaos)",
  };
  const VIEW_W = 1000;
  const RIBBON_H = 30;
  const MODE_KEY = "pob-redux:calcs-flow-mode";

  let flow = $state<DamageFlow | null>(null);
  let failed = $state(false);
  let selected = $state<StepId | null>(null);
  let showIdle = $state(false);
  let mode = $state<"scaled" | "composition">("scaled");
  try {
    if (localStorage.getItem(MODE_KEY) === "composition") mode = "composition";
  } catch {}

  function setMode(next: "scaled" | "composition") {
    mode = next;
    try {
      localStorage.setItem(MODE_KEY, next);
    } catch {}
  }

  let request = 0;
  $effect(() => {
    build.rev;
    untrack(() => {
      const id = ++request;
      engine
        .damageFlow()
        .then((r) => {
          if (id !== request) return;
          flow = r;
          failed = false;
        })
        .catch(() => {
          if (id === request) failed = true;
        });
    });
  });

  const ready = $derived(flow && flow.supported ? (flow as Ready) : null);

  function total(step: DamageFlowStep) {
    return TYPES.reduce((sum, t) => sum + (step.values[t]?.avg ?? 0), 0);
  }

  // How damage moves from the previous step into this one, in the previous step's units.
  function linksFor(step: DamageFlowStep, prev: DamageFlowStep | null): Link[] {
    if (!prev) return [];
    const links: Link[] = [];
    const prevVal = (t: DamageType) => prev.values[t]?.avg ?? 0;
    if (step.id === "added") {
      const mult = step.baseMultiplier ?? 1;
      for (const t of TYPES) {
        const carried = prevVal(t) * mult;
        if (carried > 0) links.push({ from: t, to: t, amount: carried });
        const fresh = (step.added?.[t]?.avg ?? 0) * (step.addedMult?.[t] ?? 1) * mult;
        if (fresh > 0) links.push({ from: null, to: t, amount: fresh });
      }
      return links;
    }
    if (step.id === "convSkill" || step.id === "convGlobal") {
      const out: Record<string, number> = {};
      for (const f of step.flows ?? []) {
        if (f.amount <= 0) continue;
        out[f.from] = (out[f.from] ?? 0) + f.amount;
        links.push({ from: f.from, to: f.to, amount: f.amount });
      }
      for (const t of TYPES) {
        const kept = prevVal(t) - (out[t] ?? 0);
        if (kept > 1e-9) links.push({ from: t, to: t, amount: kept });
      }
      return links;
    }
    if (step.id === "gain") {
      for (const t of TYPES) if (prevVal(t) > 0) links.push({ from: t, to: t, amount: prevVal(t) });
      for (const f of step.flows ?? []) if (f.amount > 0) links.push({ from: f.from, to: f.to, amount: f.amount });
      return links;
    }
    for (const t of TYPES) if (prevVal(t) > 0 && (step.values[t]?.avg ?? 0) > 0) links.push({ from: t, to: t, amount: prevVal(t) });
    return links;
  }

  function sameValues(a: DamageFlowStep, b: DamageFlowStep) {
    return TYPES.every((t) => Math.abs((a.values[t]?.avg ?? 0) - (b.values[t]?.avg ?? 0)) < 1e-6);
  }

  const allRows = $derived.by<Row[]>(() => {
    if (!ready) return [];
    const rows: Row[] = [];
    let prev: DamageFlowStep | null = null;
    for (const step of ready.steps) {
      const idle = prev !== null && step.id !== "taken" && sameValues(step, prev) && !(step.flows ?? []).some((f) => f.amount > 0);
      rows.push({ step, total: total(step), prevTotal: prev ? total(prev) : null, links: linksFor(step, prev), idle });
      prev = step;
    }
    return rows;
  });

  const rows = $derived(showIdle ? allRows : allRows.filter((r) => !r.idle));
  const hiddenCount = $derived(allRows.length - rows.length);
  const maxTotal = $derived(Math.max(1e-9, ...rows.map((r) => r.total)));
  const presentTypes = $derived(TYPES.filter((t) => rows.some((r) => (r.step.values[t]?.avg ?? 0) > 0)));

  function segments(row: Row): Segment[] {
    const scale = mode === "composition" ? VIEW_W / Math.max(row.total, 1e-9) : VIEW_W / maxTotal;
    let x = 0;
    const segs: Segment[] = [];
    for (const t of TYPES) {
      const v = row.step.values[t]?.avg ?? 0;
      if (v <= 0) continue;
      const w = v * scale;
      segs.push({ type: t, x0: x, x1: x + w, value: v });
      x += w;
    }
    return segs;
  }

  type Ribbon = { key: string; from: DamageType | null; to: DamageType; d: string; fresh: boolean };

  // Sankey ribbons between two consecutive bars: each source segment is shared
  // among its outgoing links, each target segment among its incoming ones.
  function ribbons(a: Row, b: Row): Ribbon[] {
    const segA = new Map(segments(a).map((s) => [s.type, s]));
    const segB = new Map(segments(b).map((s) => [s.type, s]));
    const links = b.links.filter((l) => l.amount > 0 && segB.has(l.to) && (l.from === null || segA.has(l.from)));
    const outTotal: Record<string, number> = {};
    const inTotal: Record<string, number> = {};
    for (const l of links) {
      if (l.from) outTotal[l.from] = (outTotal[l.from] ?? 0) + l.amount;
      inTotal[l.to] = (inTotal[l.to] ?? 0) + l.amount;
    }
    const order = (t: DamageType | null) => (t === null ? -1 : TYPES.indexOf(t));
    const outCursor: Record<string, number> = {};
    const inCursor: Record<string, number> = {};
    const byTarget = [...links].sort((x, y) => order(x.from) - order(y.from) || order(x.to) - order(y.to));
    const outOrder = [...links].sort((x, y) => order(x.to) - order(y.to));
    const outStart = new Map<Link, number>();
    for (const l of outOrder) {
      if (!l.from) continue;
      const s = segA.get(l.from)!;
      const w = ((s.x1 - s.x0) * l.amount) / outTotal[l.from]!;
      const x = s.x0 + (outCursor[l.from] ?? 0);
      outCursor[l.from] = (outCursor[l.from] ?? 0) + w;
      outStart.set(l, x);
    }
    const result: Ribbon[] = [];
    const h = RIBBON_H;
    for (const [li, l] of byTarget.entries()) {
      const t = segB.get(l.to)!;
      const wIn = ((t.x1 - t.x0) * l.amount) / inTotal[l.to]!;
      const bx0 = t.x0 + (inCursor[l.to] ?? 0);
      inCursor[l.to] = (inCursor[l.to] ?? 0) + wIn;
      const bx1 = bx0 + wIn;
      let ax0: number;
      let ax1: number;
      if (l.from) {
        const s = segA.get(l.from)!;
        ax0 = outStart.get(l)!;
        ax1 = ax0 + ((s.x1 - s.x0) * l.amount) / outTotal[l.from]!;
      } else {
        ax0 = bx0;
        ax1 = bx1;
      }
      const mid = h / 2;
      const d = `M${ax0},0 C${ax0},${mid} ${bx0},${mid} ${bx0},${h} L${bx1},${h} C${bx1},${mid} ${ax1},${mid} ${ax1},0 Z`;
      result.push({ key: `${li}-${l.from ?? "new"}-${l.to}`, from: l.from, to: l.to, d, fresh: l.from === null });
    }
    return result;
  }

  function fmt(value: number) {
    const magnitude = Math.abs(value);
    const compact = (divisor: number, suffix: string) => `${Number((value / divisor).toPrecision(3))}${suffix}`;
    if (magnitude >= 1_000_000_000) return compact(1_000_000_000, "b");
    if (magnitude >= 1_000_000) return compact(1_000_000, "m");
    if (magnitude >= 10_000) return compact(1_000, "k");
    if (magnitude >= 100) return Math.round(value).toLocaleString("en-US");
    return Number(value.toFixed(1)).toLocaleString("en-US");
  }

  function pct(value: number, digits = 0) {
    return `${Number(value.toFixed(digits))}%`;
  }

  function signedPct(value: number) {
    const rounded = Number(value.toFixed(Math.abs(value) < 10 ? 1 : 0));
    return `${rounded > 0 ? "+" : ""}${rounded}%`;
  }

  function mult(value: number) {
    return `×${Number(value.toFixed(value >= 10 ? 1 : 3))}`;
  }

  function typeName(t: DamageType) {
    return t;
  }

  function stepName(id: StepId) {
    switch (id) {
      case "source":
        return ready?.isAttack ? m.calcs_flow_step_source_attack() : m.calcs_flow_step_source_spell();
      case "added":
        return m.calcs_flow_step_added();
      case "convSkill":
        return m.calcs_flow_step_convSkill();
      case "convGlobal":
        return m.calcs_flow_step_convGlobal();
      case "gain":
        return m.calcs_flow_step_gain();
      case "increased":
        return m.calcs_flow_step_increased();
      case "more":
        return m.calcs_flow_step_more();
      case "other":
        return m.calcs_flow_step_other();
      case "crit":
        return m.calcs_flow_step_crit();
      case "mitigation":
        return m.calcs_flow_step_mitigation();
      case "taken":
        return m.calcs_flow_step_taken();
    }
  }

  function stepWhy(id: StepId) {
    switch (id) {
      case "source":
        return ready?.isAttack ? m.calcs_flow_why_source_attack() : m.calcs_flow_why_source_spell();
      case "added":
        return m.calcs_flow_why_added();
      case "convSkill":
        return m.calcs_flow_why_convSkill();
      case "convGlobal":
        return m.calcs_flow_why_convGlobal();
      case "gain":
        return m.calcs_flow_why_gain();
      case "increased":
        return m.calcs_flow_why_increased();
      case "more":
        return m.calcs_flow_why_more();
      case "other":
        return m.calcs_flow_why_other();
      case "crit":
        return m.calcs_flow_why_crit();
      case "mitigation":
        return m.calcs_flow_why_mitigation();
      case "taken":
        return m.calcs_flow_why_taken();
    }
  }

  // Short chips under a step: the factor each damage type got.
  function chips(row: Row): { type: DamageType | null; text: string }[] {
    const s = row.step;
    const present = TYPES.filter((t) => (s.values[t]?.avg ?? 0) > 0);
    switch (s.id) {
      case "source":
        return s.weapon ? [{ type: null, text: s.weapon.name }] : [];
      case "added": {
        const out: { type: DamageType | null; text: string }[] = [];
        for (const t of TYPES) {
          const a = s.added?.[t];
          if (a && a.max > 0) out.push({ type: t, text: `+${fmt(a.min)}–${fmt(a.max)}` });
        }
        if ((s.baseMultiplier ?? 1) !== 1) out.push({ type: null, text: `${mult(s.baseMultiplier ?? 1)} ${m.calcs_flow_effectiveness()}` });
        return out;
      }
      case "convSkill":
      case "convGlobal":
      case "gain":
        return flowChips(s, row);
      case "increased":
      case "more":
        return present.filter((t) => Math.abs((s.factor?.[t] ?? 1) - 1) > 1e-9).map((t) => ({ type: t, text: `${typeName(t)} ${mult(s.factor?.[t] ?? 1)}` }));
      case "other":
        return [{ type: null, text: mult(s.allMult ?? 1) }];
      case "crit":
        return [
          { type: null, text: `${pct(s.critChance ?? 0, 1)} × ${mult(s.critMultiplier ?? 1)}` },
          { type: null, text: `${m.calcs_flow_average_multiplier()} ${mult(s.critAvgMult ?? 1)}` },
        ];
      case "mitigation":
        return present.map((t) => {
          const e = s.enemy?.[t];
          const res = e ? e.effectiveResist : 0;
          return { type: t, text: t === "Physical" ? `${m.calcs_flow_armour()} −${pct(res, 1)}` : `${typeName(t)} ${pct(res, 1)}` };
        });
      case "taken":
        return present
          .map((t) => ({ t, e: s.enemy?.[t] ?? ready?.steps.find((x) => x.id === "mitigation")?.enemy?.[t] }))
          .filter(({ e }) => e && Math.abs(e.takenMult - 1) > 1e-9)
          .map(({ t, e }) => ({ type: t, text: `${typeName(t)} ${mult(e!.takenMult)}` }));
    }
  }

  // One chip per conversion; a gain every damage type shares ("all damage as extra fire") is one chip.
  function flowChips(s: DamageFlowStep, row: Row): { type: DamageType | null; text: string }[] {
    const flows = (s.flows ?? []).filter((f) => f.amount > 0);
    const groups = new Map<string, typeof flows>();
    for (const f of flows) {
      const key = `${f.to}|${f.pct.toFixed(3)}`;
      groups.set(key, [...(groups.get(key) ?? []), f]);
    }
    const prevIndex = rows.indexOf(row) - 1;
    const sourcesBefore = prevIndex >= 0 ? TYPES.filter((t) => (rows[prevIndex]!.step.values[t]?.avg ?? 0) > 0) : [];
    const out: { type: DamageType | null; text: string }[] = [];
    for (const group of groups.values()) {
      const f = group[0]!;
      const all = s.id === "gain" && group.length > 1 && sourcesBefore.every((t) => group.some((g) => g.from === t));
      if (all) out.push({ type: f.to, text: `${m.calcs_flow_all_damage()} → ${typeName(f.to)} ${pct(f.pct, 1)}` });
      else for (const g of group) out.push({ type: g.to, text: `${typeName(g.from)} → ${typeName(g.to)} ${pct(g.pct, 1)}` });
    }
    return out;
  }

  const selectedRow = $derived(rows.find((r) => r.step.id === selected) ?? null);
  const enemyStep = $derived(ready?.steps.find((s) => s.id === "mitigation") ?? null);
  const pobMatches = $derived(
    ready && ready.pobAverageHit != null ? Math.abs(ready.averageHit - ready.pobAverageHit) <= Math.max(0.01, ready.pobAverageHit * 1e-4) : false,
  );

  function toggle(id: StepId) {
    selected = selected === id ? null : id;
  }

  type SourceKind = "inc" | "more" | "flat" | "pct";
  function sourceValue(s: DamageFlowSource, kind: SourceKind) {
    const n = `${s.value > 0 ? "+" : ""}${Number(s.value.toFixed(1))}`;
    if (kind === "more") return `${n}% more`;
    if (kind === "flat") return n;
    return `${n}%`;
  }
</script>

{#snippet sourceList(list: DamageFlowSource[] | undefined, kind: SourceKind, againstYou = false)}
  {#if list && list.length > 0}
    <ul class="srcs">
      {#each list as s (s.source + s.mods.join())}
        <li title={s.mods.join(", ")}>
          <span class="srcname">{s.source}</span>
          <span class="srcval num" class:neg={againstYou ? s.value > 0 : s.value < 0}>{sourceValue(s, kind)}</span>
        </li>
      {/each}
    </ul>
  {:else}
    <p class="dim">{m.calcs_flow_no_sources()}</p>
  {/if}
{/snippet}

<div class="df">
  {#if failed && !flow}
    <p class="dim pad">{m.calcs_flow_unsupported()}</p>
  {:else if !flow}
    <p class="dim pad">{m.calcs_flow_loading()}</p>
  {:else if !ready}
    <p class="dim pad">{m.calcs_flow_unsupported()}</p>
  {:else}
    <div class="topbar">
      <div class="legend">
        {#each presentTypes as t (t)}
          <span class="key"><span class="swatch" style:background={TYPE_VAR[t]}></span>{typeName(t)}</span>
        {/each}
        {#if ready.pass !== "Skill"}<span class="dim pass">{ready.pass}</span>{/if}
      </div>
      <div class="controls">
        {#if hiddenCount > 0 || showIdle}
          <label class="idle">
            <input type="checkbox" bind:checked={showIdle} />
            <span>{showIdle ? m.calcs_flow_show_idle() : m.calcs_flow_hidden({ count: String(hiddenCount) })}</span>
          </label>
        {/if}
        <span class="tabs2" title={m.calcs_flow_scale_help()}>
          <button class="t2" class:on={mode === "scaled"} onclick={() => setMode("scaled")}>{m.calcs_flow_scaled()}</button>
          <button class="t2" class:on={mode === "composition"} onclick={() => setMode("composition")}>{m.calcs_flow_composition()}</button>
        </span>
      </div>
    </div>

    <div class="tree">
      {#each rows as row, i (row.step.id)}
        {#if i > 0}
          <div class="link-row">
            <span></span>
            <svg class="ribbons" viewBox={`0 0 ${VIEW_W} ${RIBBON_H}`} preserveAspectRatio="none" aria-hidden="true">
              <defs>
                {#each ribbons(rows[i - 1]!, row) as r (r.key)}
                  <linearGradient id={`df-${row.step.id}-${r.key}`} x1="0" y1="0" x2="0" y2="1">
                    <stop offset="0" stop-color={r.from ? TYPE_VAR[r.from] : TYPE_VAR[r.to]} stop-opacity={r.fresh ? 0 : 0.42} />
                    <stop offset="1" stop-color={TYPE_VAR[r.to]} stop-opacity="0.42" />
                  </linearGradient>
                {/each}
              </defs>
              {#each ribbons(rows[i - 1]!, row) as r (r.key)}
                <path d={r.d} fill={`url(#df-${row.step.id}-${r.key})`}>
                  <title>{r.from ? `${typeName(r.from)} → ${typeName(r.to)}` : `+ ${typeName(r.to)} (${m.calcs_flow_new_damage()})`}</title>
                </path>
              {/each}
            </svg>
            <span></span>
          </div>
        {/if}
        {@const delta = row.prevTotal != null && row.prevTotal > 0 ? (row.total / row.prevTotal - 1) * 100 : null}
        <button class="step" class:on={selected === row.step.id} class:idle={row.idle} onclick={() => toggle(row.step.id)} aria-expanded={selected === row.step.id}>
          <span class="steplabel">
            <span class="name"><span class="idx num">{i + 1}</span>{stepName(row.step.id)}</span>
            <span class="chips">
              {#each chips(row) as c, ci (ci)}
                <span class="chip" style:--chip={c.type ? TYPE_VAR[c.type] : "var(--fg-2)"}>{c.text}</span>
              {/each}
            </span>
          </span>
          <span class="bar">
            {#each segments(row) as s (s.type)}
              {@const r = row.step.values[s.type]}
              <span
                class="seg"
                style:left={`${(s.x0 / VIEW_W) * 100}%`}
                style:width={`max(2px, ${((s.x1 - s.x0) / VIEW_W) * 100}%)`}
                style:background={TYPE_VAR[s.type]}
                title={`${typeName(s.type)}: ${fmt(s.value)}${r && Math.abs(r.max - r.min) > 1e-9 ? ` (${fmt(r.min)}–${fmt(r.max)})` : ""}`}
              ></span>
            {/each}
          </span>
          <span class="value">
            <span class="num total">{fmt(row.total)}</span>
            {#if delta != null && Math.abs(delta) >= 0.05}
              <span class="num delta" class:up={delta > 0} class:down={delta < 0} title={m.calcs_flow_vs_previous()}>{signedPct(delta)}</span>
            {/if}
          </span>
        </button>
        {#if selected === row.step.id && selectedRow}
          {@const s = selectedRow.step}
          <div class="detail">
            <p class="why">{stepWhy(s.id)}</p>
            <div class="typevals">
              {#each TYPES.filter((t) => (s.values[t]?.avg ?? 0) > 0) as t (t)}
                {@const v = s.values[t]}
                <span class="tv" style:--chip={TYPE_VAR[t]}>
                  <span class="tvname">{typeName(t)}</span>
                  <span class="num">{fmt(v.avg)}</span>
                  {#if Math.abs(v.max - v.min) > 1e-9}<span class="num dim">{fmt(v.min)}–{fmt(v.max)}</span>{/if}
                </span>
              {/each}
            </div>

            {#if s.id === "source" && s.weapon}
              {@const w = s.weapon}
              <table class="mini">
                <thead><tr><th></th><th>{m.calcs_flow_base()}</th><th>{m.calcs_flow_final()}</th></tr></thead>
                <tbody>
                  {#each TYPES.filter((t) => w.final[t].max > 0 || w.base[t].max > 0) as t (t)}
                    <tr>
                      <td style:color={TYPE_VAR[t]}>{typeName(t)}</td>
                      <td class="num">{fmt(w.base[t].min)}–{fmt(w.base[t].max)}</td>
                      <td class="num">{fmt(w.final[t].min)}–{fmt(w.final[t].max)}</td>
                    </tr>
                  {/each}
                </tbody>
              </table>
              <p class="dim">{m.calcs_flow_quality()}: {w.quality}%</p>
              {#if w.localLines.length > 0}
                <h4>{m.calcs_flow_local_mods()}</h4>
                <ul class="lines">
                  {#each w.localLines as line, li (li)}<li>{line}</li>{/each}
                </ul>
              {/if}
            {:else if s.id === "added"}
              <p class="dim">{m.calcs_flow_effectiveness()}: {mult(s.baseMultiplier ?? 1)}</p>
              <h4>{m.calcs_flow_sources()}</h4>
              {@render sourceList(s.sources, "flat")}
            {:else if s.id === "convSkill" || s.id === "convGlobal" || s.id === "gain"}
              {#if (s.flows ?? []).some((f) => f.amount > 0)}
                <ul class="flows">
                  {#each (s.flows ?? []).filter((f) => f.amount > 0) as f (f.from + f.to)}
                    <li>
                      <span style:color={TYPE_VAR[f.from]}>{typeName(f.from)}</span> →
                      <span style:color={TYPE_VAR[f.to]}>{typeName(f.to)}</span>
                      <span class="num">{pct(f.pct, 1)}</span>
                      <span class="num dim">({fmt(f.amount)})</span>
                    </li>
                  {/each}
                </ul>
              {/if}
              <h4>{m.calcs_flow_sources()}</h4>
              {@render sourceList(s.sources, "pct")}
              {#if s.id === "convGlobal" && s.table}
                {@const conv = s.table}
                {@const fromTypes = TYPES.filter((t) => (rows.find((r) => r.step.id === "added")?.step.values[t]?.avg ?? 0) > 0)}
                {#if fromTypes.length > 0}
                  <h4>{m.calcs_flow_conversion_table()}</h4>
                  <table class="mini">
                    <thead>
                      <tr>
                        <th></th>
                        <th>{m.calcs_flow_kept()}</th>
                        {#each TYPES as t (t)}<th style:color={TYPE_VAR[t]}>{typeName(t)}</th>{/each}
                      </tr>
                    </thead>
                    <tbody>
                      {#each fromTypes as f (f)}
                        <tr>
                          <td style:color={TYPE_VAR[f]}>{typeName(f)}</td>
                          <td class="num">{pct(conv[f].kept, 1)}</td>
                          {#each TYPES as t (t)}<td class="num" class:dim={!conv[f][t]}>{conv[f][t] ? pct(conv[f][t], 1) : "–"}</td>{/each}
                        </tr>
                      {/each}
                    </tbody>
                  </table>
                {/if}
              {/if}
            {:else if s.id === "increased" || s.id === "more"}
              <div class="bytype">
                {#each TYPES.filter((t) => (s.values[t]?.avg ?? 0) > 0) as t (t)}
                  <div class="col">
                    <h4 style:color={TYPE_VAR[t]}>{typeName(t)} <span class="num">{mult(s.factor?.[t] ?? 1)}</span></h4>
                    {@render sourceList(s.sourcesByType?.[t], s.id === "more" ? "more" : "inc")}
                  </div>
                {/each}
              </div>
            {:else if s.id === "other"}
              <p class="dim">{mult(s.allMult ?? 1)}</p>
            {:else if s.id === "crit"}
              <table class="mini kv">
                <tbody>
                  <tr><td>{m.calcs_flow_crit_chance()}</td><td class="num">{pct(s.critChance ?? 0, 2)}</td></tr>
                  <tr><td>{m.calcs_flow_crit_multiplier()}</td><td class="num">{mult(s.critMultiplier ?? 1)}</td></tr>
                  <tr><td>{m.calcs_flow_average_multiplier()}</td><td class="num">{mult(s.critAvgMult ?? 1)}</td></tr>
                </tbody>
              </table>
            {:else if (s.id === "mitigation" || s.id === "taken") && enemyStep?.enemy}
              <div class="bytype">
                {#each TYPES.filter((t) => (s.values[t]?.avg ?? 0) > 0) as t (t)}
                  {@const e = enemyStep.enemy[t]}
                  {#if e}
                    <div class="col">
                      <h4 style:color={TYPE_VAR[t]}>{typeName(t)}</h4>
                      {#if s.id === "mitigation"}
                        {#if t === "Physical"}
                          <table class="mini kv">
                            <tbody>
                              <tr><td>{m.calcs_flow_armour()}</td><td class="num">{fmt(e.armour ?? 0)}</td></tr>
                              <tr><td>{m.calcs_flow_reduction()}</td><td class="num">{pct(e.effectiveResist, 1)}</td></tr>
                            </tbody>
                          </table>
                        {:else}
                          <table class="mini kv">
                            <tbody>
                              <tr><td>{m.calcs_flow_resistance()}</td><td class="num">{pct(e.resistOverride ?? e.resist ?? 0, 1)}</td></tr>
                              {#if e.penetration}<tr><td>{m.calcs_flow_penetration()}</td><td class="num">−{pct(e.penetration, 1)}</td></tr>{/if}
                              <tr><td>{m.calcs_flow_effective()}</td><td class="num">{pct(e.effectiveResist, 1)}</td></tr>
                            </tbody>
                          </table>
                          <h5>{m.calcs_flow_resistance()}</h5>
                          {@render sourceList(e.resistSources, "pct", true)}
                          {#if e.penetrationSources && e.penetrationSources.length > 0}
                            <h5>{m.calcs_flow_penetration()}</h5>
                            {@render sourceList(e.penetrationSources, "pct")}
                          {/if}
                        {/if}
                      {:else}
                        <table class="mini kv">
                          <tbody>
                            <tr><td>{m.calcs_flow_taken_inc()}</td><td class="num">{signedPct(e.takenInc)}</td></tr>
                            <tr><td>{m.calcs_flow_taken_more()}</td><td class="num">{mult(e.takenMore)}</td></tr>
                          </tbody>
                        </table>
                        {#if e.takenIncSources.length > 0}
                          <h5>{m.calcs_flow_taken_inc()}</h5>
                          {@render sourceList(e.takenIncSources, "inc")}
                        {/if}
                        {#if e.takenMoreSources.length > 0}
                          <h5>{m.calcs_flow_taken_more()}</h5>
                          {@render sourceList(e.takenMoreSources, "more")}
                        {/if}
                      {/if}
                    </div>
                  {/if}
                {/each}
              </div>
            {/if}
          </div>
        {/if}
      {/each}
    </div>

    <div class="footer">
      <span class="dim">{m.calcs_flow_click_hint()}</span>
      <span class="sum">
        {#if ready.hitChance != null && ready.averageDamage != null && ready.hitChance < 100}
          <span>{m.calcs_flow_hit_chance()} <span class="num">{pct(ready.hitChance, 1)}</span></span>
          <span>{m.calcs_flow_average_damage()} <span class="num strong">{fmt(ready.averageDamage)}</span></span>
        {/if}
        {#if pobMatches}<span class="ok" title={m.calcs_flow_pob_check()}>✓ PoB</span>{/if}
      </span>
    </div>
  {/if}
</div>

<style>
  .df {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 10px 12px 10px;
    font-size: var(--fs-xs);
  }
  .pad {
    padding: 6px 0;
  }
  .dim {
    color: var(--fg-3);
  }
  .topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    flex-wrap: wrap;
  }
  .legend {
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    color: var(--fg-1);
  }
  .key {
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }
  .swatch {
    width: 10px;
    height: 10px;
    border-radius: 2px;
  }
  .pass {
    font-style: italic;
  }
  .controls {
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    color: var(--fg-2);
    cursor: var(--cursor-pointer, pointer);
  }
  .tabs2 {
    display: inline-flex;
    border: 1px solid var(--line-1);
    border-radius: var(--r-1);
    overflow: hidden;
  }
  .t2 {
    appearance: none;
    border: 0;
    padding: 3px 9px;
    background: transparent;
    color: var(--fg-2);
    font: inherit;
    cursor: var(--cursor-pointer, pointer);
  }
  .t2.on {
    background: var(--bg-active);
    color: var(--fg-0);
  }
  .tree {
    display: flex;
    flex-direction: column;
  }
  .step,
  .link-row {
    display: grid;
    grid-template-columns: minmax(190px, 30%) minmax(0, 1fr) 92px;
    column-gap: 12px;
  }
  .step {
    appearance: none;
    text-transform: none;
    letter-spacing: normal;
    align-items: center;
    width: 100%;
    padding: 6px 8px;
    border: 1px solid transparent;
    border-radius: var(--r-1);
    background: transparent;
    color: var(--fg-0);
    font: inherit;
    text-align: left;
    cursor: var(--cursor-pointer, pointer);
  }
  .step:hover {
    background: var(--bg-hover);
  }
  .step.on {
    border-color: var(--line-2);
    background: var(--bg-2);
  }
  .step.idle {
    opacity: 0.6;
  }
  .steplabel {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }
  .name {
    display: flex;
    align-items: center;
    gap: 7px;
    font-weight: 600;
    color: var(--fg-0);
  }
  .idx {
    display: inline-grid;
    place-items: center;
    min-width: 18px;
    height: 18px;
    border-radius: 9px;
    background: var(--bg-3);
    color: var(--fg-2);
    font-size: 10px;
  }
  .chips {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
  }
  .chip {
    padding: 0 6px;
    border-radius: 8px;
    border: 1px solid color-mix(in srgb, var(--chip) 45%, transparent);
    background: color-mix(in srgb, var(--chip) 12%, transparent);
    color: var(--fg-1);
    font-size: 10.5px;
    line-height: 16px;
    white-space: nowrap;
    max-width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .bar {
    position: relative;
    height: 22px;
    border-radius: 3px;
    background: var(--bg-0);
    overflow: hidden;
  }
  .seg {
    position: absolute;
    top: 0;
    bottom: 0;
    opacity: 0.9;
  }
  .seg + .seg {
    box-shadow: inset 1px 0 0 var(--bg-0);
  }
  .value {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 1px;
  }
  .total {
    font-size: 14px;
    font-weight: 700;
    color: var(--fg-0);
  }
  .delta {
    font-size: 10.5px;
    color: var(--fg-2);
  }
  .delta.up {
    color: var(--ok);
  }
  .delta.down {
    color: var(--bad);
  }
  .link-row {
    padding: 0 8px;
  }
  .ribbons {
    display: block;
    width: 100%;
    height: 30px;
  }
  .detail {
    margin: 2px 8px 8px;
    padding: 10px 12px;
    border: 1px solid var(--line-1);
    border-radius: var(--r-1);
    background: var(--bg-2);
    display: flex;
    flex-direction: column;
    gap: 8px;
    color: var(--fg-1);
  }
  .why {
    margin: 0;
    color: var(--fg-1);
  }
  .typevals {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }
  .tv {
    display: inline-flex;
    gap: 6px;
    align-items: baseline;
    padding: 2px 8px;
    border-left: 3px solid var(--chip);
    background: var(--bg-1);
    border-radius: 2px;
  }
  .tvname {
    color: var(--fg-2);
  }
  h4,
  h5 {
    margin: 2px 0 0;
    font-size: var(--fs-xs);
    font-weight: 600;
    color: var(--fg-2);
    letter-spacing: 0.04em;
  }
  h5 {
    font-weight: 500;
  }
  .srcs,
  .lines,
  .flows {
    margin: 0;
    padding: 0;
    list-style: none;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }
  .srcs li {
    display: flex;
    justify-content: space-between;
    gap: 12px;
  }
  .srcname {
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .srcval {
    color: var(--ok);
  }
  .srcval.neg {
    color: var(--bad);
  }
  .bytype {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(190px, 1fr));
    gap: 10px 16px;
  }
  .col {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }
  table.mini {
    border-collapse: collapse;
    width: max-content;
    max-width: 100%;
  }
  table.mini th,
  table.mini td {
    padding: 2px 10px 2px 0;
    text-align: right;
    font-weight: 400;
  }
  table.mini th:first-child,
  table.mini td:first-child {
    text-align: left;
  }
  table.mini th {
    color: var(--fg-2);
  }
  table.kv td:first-child {
    color: var(--fg-2);
  }
  .footer {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    flex-wrap: wrap;
    padding-top: 6px;
    border-top: 1px solid var(--line-0);
  }
  .sum {
    display: flex;
    gap: 14px;
    color: var(--fg-2);
  }
  .strong {
    color: var(--fg-0);
    font-weight: 700;
  }
  .ok {
    color: var(--ok);
  }
  @container (width < 720px) {
    .step,
    .link-row {
      grid-template-columns: minmax(130px, 38%) minmax(0, 1fr) 70px;
    }
  }
</style>
