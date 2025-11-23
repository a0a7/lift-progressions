<script lang="ts">
  import { onMount } from 'svelte';

  export let activities: any[];
  export let selectedLifts: string[];
  export let liftNames: Record<string, string>;
  export let getWeightMultiplier: (lift: string, date: string) => number;

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

  let results: Record<string, LiftSummary[]> = {};
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

  const kgToLbs = (kg: number) => kg * 2.20462;
  const lombardi1RM = (weight: number, reps: number) => weight * Math.pow(reps, 0.1);
  const lombardi6RM = (weight: number, reps: number) => lombardi1RM(weight, reps) / Math.pow(6, 0.1);

  $: if (activities.length > 0) {
    processActivities();
  }

  function processActivities() {
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
        const sets = activity.exercise_sets?.filter((s: any) => s.exercise_name === lift);
        if (!sets || sets.length === 0) continue;

        const date = activity.start_time.split(' ')[0];
        const multiplier = getWeightMultiplier(lift, date);
        const adjustedSets = sets.map((s: any) => ({ ...s, weight: s.weight * multiplier }));

        const totalVolumeKg = adjustedSets.reduce((sum: number, s: any) => sum + s.weight * s.reps, 0);
        const totalReps = adjustedSets.reduce((sum: number, s: any) => sum + s.reps, 0);
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

      allRows.sort((a, b) => a[0].localeCompare(b[0]));

      function avg6RMInRange(start: Date, end: Date) {
        const filtered = allRows.filter(row => {
          const d = parseDate(row[0]);
          return d >= start && d < end;
        });
        if (filtered.length === 0) return 0;
        return filtered.reduce((sum, row) => sum + (row[8] || 0), 0) / filtered.length;
      }

      let avg6RM_recent = avg6RMInRange(twoWeeksAgo, today);
      
      // If no recent data, use the most recent workout's 6RM
      if (avg6RM_recent === 0 && allRows.length > 0) {
        avg6RM_recent = allRows[allRows.length - 1][8] || 0;
      }
      
      const avg6RM_prev = avg6RMInRange(fourWeeksAgo, twoWeeksAgo);
      let avg6RM_6mo_ago = avg6RMInRange(sixMonthsAgo, new Date(sixMonthsAgo.getTime() + 14 * msInDay));
      
      if (avg6RM_6mo_ago === 0 && allRows.length > 0) {
        const firstDate = parseDate(allRows[0][0]);
        const first2w = new Date(firstDate.getTime() + 14 * msInDay);
        avg6RM_6mo_ago = avg6RMInRange(firstDate, first2w);
      }

      const raw_increase = avg6RM_recent - avg6RM_prev;
      const pct_increase = avg6RM_prev === 0 ? 0 : (raw_increase / avg6RM_prev) * 100;
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
  }
</script>

<div class="max-w-7xl mx-auto px-4">
  <!-- Summary Table View -->
  <div class="bg-white rounded-lg shadow-md p-6">
    <h2 class="text-2xl font-bold text-gray-800 mb-6">Summary Table</h2>
    <div class="overflow-x-auto">
      <table class="w-full text-sm">
        <thead>
          <tr class="bg-gray-100 border-b-2 border-gray-300">
            <th class="px-6 py-3 text-left font-semibold text-gray-700 sticky left-0 bg-gray-100">Metric</th>
            {#each summaryTable as row}
              <th class="px-4 py-3 text-center font-semibold text-gray-700 min-w-[120px]" title={liftNames[row.lift] || row.lift}>
                {liftNames[row.lift] || row.lift}
              </th>
            {/each}
          </tr>
        </thead>
        <tbody>
          <tr class="border-b hover:bg-gray-50">
            <td class="px-6 py-3 font-semibold text-gray-700 sticky left-0 bg-white">6RM Avg Last 2w</td>
            {#each summaryTable as row}
              <td class="px-4 py-3 text-center font-bold text-blue-900">{row.avg6RM_recent.toFixed(0)}</td>
            {/each}
          </tr>
          <tr class="border-b hover:bg-gray-50">
            <td class="px-6 py-3 font-semibold text-gray-700 sticky left-0 bg-white">2-wk Δ%</td>
            {#each summaryTable as row}
              <td class="px-4 py-3 text-center">
                <span class="font-bold {row.pct_increase > 0 ? 'text-green-600' : row.pct_increase < 0 ? 'text-red-600' : 'text-gray-600'}">
                  {row.pct_increase > 0 ? '+' : ''}{row.pct_increase.toFixed(1)}%
                </span>
              </td>
            {/each}
          </tr>
          <tr class="border-b hover:bg-gray-50">
            <td class="px-6 py-3 font-semibold text-gray-700 sticky left-0 bg-white">6mo Δ%</td>
            {#each summaryTable as row}
              <td class="px-4 py-3 text-center">
                <span class="font-bold {row.pct_increase_6mo > 0 ? 'text-green-600' : row.pct_increase_6mo < 0 ? 'text-red-600' : 'text-gray-600'}">
                  {row.pct_increase_6mo > 0 ? '+' : ''}{row.pct_increase_6mo.toFixed(1)}%
                </span>
              </td>
            {/each}
          </tr>
          <tr class="border-b hover:bg-gray-50">
            <td class="px-6 py-3 font-semibold text-gray-700 sticky left-0 bg-white">6mo Δ (lbs)</td>
            {#each summaryTable as row}
              <td class="px-4 py-3 text-center">
                <span class="font-semibold {row.raw_increase_6mo > 0 ? 'text-green-600' : row.raw_increase_6mo < 0 ? 'text-red-600' : 'text-gray-600'}">
                  {row.raw_increase_6mo > 0 ? '+' : ''}{row.raw_increase_6mo.toFixed(1)}
                </span>
              </td>
            {/each}
          </tr>
          <tr class="border-b hover:bg-gray-50">
            <td class="px-6 py-3 font-semibold text-gray-700 sticky left-0 bg-white">6RM Avg 6mo Ago</td>
            {#each summaryTable as row}
              <td class="px-4 py-3 text-center text-gray-600">{row.avg6RM_6mo_ago.toFixed(0)}</td>
            {/each}
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</div>
