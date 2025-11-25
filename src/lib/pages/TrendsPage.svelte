<script lang="ts">
  import { onMount } from 'svelte';
  import { Chart, registerables } from 'chart.js';
  import 'chartjs-adapter-date-fns';
  import { subDays, subYears, format, startOfWeek, endOfWeek, eachDayOfInterval, eachWeekOfInterval } from 'date-fns';

  Chart.register(...registerables);

  export let activities: any[] = [];

  let timeFilter = 'allTime';
  let filteredActivities: any[] = [];
  let startDate = new Date();
  let endDate = new Date();

  // Chart containers
  let setsPerWorkoutChartCanvas: HTMLCanvasElement;
  let setsPerWeekChartCanvas: HTMLCanvasElement;
  let timePerSetChartCanvas: HTMLCanvasElement;
  let avgWeightPerSetChartCanvas: HTMLCanvasElement;
  let avgRepsPerSetChartCanvas: HTMLCanvasElement;

  
  let setsPerWorkoutChart: Chart | null = null;
  let setsPerWeekChart: Chart | null = null;
  let timePerSetChart: Chart | null = null;
  let avgWeightPerSetChart: Chart | null = null;
  let avgRepsPerSetChart: Chart | null = null;

  // Yearly chart containers
  let yearlySetsPerWorkoutCharts: Map<number, Chart> = new Map();
  let yearlySetsPerWeekCharts: Map<number, Chart> = new Map();
  let yearlyTimePerSetCharts: Map<number, Chart> = new Map();
  let yearlyAvgWeightPerSetCharts: Map<number, Chart> = new Map();
  let yearlyAvgRepsPerSetCharts: Map<number, Chart> = new Map();

  let years: number[] = [];

  interface DataPoint {
    date: Date;
    value: number;
  }

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
  };

  const calculateMovingAverage = (data: DataPoint[], windowDays: number): DataPoint[] => {
    const result: DataPoint[] = [];
    
    for (let i = 0; i < data.length; i++) {
      const currentDate = data[i].date;
      const windowStart = subDays(currentDate, windowDays);
      
      const windowData = data.filter(d => d.date >= windowStart && d.date <= currentDate);
      
      if (windowData.length > 0) {
        const avg = windowData.reduce((sum, d) => sum + d.value, 0) / windowData.length;
        result.push({ date: currentDate, value: avg });
      }
    }
    
    return result;
  };

  const calculateWeeklyMovingAverage = (data: DataPoint[], windowWeeks: number): DataPoint[] => {
    const result: DataPoint[] = [];
    
    for (let i = 0; i < data.length; i++) {
      const startIdx = Math.max(0, i - windowWeeks + 1);
      const windowData = data.slice(startIdx, i + 1);
      
      if (windowData.length > 0) {
        const avg = windowData.reduce((sum, d) => sum + d.value, 0) / windowData.length;
        result.push({ date: data[i].date, value: avg });
      }
    }
    
    return result;
  };

  const calculateLinearRegression = (data: DataPoint[]): { slope: number; intercept: number; points: DataPoint[] } => {
    if (data.length < 2) {
      return { slope: 0, intercept: 0, points: [] };
    }

    // Convert dates to timestamps for calculation
    const points = data.map(d => ({ x: d.date.getTime(), y: d.value }));
    
    const n = points.length;
    let sumX = 0;
    let sumY = 0;
    let sumXY = 0;
    let sumXX = 0;

    for (const point of points) {
      sumX += point.x;
      sumY += point.y;
      sumXY += point.x * point.y;
      sumXX += point.x * point.x;
    }

    const slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
    const intercept = (sumY - slope * sumX) / n;

    // Generate trendline points
    const minX = Math.min(...points.map(p => p.x));
    const maxX = Math.max(...points.map(p => p.x));
    
    const trendlinePoints: DataPoint[] = [
      { date: new Date(minX), value: slope * minX + intercept },
      { date: new Date(maxX), value: slope * maxX + intercept }
    ];

    return { slope, intercept, points: trendlinePoints };
  };

  const calculateSetsPerWorkout = (activitiesToProcess: any[]): DataPoint[] => {
    return activitiesToProcess.map(activity => ({
      date: new Date(activity.start_time),
      value: activity.exercise_sets ? activity.exercise_sets.length : 0
    })).sort((a, b) => a.date.getTime() - b.date.getTime());
  };

  const calculateSetsPerWeek = (activitiesToProcess: any[]): DataPoint[] => {
    if (activitiesToProcess.length === 0) return [];

    const weeklyData = new Map<string, number>();
    
    activitiesToProcess.forEach(activity => {
      const activityDate = new Date(activity.start_time);
      const weekStart = startOfWeek(activityDate, { weekStartsOn: 1 }); // Monday
      const weekKey = format(weekStart, 'yyyy-MM-dd');
      
      const sets = activity.exercise_sets ? activity.exercise_sets.length : 0;
      weeklyData.set(weekKey, (weeklyData.get(weekKey) || 0) + sets);
    });

    // Convert to array and sort
    const result: DataPoint[] = [];
    weeklyData.forEach((sets, weekKey) => {
      result.push({
        date: new Date(weekKey),
        value: sets
      });
    });

    return result.sort((a, b) => a.date.getTime() - b.date.getTime());
  };

  const calculateTimePerSet = (activitiesToProcess: any[]): DataPoint[] => {
    return activitiesToProcess.map(activity => {
      const sets = activity.exercise_sets ? activity.exercise_sets.length : 0;
      const duration = activity.duration || 0; // in seconds
      return {
        date: new Date(activity.start_time),
        value: sets > 0 ? duration / sets : 0 // seconds per set
      };
    }).filter(d => d.value > 0).sort((a, b) => a.date.getTime() - b.date.getTime());
  };

  const calculateAvgWeightPerSet = (activitiesToProcess: any[]): DataPoint[] => {
    return activitiesToProcess.map(activity => {
      if (!activity.exercise_sets || activity.exercise_sets.length === 0) {
        return { date: new Date(activity.start_time), value: 0 };
      }
      
      const totalWeight = activity.exercise_sets.reduce((sum: number, set: any) => 
        sum + (set.weight || 0), 0
      );
      const avgWeight = totalWeight / activity.exercise_sets.length;
      
      return {
        date: new Date(activity.start_time),
        value: avgWeight
      };
    }).filter(d => d.value > 0).sort((a, b) => a.date.getTime() - b.date.getTime());
  };

  const calculateAvgRepsPerSet = (activitiesToProcess: any[]): DataPoint[] => {
    return activitiesToProcess.map(activity => {
      if (!activity.exercise_sets || activity.exercise_sets.length === 0) {
        return { date: new Date(activity.start_time), value: 0 };
      }
      
      const totalReps = activity.exercise_sets.reduce((sum: number, set: any) => 
        sum + (set.reps || 0), 0
      );
      const avgReps = totalReps / activity.exercise_sets.length;
      
      return {
        date: new Date(activity.start_time),
        value: avgReps
      };
    }).filter(d => d.value > 0).sort((a, b) => a.date.getTime() - b.date.getTime());
  };

  const createChart = (
    canvas: HTMLCanvasElement,
    rawData: DataPoint[],
    movingAvgData: DataPoint[],
    trendlineData: DataPoint[],
    label: string,
    yAxisLabel: string,
    color: string,
    minDate?: Date,
    maxDate?: Date
  ): Chart => {
    return new Chart(canvas, {
      type: 'scatter',
      data: {
        datasets: [
          {
            label: 'Individual Data Points',
            data: rawData.map(d => ({ x: d.date.getTime(), y: d.value })),
            backgroundColor: color + '20',
            borderColor: color + '40',
            pointRadius: 3,
            pointHoverRadius: 5,
          },
          {
            label: '30-Day Moving Average',
            data: movingAvgData.map(d => ({ x: d.date.getTime(), y: d.value })),
            type: 'line',
            borderColor: color,
            backgroundColor: 'transparent',
            borderWidth: 2,
            pointRadius: 0,
            pointHoverRadius: 0,
          },
          {
            label: 'Trendline',
            data: trendlineData.map(d => ({ x: d.date.getTime(), y: d.value })),
            type: 'line',
            borderColor: color + 'B0',
            backgroundColor: 'transparent',
            borderWidth: 2,
            borderDash: [5, 5],
            pointRadius: 0,
            pointHoverRadius: 0,
          }
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: true,
        aspectRatio: 2,
        plugins: {
          legend: {
            display: true,
            position: 'top',
          },
          title: {
            display: true,
            text: label,
            font: {
              size: 16,
              weight: 'bold'
            }
          },
          tooltip: {
            callbacks: {
              label: function(context) {
                const value = context.parsed.y;
                if (yAxisLabel === 'Seconds') {
                  return `${Math.floor(value / 60)}m ${Math.floor(value % 60)}s`;
                }
                return `${value.toFixed(2)} ${yAxisLabel}`;
              }
            }
          }
        },
        scales: {
          x: {
            type: 'time',
            time: {
              unit: 'day',
              displayFormats: {
                day: 'MMM d'
              }
            },
            min: minDate ? minDate.getTime() : undefined,
            max: maxDate ? maxDate.getTime() : undefined,
            title: {
              display: false
            }
          },
          y: {
            beginAtZero: true,
            title: {
              display: true,
              text: yAxisLabel
            },
            ticks: {
              callback: function(value) {
                if (yAxisLabel === 'Seconds') {
                  const numValue = typeof value === 'number' ? value : 0;
                  return `${Math.floor(numValue / 60)}m`;
                }
                return value;
              }
            }
          }
        }
      }
    });
  };

  const updateCharts = () => {
    if (!filteredActivities || filteredActivities.length === 0) return;

    // Calculate data
    const setsPerWorkoutData = calculateSetsPerWorkout(filteredActivities);
    const setsPerWeekData = calculateSetsPerWeek(filteredActivities);
    const timePerSetData = calculateTimePerSet(filteredActivities);
    const avgWeightPerSetData = calculateAvgWeightPerSet(filteredActivities);
    const avgRepsPerSetData = calculateAvgRepsPerSet(filteredActivities);

    // Calculate moving averages
    const setsPerWorkoutMA = calculateMovingAverage(setsPerWorkoutData, 30);
    const setsPerWeekMA = calculateWeeklyMovingAverage(setsPerWeekData, 4); // 4-week moving average
    const timePerSetMA = calculateMovingAverage(timePerSetData, 30);
    const avgWeightPerSetMA = calculateMovingAverage(avgWeightPerSetData, 30);
    const avgRepsPerSetMA = calculateMovingAverage(avgRepsPerSetData, 30);

    // Calculate trendlines
    const setsPerWorkoutTrend = calculateLinearRegression(setsPerWorkoutData);
    const setsPerWeekTrend = calculateLinearRegression(setsPerWeekData);
    const timePerSetTrend = calculateLinearRegression(timePerSetData);
    const avgWeightPerSetTrend = calculateLinearRegression(avgWeightPerSetData);
    const avgRepsPerSetTrend = calculateLinearRegression(avgRepsPerSetData);

    // Destroy existing charts
    if (setsPerWorkoutChart) setsPerWorkoutChart.destroy();
    if (setsPerWeekChart) setsPerWeekChart.destroy();
    if (timePerSetChart) timePerSetChart.destroy();
    if (avgWeightPerSetChart) avgWeightPerSetChart.destroy();
    if (avgRepsPerSetChart) avgRepsPerSetChart.destroy();

    // Create new charts
    if (setsPerWorkoutChartCanvas) {
      setsPerWorkoutChart = createChart(
        setsPerWorkoutChartCanvas,
        setsPerWorkoutData,
        setsPerWorkoutMA,
        setsPerWorkoutTrend.points,
        'Sets per Workout',
        'Sets',
        '#3b82f6'
      );
    }

    if (setsPerWeekChartCanvas) {
      setsPerWeekChart = createChart(
        setsPerWeekChartCanvas,
        setsPerWeekData,
        setsPerWeekMA,
        setsPerWeekTrend.points,
        'Sets per Week',
        'Sets',
        '#8b5cf6'
      );
    }

    if (timePerSetChartCanvas) {
      timePerSetChart = createChart(
        timePerSetChartCanvas,
        timePerSetData,
        timePerSetMA,
        timePerSetTrend.points,
        'Time per Set',
        'Seconds',
        '#10b981'
      );
    }

    if (avgWeightPerSetChartCanvas) {
      avgWeightPerSetChart = createChart(
        avgWeightPerSetChartCanvas,
        avgWeightPerSetData,
        avgWeightPerSetMA,
        avgWeightPerSetTrend.points,
        'Average Weight per Set',
        'kg',
        '#f59e0b'
      );
    }

    if (avgRepsPerSetChartCanvas) {
      avgRepsPerSetChart = createChart(
        avgRepsPerSetChartCanvas,
        avgRepsPerSetData,
        avgRepsPerSetMA,
        avgRepsPerSetTrend.points,
        'Average Reps per Set',
        'Reps',
        '#ef4444'
      );
    }
  };

  const updateYearlyCharts = () => {
    // Destroy existing yearly charts
    yearlySetsPerWorkoutCharts.forEach(chart => chart.destroy());
    yearlySetsPerWeekCharts.forEach(chart => chart.destroy());
    yearlyTimePerSetCharts.forEach(chart => chart.destroy());
    yearlyAvgWeightPerSetCharts.forEach(chart => chart.destroy());
    yearlyAvgRepsPerSetCharts.forEach(chart => chart.destroy());

    yearlySetsPerWorkoutCharts.clear();
    yearlySetsPerWeekCharts.clear();
    yearlyTimePerSetCharts.clear();
    yearlyAvgWeightPerSetCharts.clear();
    yearlyAvgRepsPerSetCharts.clear();

    // Create charts for each year
    years.forEach(year => {
      const yearActivities = activities.filter(activity => {
        const activityDate = new Date(activity.start_time);
        return activityDate.getFullYear() === year && activity.exercise_sets && activity.exercise_sets.length > 0;
      });

      if (yearActivities.length === 0) return;

      // Calculate data for this year
      const setsPerWorkoutData = calculateSetsPerWorkout(yearActivities);
      const setsPerWeekData = calculateSetsPerWeek(yearActivities);
      const timePerSetData = calculateTimePerSet(yearActivities);
      const avgWeightPerSetData = calculateAvgWeightPerSet(yearActivities);
      const avgRepsPerSetData = calculateAvgRepsPerSet(yearActivities);

      // Calculate moving averages
      const setsPerWorkoutMA = calculateMovingAverage(setsPerWorkoutData, 30);
      const setsPerWeekMA = calculateWeeklyMovingAverage(setsPerWeekData, 4);
      const timePerSetMA = calculateMovingAverage(timePerSetData, 30);
      const avgWeightPerSetMA = calculateMovingAverage(avgWeightPerSetData, 30);
      const avgRepsPerSetMA = calculateMovingAverage(avgRepsPerSetData, 30);

      // Calculate trendlines
      const setsPerWorkoutTrend = calculateLinearRegression(setsPerWorkoutData);
      const setsPerWeekTrend = calculateLinearRegression(setsPerWeekData);
      const timePerSetTrend = calculateLinearRegression(timePerSetData);
      const avgWeightPerSetTrend = calculateLinearRegression(avgWeightPerSetData);
      const avgRepsPerSetTrend = calculateLinearRegression(avgRepsPerSetData);

      // Set year boundaries (Jan 1 to Dec 31)
      const yearStart = new Date(year, 0, 1); // January 1
      const yearEnd = new Date(year, 11, 31, 23, 59, 59); // December 31

      // Get canvas elements
      const setsPerWorkoutCanvas = document.getElementById(`yearly-sets-per-workout-${year}`) as HTMLCanvasElement;
      const setsPerWeekCanvas = document.getElementById(`yearly-sets-per-week-${year}`) as HTMLCanvasElement;
      const timePerSetCanvas = document.getElementById(`yearly-time-per-set-${year}`) as HTMLCanvasElement;
      const avgWeightPerSetCanvas = document.getElementById(`yearly-avg-weight-per-set-${year}`) as HTMLCanvasElement;
      const avgRepsPerSetCanvas = document.getElementById(`yearly-avg-reps-per-set-${year}`) as HTMLCanvasElement;

      if (setsPerWorkoutCanvas) {
        yearlySetsPerWorkoutCharts.set(year, createChart(
          setsPerWorkoutCanvas,
          setsPerWorkoutData,
          setsPerWorkoutMA,
          setsPerWorkoutTrend.points,
          `Sets per Workout - ${year}`,
          'Sets',
          '#3b82f6',
          yearStart,
          yearEnd
        ));
      }

      if (setsPerWeekCanvas) {
        yearlySetsPerWeekCharts.set(year, createChart(
          setsPerWeekCanvas,
          setsPerWeekData,
          setsPerWeekMA,
          setsPerWeekTrend.points,
          `Sets per Week - ${year}`,
          'Sets',
          '#8b5cf6',
          yearStart,
          yearEnd
        ));
      }

      if (timePerSetCanvas) {
        yearlyTimePerSetCharts.set(year, createChart(
          timePerSetCanvas,
          timePerSetData,
          timePerSetMA,
          timePerSetTrend.points,
          `Time per Set - ${year}`,
          'Seconds',
          '#10b981',
          yearStart,
          yearEnd
        ));
      }

      if (avgWeightPerSetCanvas) {
        yearlyAvgWeightPerSetCharts.set(year, createChart(
          avgWeightPerSetCanvas,
          avgWeightPerSetData,
          avgWeightPerSetMA,
          avgWeightPerSetTrend.points,
          `Average Weight per Set - ${year}`,
          'kg',
          '#f59e0b',
          yearStart,
          yearEnd
        ));
      }

      if (avgRepsPerSetCanvas) {
        yearlyAvgRepsPerSetCharts.set(year, createChart(
          avgRepsPerSetCanvas,
          avgRepsPerSetData,
          avgRepsPerSetMA,
          avgRepsPerSetTrend.points,
          `Average Reps per Set - ${year}`,
          'Reps',
          '#ef4444',
          yearStart,
          yearEnd
        ));
      }
    });
  };

  $: if (activities.length > 0) {
    years = Array.from(new Set(activities.map(activity => new Date(activity.start_time).getFullYear()))).sort((a, b) => b - a);
  }

  $: if (activities.length > 0 || timeFilter) {
    filterActivities();
    // Use setTimeout to ensure DOM is ready
    setTimeout(() => {
      updateCharts();
    }, 0);
  }

  onMount(() => {
    filterActivities();
    updateCharts();
    
    // Update yearly charts after a short delay to ensure DOM is ready
    setTimeout(() => {
      updateYearlyCharts();
    }, 100);

    return () => {
      // Cleanup charts on unmount
      if (setsPerWorkoutChart) setsPerWorkoutChart.destroy();
      if (setsPerWeekChart) setsPerWeekChart.destroy();
      if (timePerSetChart) timePerSetChart.destroy();
      if (avgWeightPerSetChart) avgWeightPerSetChart.destroy();
      if (avgRepsPerSetChart) avgRepsPerSetChart.destroy();
      
      yearlySetsPerWorkoutCharts.forEach(chart => chart.destroy());
      yearlySetsPerWeekCharts.forEach(chart => chart.destroy());
      yearlyTimePerSetCharts.forEach(chart => chart.destroy());
      yearlyAvgWeightPerSetCharts.forEach(chart => chart.destroy());
      yearlyAvgRepsPerSetCharts.forEach(chart => chart.destroy());
    };
  });

  // Watch for years changes and update yearly charts
  $: if (years.length > 0) {
    setTimeout(() => {
      updateYearlyCharts();
    }, 100);
  }
</script>

<div class="max-w-7xl px-6 lg:px-8 mx-auto">
  <!-- Time Filter Buttons -->
  <div class="flex flex-wrap justify-center gap-2 mt-6 mb-8">
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

  <!-- Filtered Period Charts -->
  <div class="grid grid-cols-1 lg:grid-cols-2 gap-2 mb-4">
    <div class="bg-white rounded-lg shadow p-2">
      <canvas bind:this={setsPerWorkoutChartCanvas}></canvas>
    </div>

    <div class="bg-white rounded-lg shadow p-2">
      <canvas bind:this={setsPerWeekChartCanvas}></canvas>
    </div>

    <div class="bg-white rounded-lg shadow p-2">
      <canvas bind:this={timePerSetChartCanvas}></canvas>
    </div>

    <div class="bg-white rounded-lg shadow p-2">
      <canvas bind:this={avgWeightPerSetChartCanvas}></canvas>
    </div>

    <div class="bg-white rounded-lg shadow p-2">
      <canvas bind:this={avgRepsPerSetChartCanvas}></canvas>
    </div>
  </div>

  <!-- Yearly Section -->
  <div class="mb-8">
    <h3 class="text-2xl font-bold mx-auto text-center mb-4">Yearly Trends</h3>
    
    {#each years as year}
      {@const yearActivities = activities.filter(activity => {
        const activityDate = new Date(activity.start_time);
        return activityDate.getFullYear() === year && activity.exercise_sets && activity.exercise_sets.length > 0;
      })}
      
      {#if yearActivities.length > 0}
        <div class="border rounded-lg p-4 mb-6">
          <h4 class="text-3xl font-black text-center mb-4 text-gray-700">{year}</h4>
          
          <div class="grid grid-cols-1 lg:grid-cols-2 gap-2">
            <div class="bg-white rounded-lg shadow p-2">
              <canvas id="yearly-sets-per-workout-{year}"></canvas>
            </div>

            <div class="bg-white rounded-lg shadow p-2">
              <canvas id="yearly-sets-per-week-{year}"></canvas>
            </div>

            <div class="bg-white rounded-lg shadow p-2">
              <canvas id="yearly-time-per-set-{year}"></canvas>
            </div>

            <div class="bg-white rounded-lg shadow p-2">
              <canvas id="yearly-avg-weight-per-set-{year}"></canvas>
            </div>

            <div class="bg-white rounded-lg shadow p-2">
              <canvas id="yearly-avg-reps-per-set-{year}"></canvas>
            </div>
          </div>
        </div>
      {/if}
    {/each}
  </div>
</div>
