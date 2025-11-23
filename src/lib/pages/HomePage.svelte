<script lang="ts">
    import { onMount } from 'svelte';
    import { subDays, subYears } from 'date-fns';
  
    export let activities: any[] = [];
  
    let filteredActivities: any[] = [];
    let startDate = new Date();
    let endDate = new Date();
    let timeFilter = 'allTime';
  
    let totalWorkouts = 0;
    let totalTime = 0;
    let totalWorkingTime = 0;
    let totalSets = 0;
    let totalReps = 0;
    let totalVolume = 0;
    let averageRepsPerSet = 0;
    let averageTimePerWorkout = 0;
    let averageWorkingTimePerWorkout = 0;
    let totalCaloriesBurned = 0;
    let totalSweatLost = 0;

    let weightUnit = 'kg';
    let sweatUnit = 'mL';

    const filterActivities = () => {
      const now = new Date();
      switch (timeFilter) {
        case 'last7days':
          startDate = subDays(now, 7);
          endDate = now;
          break;
        case 'last30days':
          startDate = subDays(now, 30);
          endDate = now;
          break;
        case 'lastYear':
          startDate = subYears(now, 1);
          endDate = now;
          break;
        case 'allTime':
          startDate = new Date(0);
          endDate = now;
          break;
        default:
          startDate = new Date(timeFilter);
          endDate = new Date(timeFilter);
          endDate.setFullYear(endDate.getFullYear() + 1);
          break;
      }
      filteredActivities = activities.filter(activity => {
        const activityDate = new Date(activity.start_time);
        return activityDate >= startDate && activityDate < endDate && activity.exercise_sets && activity.exercise_sets.length > 0;
      });
      processActivities(filteredActivities);
    };
  
    const processActivities = (activitiesToProcess: any[]) => {
      totalWorkouts = activitiesToProcess.length;
      totalTime = activitiesToProcess.reduce((acc, activity) => {
        // Use moving_time (in seconds) from the API
        return acc + (activity.duration || 0);
      }, 0);
      totalWorkingTime = activitiesToProcess.reduce((acc, activity) => {
        // Use total_working_time (in seconds) from the API
        return acc + (activity.total_working_time || 0);
      }, 0);
      totalSets = activitiesToProcess.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.length : 0), 0);
      totalReps = activitiesToProcess.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.reduce((setAcc: any, set: { reps: any; }) => setAcc + (set.reps || 0), 0) : 0), 0);
      totalVolume = activitiesToProcess.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.reduce((setAcc: number, set: { weight: number; reps: number; }) => setAcc + ((set.weight || 0) * (set.reps || 0)), 0) : 0), 0);
      averageRepsPerSet = totalSets > 0 ? totalReps / totalSets : 0;
      averageTimePerWorkout = totalWorkouts > 0 ? totalTime / totalWorkouts : 0;
      averageWorkingTimePerWorkout = totalWorkouts > 0 ? totalWorkingTime / totalWorkouts : 0;
      // We don't have calorie or sweat data
      totalCaloriesBurned = 0;
      totalSweatLost = 0;
    };
  
    interface YearlyStats {
      yearTotalWorkouts: number;
      yearTotalTime: number;
      yearTotalWorkingTime: number;
      yearTotalSets: number;
      yearTotalReps: number;
      yearTotalVolume: number;
      yearAverageRepsPerSet: number;
      yearAverageTimePerWorkout: number;
      yearAverageWorkingTimePerWorkout: number;
    }
    
    const getYearlyStats = (year: number): YearlyStats => {
      const yearActivities = activities.filter(activity => {
        const activityDate = new Date(activity.start_time);
        return activityDate.getFullYear() === year && activity.exercise_sets && activity.exercise_sets.length > 0;
      });
      const yearTotalWorkouts = yearActivities.length;
      const yearTotalTime = yearActivities.reduce((acc, activity) => {
        return acc + (activity.duration || 0);
      }, 0);
      const yearTotalWorkingTime = yearActivities.reduce((acc, activity) => {
        return acc + (activity.total_working_time || 0);
      }, 0);
      const yearTotalSets = yearActivities.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.length : 0), 0);
      const yearTotalReps = yearActivities.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.reduce((setAcc: any, set: { reps: any; }) => setAcc + (set.reps || 0), 0) : 0), 0);
      const yearTotalVolume = yearActivities.reduce((acc, activity) => acc + (activity.exercise_sets ? activity.exercise_sets.reduce((setAcc: number, set: { weight: number; reps: number; }) => setAcc + ((set.weight || 0) * (set.reps || 0)), 0) : 0), 0);
      const yearAverageRepsPerSet = yearTotalSets > 0 ? yearTotalReps / yearTotalSets : 0;
      const yearAverageTimePerWorkout = yearTotalWorkouts > 0 ? yearTotalTime / yearTotalWorkouts : 0;
      const yearAverageWorkingTimePerWorkout = yearTotalWorkouts > 0 ? yearTotalWorkingTime / yearTotalWorkouts : 0;

      return {
        yearTotalWorkouts,
        yearTotalTime,
        yearTotalWorkingTime,
        yearTotalSets,
        yearTotalReps,
        yearTotalVolume,
        yearAverageRepsPerSet,
        yearAverageTimePerWorkout,
        yearAverageWorkingTimePerWorkout,
      };
    };

    let years: number[] = [];
  
    $: if (activities.length > 0) {
      years = Array.from(new Set(activities.map(activity => new Date(activity.start_time).getFullYear()))).sort((a, b) => b - a);
    }

    $: if (activities.length > 0 || timeFilter) {
      filterActivities();
    }

    const numberFormatter = new Intl.NumberFormat('en-US');
</script>
  
<div class="max-w-6xl px-6 lg:px-8 mx-auto">
  <div class="flex flex-wrap justify-center gap-2 mt-6">
    <button 
      class="px-4 py-2 rounded border {timeFilter === 'allTime' ? 'bg-gray-200 border-gray-400' : 'border-gray-300 hover:bg-gray-100'}"
      on:click={() => timeFilter = 'allTime'}>
      All Time
    </button>
    <button 
      class="px-4 py-2 rounded border {timeFilter === 'lastYear' ? 'bg-gray-200 border-gray-400' : 'border-gray-300 hover:bg-gray-100'}"
      on:click={() => timeFilter = 'lastYear'}>
      Last Year
    </button>
    <button 
      class="px-4 py-2 rounded border {timeFilter === 'last30days' ? 'bg-gray-200 border-gray-400' : 'border-gray-300 hover:bg-gray-100'}"
      on:click={() => timeFilter = 'last30days'}>
      Last 30 Days
    </button>
    <button 
      class="px-4 py-2 rounded border {timeFilter === 'last7days' ? 'bg-gray-200 border-gray-400' : 'border-gray-300 hover:bg-gray-100'}"
      on:click={() => timeFilter = 'last7days'}>
      Last 7 Days
    </button>
  </div>

  <div class="mb-6 max-w-full lg:max-w-4xl mx-auto">
    <div class="flex flex-wrap justify-center gap-4 bg-gray-100 p-4 rounded-lg mt-8">
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Workouts</div>
        <div class="text-xl font-bold">{numberFormatter.format(Number(totalWorkouts))}</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Time</div>
        <div class="text-xl font-bold">{Math.floor(totalTime / 3600)}h {Math.floor((totalTime % 3600) / 60)}m</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Working Time</div>
        <div class="text-xl font-bold">{Math.floor(totalWorkingTime / 3600)}h {Math.floor((totalWorkingTime % 3600) / 60)}m</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Sets</div>
        <div class="text-xl font-bold">{numberFormatter.format(Number(totalSets))}</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Reps</div>
        <div class="text-xl font-bold">{numberFormatter.format(Number(totalReps))}</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Total Volume</div>
        <div class="text-xl font-bold">{numberFormatter.format(Number((totalVolume).toFixed(0)))} {weightUnit}</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Average Reps per Set</div>
        <div class="text-xl font-bold">{averageRepsPerSet.toFixed(2)}</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Average Time per Workout</div>
        <div class="text-xl font-bold">{Math.floor(averageTimePerWorkout / 3600)}h {Math.floor((averageTimePerWorkout % 3600) / 60)}m</div>
      </div>
      <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
        <div class="text-sm text-gray-500">Average Working Time per Workout</div>
        <div class="text-xl font-bold">{Math.floor(averageWorkingTimePerWorkout / 3600)}h {Math.floor((averageWorkingTimePerWorkout % 3600) / 60)}m</div>
      </div>
    </div>
  </div>

  <!-- Yearly Section -->
  <div class="mb-8">
    <h3 class="text-2xl font-bold mx-auto text-center mb-4">Yearly Statistics</h3>
    <div class="space-y-6">
      {#each years as year}
        {@const stats = getYearlyStats(year)}
        {#if stats.yearTotalWorkouts > 0}
          <div class="border rounded-lg p-4">
            <h4 class="text-3xl font-black text-center mb-4 text-gray-700">{year}</h4>
            <div class="flex flex-wrap justify-center gap-4 bg-gray-100 p-4 rounded-lg">
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Workouts</div>
                <div class="text-xl font-bold">{numberFormatter.format(Number(stats.yearTotalWorkouts))}</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Time</div>
                <div class="text-xl font-bold">{Math.floor(stats.yearTotalTime / 3600)}h {Math.floor((stats.yearTotalTime % 3600) / 60)}m</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Working Time</div>
                <div class="text-xl font-bold">{Math.floor(stats.yearTotalWorkingTime / 3600)}h {Math.floor((stats.yearTotalWorkingTime % 3600) / 60)}m</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Sets</div>
                <div class="text-xl font-bold">{numberFormatter.format(Number(stats.yearTotalSets))}</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Reps</div>
                <div class="text-xl font-bold">{numberFormatter.format(Number(stats.yearTotalReps))}</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Total Volume</div>
                <div class="text-xl font-bold">{numberFormatter.format(Number((stats.yearTotalVolume).toFixed(0)))} {weightUnit}</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Average Reps per Set</div>
                <div class="text-xl font-bold">{stats.yearAverageRepsPerSet.toFixed(2)}</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Average Time per Workout</div>
                <div class="text-xl font-bold">{Math.floor(stats.yearAverageTimePerWorkout / 3600)}h {Math.floor((stats.yearAverageTimePerWorkout % 3600) / 60)}m</div>
              </div>
              <div class="flex-1 min-w-[150px] text-center basis-[calc(50%-1rem)] md:basis-[calc(25%-1rem)]">
                <div class="text-sm text-gray-500">Average Working Time per Workout</div>
                <div class="text-xl font-bold">{Math.floor(stats.yearAverageWorkingTimePerWorkout / 3600)}h {Math.floor((stats.yearAverageWorkingTimePerWorkout % 3600) / 60)}m</div>
              </div>
            </div>
          </div>
        {/if}
      {/each}
    </div>
  </div>
</div>
