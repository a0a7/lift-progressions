<script lang="ts">
  let syncing = false;
  let syncMessage = '';

  async function syncGarmin() {
    syncing = true;
    syncMessage = '';
    try {
      const res = await fetch('https://garmin-sync-worker.lev-s-cloudflare.workers.dev/sync');
      if (res.ok) {
        syncMessage = 'Sync request sent!';
      } else {
        syncMessage = 'Sync failed: ' + res.status;
      }
    } catch (e) {
      syncMessage = 'Sync error: ' + e;
    }
    syncing = false;
  }
  import { onMount } from 'svelte';

  type ExerciseSet = {
    exercise_name: string;
    set_number: number;
    reps: number;
    weight: number;
    start_time: string;
  };

  type Activity = {
    start_time: string;
    exercise_sets: ExerciseSet[];
  };

  type LiftSummary = [
    string, // date
    number, // volume (kg)
    number, // total reps
    number, // total sets
    number, // volume (lbs)
    number, // avg weight (lbs)
    number, // avg reps
    number, // Lombardi 1RM (lbs)
    number  // Lombardi 6RM (lbs)
  ];


let activities: Activity[] = [];
let selectedLifts = [
  'INCLINE_SMITH_MACHINE_BENCH_PRESS',
  'FLYE',
  'LATERAL_RAISE',
  'SEATED_REAR_LATERAL_RAISE',
  'EZ_BAR_PREACHER_CURL',
  'REVERSE_EZ_BAR_CURL',
  'TRICEPS_PRESSDOWN',
  'CLOSE_GRIP_LAT_PULLDOWN',
  'BARBELL_HACK_SQUAT',
  'WEIGHTED_LEG_EXTENSIONS',
  'WEIGHTED_LEG_CURL',
  'WEIGHTED_CRUNCH'
]
let results: Record<string, LiftSummary[]> = {};

// Table data for summary
let summaryTable: Array<{
  lift: string;
  avg6RM_recent: number;
  avg6RM_prev: number;
  raw_increase: number;
  pct_increase: number;
  avg6RM_6mo_ago: number;
  raw_increase_6mo: number;
  pct_increase_6mo: number;
}> = [];

// User-readable names for each exercise
const liftNames: Record<string, string> = {
  'INCLINE_SMITH_MACHINE_BENCH_PRESS': 'Incline Smith Bench',
  'FLYE': 'Chest Fly',
  'LATERAL_RAISE': 'Lateral Raise',
  'SEATED_REAR_LATERAL_RAISE': 'Reverse Fly',
  'EZ_BAR_PREACHER_CURL': 'Preacher Curl',
  'REVERSE_EZ_BAR_CURL': 'Reverse Curl',
  'TRICEPS_PRESSDOWN': 'Triceps Pushdown',
  'CLOSE_GRIP_LAT_PULLDOWN': 'Close Grip Lat Pulldown',
  'BARBELL_HACK_SQUAT': 'Barbell Hack Squat',
  'WEIGHTED_LEG_EXTENSIONS': 'Leg Extension',
  'WEIGHTED_LEG_CURL': 'Leg Curl',
  'WEIGHTED_CRUNCH': 'Weighted Crunch',
};

// Weight multipliers for specific lifts between specific dates
// Each entry: { lift: string, start: string (YYYY-MM-DD), end: string (YYYY-MM-DD), multiplier: number }
const weightMultipliers: Array<{ lift: string; start: string; end: string; multiplier: number }> = [
        { lift: 'WEIGHTED_CRUNCH', start: '2025-06-23', end: '2025-06-23', multiplier: 0.9 },
    { lift: 'WEIGHTED_CRUNCH', start: '2024-08-18', end: '2025-04-13', multiplier: 0.7 },    
    { lift: 'WEIGHTED_LEG_CURL', start: '2025-06-15', end: '2025-07-25', multiplier: 1.7 },
    { lift: 'EZ_BAR_PREACHER_CURL', start: '2024-06-24', end: '2025-07-12', multiplier: 0.85 },
    { lift: 'SEATED_REAR_LATERAL_RAISE', start: '2025-06-13', end: '2025-07-11', multiplier: 1.3 },
    { lift: 'LATERAL_RAISE', start: '2025-02-16', end: '2025-02-16', multiplier: 0.7 },
    { lift: 'REVERSE_EZ_BAR_CURL', start: '2025-07-29', end: '2025-07-29', multiplier: 1.8 },
        { lift: 'REVERSE_EZ_BAR_CURL', start: '2025-01-07   ', end: '2025-04-30', multiplier: 1.8 },


];

function getWeightMultiplier(lift: string, date: string): number {
  // date: YYYY-MM-DD
  let multiplier = 1;
  for (const entry of weightMultipliers) {
    if (
      entry.lift === lift &&
      date >= entry.start &&
      date <= entry.end
    ) {
      multiplier *= entry.multiplier;
    }
  }
  return multiplier;
}

  const kgToLbs = (kg: number) => kg * 2.20462;
  const lombardi1RM = (weight: number, reps: number) => weight * Math.pow(reps, 0.1);
  const lombardi6RM = (weight: number, reps: number) => lombardi1RM(weight, reps) / Math.pow(6, 0.1);

  onMount(async () => {
    const res = await fetch('https://garmin-sync-worker.lev-s-cloudflare.workers.dev/strength-activities');
    const json = await res.json();
    activities = json.activities;

    // Helper to parse YYYY-MM-DD to Date
    const parseDate = (d: string) => new Date(d + 'T00:00:00');
    const today = new Date();
    const msInDay = 24 * 60 * 60 * 1000;
    const twoWeeksAgo = new Date(today.getTime() - 14 * msInDay);
    const fourWeeksAgo = new Date(today.getTime() - 28 * msInDay);
    const sixMonthsAgo = new Date(today.getTime() - 183 * msInDay);

    summaryTable = [];

    for (const lift of selectedLifts) {
      results[lift] = [];
      let allRows: LiftSummary[] = [];
      for (const activity of activities) {
        const sets = activity.exercise_sets?.filter(s => s.exercise_name === lift);
        if (!sets || sets.length === 0) continue;

        const date = activity.start_time.split(' ')[0];
        const multiplier = getWeightMultiplier(lift, date);
        // Apply multiplier to each set's weight
        const adjustedSets = sets.map(s => ({ ...s, weight: s.weight * multiplier }));

        const totalVolumeKg = adjustedSets.reduce((sum, s) => sum + s.weight * s.reps, 0);
        const totalReps = adjustedSets.reduce((sum, s) => sum + s.reps, 0);
        const totalSets = adjustedSets.length;

        const avgWeightLbs = kgToLbs(totalVolumeKg / totalReps);
        const avgReps = totalReps / totalSets;

        const volumeLbs = kgToLbs(totalVolumeKg);
        const est1RM = lombardi1RM(avgWeightLbs, avgReps);
        const est6RM = lombardi6RM(avgWeightLbs, avgReps);

        const row: LiftSummary = [
          date,
          totalVolumeKg,
          totalReps,
          totalSets,
          volumeLbs,
          avgWeightLbs,
          avgReps,
          est1RM,
          est6RM
        ];
        results[lift].push(row);
        allRows.push(row);
      }

      // Sort by date ascending
      allRows.sort((a, b) => a[0].localeCompare(b[0]));

      // Helper to get average 6RM in a date range
      function avg6RMInRange(start: Date, end: Date) {
        const filtered = allRows.filter(row => {
          const d = parseDate(row[0]);
          return d >= start && d < end;
        });
        if (filtered.length === 0) return 0;
        return filtered.reduce((sum, row) => sum + (row[8] || 0), 0) / filtered.length;
      }

      // Most recent 2 weeks
      const avg6RM_recent = avg6RMInRange(twoWeeksAgo, today);
      // Previous 2 weeks
      const avg6RM_prev = avg6RMInRange(fourWeeksAgo, twoWeeksAgo);
      // 6 months ago (first 2 weeks in 6 month window)
      let avg6RM_6mo_ago = avg6RMInRange(sixMonthsAgo, new Date(sixMonthsAgo.getTime() + 14 * msInDay));
      // If no data for 6 months ago, use earliest available data
      if (avg6RM_6mo_ago === 0 && allRows.length > 0) {
        // Use average of first 2 weeks of data
        const firstDate = parseDate(allRows[0][0]);
        const first2w = new Date(firstDate.getTime() + 14 * msInDay);
        avg6RM_6mo_ago = avg6RMInRange(firstDate, first2w);
      }

      // Raw and % increase (recent vs prev)
      const raw_increase = avg6RM_recent - avg6RM_prev;
      const pct_increase = avg6RM_prev === 0 ? 0 : (raw_increase / avg6RM_prev) * 100;

      // 6 month raw and % increase (recent vs 6mo ago or earliest)
      const raw_increase_6mo = avg6RM_recent - avg6RM_6mo_ago;
      const pct_increase_6mo = avg6RM_6mo_ago === 0 ? 0 : (raw_increase_6mo / avg6RM_6mo_ago) * 100;

      summaryTable.push({
        lift,
        avg6RM_recent,
        avg6RM_prev,
        raw_increase,
        pct_increase,
        avg6RM_6mo_ago,
        raw_increase_6mo,
        pct_increase_6mo
      });
    }
  });
</script>


<div class="mb-4 flex items-center gap-4">
  <button class="px-4 py-2 bg-gray-500 text-white rounded hover:bg-gray-600 cursor-pointer disabled:opacity-50" on:click={syncGarmin} disabled={syncing}>
    {syncing ? 'Syncing...' : 'Sync Garmin'}
  </button>
  {#if syncMessage}
    <span class="text-sm">{syncMessage}</span>
  {/if}
</div>

<table class="w-full text-sm mb-6 border">
  <thead>
    <tr class="bg-gray-200">
      <th class="border px-14 py-1">Metric</th>
      {#each summaryTable as row}
        <th class="border px-2 py-1" title={liftNames[row.lift] || row.lift}>
          {liftNames[row.lift] || row.lift}
        </th>
      {/each}
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="border px-2 py-1 font-semibold">6RM Avg Last 2w</td>
      {#each summaryTable as row}
        <td class="border px-2 py-1 text-center font-bold">{row.avg6RM_recent.toFixed(0)}</td>
      {/each}
    </tr>
    <tr>
      <td class="border px-2 py-1 font-semibold">2-wk Δ%</td>
      {#each summaryTable as row}
        <td class="border px-2 py-1 text-center">
          <span class={row.pct_increase > 0 ? 'text-green-600 font-bold' : row.pct_increase < 0 ? 'text-red-600 font-semibold' : ''}>
            {row.pct_increase.toFixed(1)}%
          </span>
        </td>
      {/each}
    </tr>
    <tr>
      <td class="border px-2 py-1 font-semibold">6mo Δ%</td>
      {#each summaryTable as row}
        <td class="border px-2 py-1 text-center">
          <span class={row.pct_increase_6mo > 0 ? 'text-green-600 font-bold' : row.pct_increase_6mo < 0 ? 'text-red-600 font-semibold' : ''}>
            {row.pct_increase_6mo.toFixed(1)}%
          </span>
        </td>
      {/each}
    </tr>
    <tr>
      <td class="border px-2 py-1 font-semibold">6mo Δ</td>
      {#each summaryTable as row}
        <td class="border px-2 py-1 text-center">
          <span class={row.raw_increase_6mo > 0 ? 'text-green-600 font-bold' : row.raw_increase_6mo < 0 ? 'text-red-600 font-semibold' : ''}>
            {row.raw_increase_6mo.toFixed(2)}
          </span>
        </td>
      {/each}
    </tr>
    <tr>
      <td class="border px-2 py-1 font-semibold">6RM Avg 6mo Ago</td>
      {#each summaryTable as row}
        <td class="border px-2 py-1 text-center">{row.avg6RM_6mo_ago.toFixed(0)}</td>
      {/each}
    </tr>
  </tbody>
</table>

{#each Object.entries(results) as [lift, entries]}
  <details class="mb-4">
    <summary class="cursor-pointer font-semibold">{liftNames[lift] || lift}</summary>
    <table class="w-full text-sm mt-2 border">
      <thead>
        <tr class="bg-gray-200">
          <th class="border px-2 py-1">Date</th>
          <th class="border px-2 py-1">Volume (kg)</th>
          <th class="border px-2 py-1">Reps</th>
          <th class="border px-2 py-1">Sets</th>
          <th class="border px-2 py-1">Volume (lbs)</th>
          <th class="border px-2 py-1">Avg Wt (lbs)</th>
          <th class="border px-2 py-1">Avg Reps</th>
          <th class="border px-2 py-1">Lombardi 1RM</th>
          <th class="border px-2 py-1">Lombardi 6RM</th>
        </tr>
      </thead>
      <tbody>
        {#each entries as row}
          <tr>
            {#each row as cell}
              <td class="border px-2 py-1 text-center">{typeof cell === 'number' ? cell.toFixed(2) : cell}</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>
  </details>
{/each}