<script lang="ts">
  import { onMount } from 'svelte';
  import CalendarPage from '$lib/pages/CalendarPage.svelte';
  import SummaryPage from '$lib/pages/SummaryPage.svelte';
  import HomePage from '$lib/pages/HomePage.svelte';
  import TrendsPage from '$lib/pages/TrendsPage.svelte';
  import TablePage from '$lib/pages/TablePage.svelte';
  import muscleGroupsData from '$lib/muscle-groups.json';

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

  let activities: Activity[] = [];
  let currentView: 'home' | 'summary' | 'calendar' | 'trends' | 'table' = 'home';
  
  let syncing = false;
  let syncMessage = '';

  const splitConfig = {
    push: {
      muscles: ['CHEST', 'SHOULDERS', 'TRICEPS'],
      color: 'red'
    },
    pull: {
      muscles: ['LATS', 'TRAPS', 'BICEPS'],
      color: 'yellow'
    },
    antagonist_push: {
      muscles: ['CHEST', 'SHOULDERS', 'BICEPS'],
      color: 'blue'
    },
    antagonist_pull: {
      muscles: ['LATS', 'TRAPS', 'TRICEPS'],
      color: 'green'
    },
    legs: {
      muscles: ['QUADS', 'HAMSTRINGS', 'GLUTES', 'CALVES', 'ADDUCTORS', 'ABDUCTORS'],
      color: 'purple'
    },
  };

  // Weight multipliers for specific lifts between specific dates
  const weightMultipliers: Array<{ lift: string; start: string; end: string; multiplier: number }> = [
    { lift: 'WEIGHTED_CRUNCH', start: '2025-06-23', end: '2025-06-23', multiplier: 0.9 },
    { lift: 'WEIGHTED_CRUNCH', start: '2024-08-18', end: '2025-04-13', multiplier: 0.7 },    
    { lift: 'WEIGHTED_LEG_CURL', start: '2025-06-15', end: '2025-07-25', multiplier: 1.7 },
    { lift: 'EZ_BAR_PREACHER_CURL', start: '2024-06-24', end: '2025-07-12', multiplier: 0.85 },
    { lift: 'EZ_BAR_PREACHER_CURL', start: '2025-08-24', end: '2026-07-12', multiplier: 1.35 },
    { lift: 'SEATED_REAR_LATERAL_RAISE', start: '2025-06-13', end: '2025-07-11', multiplier: 1.3 },
    { lift: 'LATERAL_RAISE', start: '2025-02-16', end: '2025-02-16', multiplier: 0.7 },
    { lift: 'REVERSE_EZ_BAR_CURL', start: '2025-07-29', end: '2025-07-29', multiplier: 1.8 },
    { lift: 'REVERSE_EZ_BAR_CURL', start: '2025-01-07', end: '2025-04-30', multiplier: 1.8 },
    { lift: 'WEIGHTED_LEG_CURL', start: '2025-08-24', end: '2026-08-24', multiplier: 1.36 },
    { lift: 'WEIGHTED_CRUNCH', start: '2025-08-24', end: '2026-08-24', multiplier: 0.64 },
    { lift: 'LATERAL_RAISE', start: '2025-08-24', end: '2026-08-24', multiplier: 0.52 },
  ];

  let selectedLifts = [
    'INCLINE_SMITH_MACHINE_BENCH_PRESS',
    'FLYE',
    'EZ_BAR_PREACHER_CURL',
    'REVERSE_EZ_BAR_CURL',
    'TRICEPS_PRESSDOWN',
    'LATERAL_RAISE',
    'SEATED_REAR_LATERAL_RAISE',
    'CLOSE_GRIP_LAT_PULLDOWN',
    'WEIGHTED_CRUNCH',
    'WEIGHTED_LEG_CURL',
    'WEIGHTED_LEG_EXTENSIONS',
    'BARBELL_HACK_SQUAT',
  ];

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

  function getWeightMultiplier(lift: string, date: string): number {
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

  // Determine which split day a workout is based on exercises
  function determineSplitDay(exerciseSets: ExerciseSet[]): { splitType: string | null; confidence: number } {
    if (!exerciseSets || exerciseSets.length === 0) {
      return { splitType: null, confidence: 0 };
    }

    // Count muscle groups worked in this session
    const muscleGroupCounts: Record<string, number> = {};
    let totalMuscleScore = 0;
    
    exerciseSets.forEach(set => {
      const exerciseData = muscleGroupsData[set.exercise_name as keyof typeof muscleGroupsData];
      if (exerciseData) {
        // Count primary muscles with more weight
        exerciseData.primaryMuscles.forEach((muscle: string) => {
          muscleGroupCounts[muscle] = (muscleGroupCounts[muscle] || 0) + 2;
          totalMuscleScore += 2;
        });
        // Count secondary muscles
        exerciseData.secondaryMuscles.forEach((muscle: string) => {
          muscleGroupCounts[muscle] = (muscleGroupCounts[muscle] || 0) + 1;
          totalMuscleScore += 1;
        });
      }
    });

    // Calculate score and coverage for each split type
    const splitResults: Array<{
      name: string;
      score: number;
      coverage: number;
      muscleCount: number;
      specificity: number;
      adjustedScore: number;
    }> = [];
    
    for (const [splitName, splitData] of Object.entries(splitConfig)) {
      let score = 0;
      let musclesHit = 0;
      
      splitData.muscles.forEach(muscle => {
        if (muscleGroupCounts[muscle]) {
          score += muscleGroupCounts[muscle];
          musclesHit++;
        }
      });
      
      // Coverage: percentage of split's muscles that were worked
      const coverage = musclesHit / splitData.muscles.length;
      
      // Specificity: inverse of muscle count (more specific splits are preferred)
      const specificity = 1 / splitData.muscles.length;
      
      // Dynamic coverage threshold based on split size
      // Larger splits need higher coverage to qualify
      const minCoverage = splitData.muscles.length <= 3 ? 0.4 :  // Small splits: 40%
                         splitData.muscles.length <= 5 ? 0.5 :  // Medium splits: 50%
                         0.65;                                   // Large splits: 65%
      
      // Only consider splits that meet the coverage threshold
      if (coverage >= minCoverage) {
        // For single-muscle splits, require that muscle to be dominant in the workout
        // Otherwise they match too easily (e.g., shoulders appearing in any push day)
        if (splitData.muscles.length === 1) {
          const singleMuscle = splitData.muscles[0];
          const muscleScore = muscleGroupCounts[singleMuscle] || 0;
          const musclePercentage = muscleScore / totalMuscleScore;
          
          // Single muscle must represent at least 40% of total work
          if (musclePercentage < 0.4) {
            continue; // Skip this split
          }
        }
        
        // Adjusted score balances raw score with specificity
        // For very small splits (1-2 muscles), reduce the specificity bonus
        const specificityWeight = splitData.muscles.length <= 2 ? 0.5 : 1.0;
        const adjustedScore = score * Math.pow(specificity, specificityWeight) * coverage * 100;
        
        splitResults.push({
          name: splitName,
          score,
          coverage,
          muscleCount: splitData.muscles.length,
          specificity,
          adjustedScore
        });
      }
    }

    if (splitResults.length === 0) {
      return { splitType: null, confidence: 0 };
    }

    // Sort primarily by adjusted score, which balances raw score with specificity
    splitResults.sort((a, b) => {
      // Use adjusted score which already accounts for specificity and coverage
      if (Math.abs(a.adjustedScore - b.adjustedScore) > 0.5) {
        return b.adjustedScore - a.adjustedScore;
      }
      // If adjusted scores are very close, prefer higher coverage
      if (Math.abs(a.coverage - b.coverage) > 0.1) {
        return b.coverage - a.coverage;
      }
      // Final tie-breaker: prefer more specific
      return b.specificity - a.specificity;
    });

    const bestSplit = splitResults[0];
    
    // Calculate confidence based on:
    // - How much of the split's muscles were worked (coverage)
    // - How dominant this split is vs the next best option
    let confidence = bestSplit.coverage * 100;
    
    if (splitResults.length > 1) {
      const secondBest = splitResults[1];
      const dominance = (bestSplit.adjustedScore - secondBest.adjustedScore) / bestSplit.adjustedScore;
      confidence *= (0.7 + Math.min(dominance, 0.3)); // Reduce confidence if there's a close second
    }

    return {
      splitType: bestSplit.name,
      confidence: Math.min(Math.round(confidence), 100)
    };
  }

  onMount(async () => {
    const res = await fetch('https://garmin-sync-worker.lev-s-cloudflare.workers.dev/strength-activities');
    const json = await res.json();
    activities = json.activities;
  });

  async function syncGarmin() {
    syncing = true;
    syncMessage = '';
    try {
      const res = await fetch('https://garmin-sync-worker.lev-s-cloudflare.workers.dev/sync');
      if (res.ok) {
        syncMessage = 'Sync successfully requested';
        // Reload activities after sync
        const activitiesRes = await fetch('https://garmin-sync-worker.lev-s-cloudflare.workers.dev/strength-activities');
        const json = await activitiesRes.json();
        activities = json.activities;
      } else {
        syncMessage = 'Sync failed';
      }
    } catch (e) {
      syncMessage = 'Sync error';
    }
    syncing = false;
    // Clear message after 3 seconds
    setTimeout(() => { syncMessage = ''; }, 3000);
  }
</script>

<div class="flex h-screen">
  <!-- Sidebar -->
  <aside class="w-64 bg-white border-r border-gray-200 p-4 flex flex-col">
    <div class="mb-8">
      <h1 class="text-xl font-bold mb-1">a0a7 <span class="text-sm font-normal text-gray-500"></span></h1>
      <p class="text-sm text-gray-600">hello ur fat</p>
    </div>
    
    <nav class="space-y-1 flex-1">
      <button 
        class="w-full text-left px-3 py-2 rounded-md flex items-center gap-3 {currentView === 'home' ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'}"
        on:click={() => currentView = 'home'}
      >
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6"/>
        </svg>
        Home
      </button>
      
      <button 
        class="w-full text-left px-3 py-2 rounded-md flex items-center gap-3 {currentView === 'calendar' ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'}"
        on:click={() => currentView = 'calendar'}
      >
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
        </svg>
        Calendar
      </button>
      
      <button 
        class="w-full text-left px-3 py-2 rounded-md flex items-center gap-3 {currentView === 'trends' ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'}"
        on:click={() => currentView = 'trends'}
      >
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 12l3-3 3 3 4-4M8 21l4-4 4 4M3 4h18M4 4h16v12a1 1 0 01-1 1H5a1 1 0 01-1-1V4z"/>
        </svg>
        Trends
      </button>
      
      <button 
        class="w-full text-left px-3 py-2 rounded-md flex items-center gap-3 {currentView === 'summary' ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'}"
        on:click={() => currentView = 'summary'}
      >
        <svg class="w-5 h-5" xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" ><path d="M17.596 12.768a2 2 0 1 0 2.829-2.829l-1.768-1.767a2 2 0 0 0 2.828-2.829l-2.828-2.828a2 2 0 0 0-2.829 2.828l-1.767-1.768a2 2 0 1 0-2.829 2.829z"/><path d="m2.5 21.5 1.4-1.4"/><path d="m20.1 3.9 1.4-1.4"/><path d="M5.343 21.485a2 2 0 1 0 2.829-2.828l1.767 1.768a2 2 0 1 0 2.829-2.829l-6.364-6.364a2 2 0 1 0-2.829 2.829l1.768 1.767a2 2 0 0 0-2.828 2.829z"/><path d="m9.6 14.4 4.8-4.8"/></svg>
        Progression
      </button>
      
      <button 
        class="w-full text-left px-3 py-2 rounded-md flex items-center gap-3 {currentView === 'table' ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'}"
        on:click={() => currentView = 'table'}
      >
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 10h18M3 14h18m-9-4v8m-7 0h14a2 2 0 002-2V8a2 2 0 00-2-2H5a2 2 0 00-2 2v8a2 2 0 002 2z"/>
        </svg>
        Summary Table
      </button>

      
    </nav>
    
    <!-- Sync Button at Bottom -->
    <div class="mt-auto pt-4 border-t border-gray-200">
      <button 
        class="w-full px-3 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed text-sm font-medium"
        on:click={syncGarmin} 
        disabled={syncing}
      >
        {syncing ? 'Syncing...' : 'Sync Garmin'}
      </button>
      {#if syncMessage}
        <div class="text-xs text-center mt-2 text-gray-600">{syncMessage}</div>
      {/if}
    </div>
  </aside>

  <!-- Main Content -->
  <main class="flex-1 overflow-auto p-6 bg-gray-50">
    {#if currentView === 'home'}
      <HomePage {activities} />
    {:else if currentView === 'summary'}
      <SummaryPage {activities} {selectedLifts} {liftNames} {weightMultipliers} {getWeightMultiplier} />
    {:else if currentView === 'calendar'}
      <CalendarPage {activities} {splitConfig} {determineSplitDay} />
    {:else if currentView === 'trends'}
      <TrendsPage {activities} />
    {:else if currentView === 'table'}
      <TablePage {activities} {selectedLifts} {liftNames} {getWeightMultiplier} />
    {/if}
  </main>
</div>
