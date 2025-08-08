<script lang="ts">
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
let selectedLifts = ['CLOSE_GRIP_LAT_PULLDOWN', 'SEATED_REAR_LATERAL_RAISE', 'LATERAL_RAISE', 'EZ_BAR_PREACHER_CURL', 'REVERSE_EZ_BAR_CURL', 'FLYE', 'TRICEPS_PRESSDOWN', 'INCLINE_SMITH_MACHINE_BENCH_PRESS', 'BARBELL_HACK_SQUAT', 'WEIGHTED_LEG_EXTENSIONS', 'WEIGHTED_LEG_CURL', 'WEIGHTED_CRUNCH'];
let results: Record<string, LiftSummary[]> = {};

// Weight multipliers for specific lifts between specific dates
// Each entry: { lift: string, start: string (YYYY-MM-DD), end: string (YYYY-MM-DD), multiplier: number }
const weightMultipliers: Array<{ lift: string; start: string; end: string; multiplier: number }> = [
    { lift: 'WEIGHTED_CRUNCH', start: '2025-06-15', end: '2025-04-13', multiplier: 0.65 },
    { lift: 'WEIGHTED_LEG_CURL', start: '2024-08-18', end: '2025-07-25', multiplier: 1.7 },
    { lift: 'EZ_BAR_PREACHER_CURL', start: '2024-06-24', end: '2025-07-12', multiplier: 0.85 },
    { lift: 'SEATED_REAR_LATERAL_RAISE', start: '2025-06-13', end: '2025-07-11', multiplier: 1.3 },

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

    for (const lift of selectedLifts) {
      results[lift] = [];
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

        results[lift].push([
          date,
          totalVolumeKg,
          totalReps,
          totalSets,
          volumeLbs,
          avgWeightLbs,
          avgReps,
          est1RM,
          est6RM
        ]);
      }
    }
  });
</script>

<h1 class="text-xl font-bold mb-4">Lift History Summary</h1>

{#each Object.entries(results) as [lift, entries]}
  <details class="mb-4">
    <summary class="cursor-pointer font-semibold">{lift}</summary>
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