<script lang="ts">
  import { onDestroy } from 'svelte';
  import Chart from 'chart.js/auto';
  import 'chartjs-adapter-date-fns';

  export let activities: any[];
  export let selectedLifts: string[];
  export let liftNames: Record<string, string>;
  export let weightMultipliers: Array<{ lift: string; start: string; end: string; multiplier: number }>;
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

  let selectedLift: string = '';
  let timeframe: 'all' | '1y' | '6m' | '3m' = '1y';
  let currentChartKey = '';  // Track what chart is currently displayed

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

  let chartInstances: Record<string, Chart> = {};
  let trendlineEquations: Record<string, string> = {};
  let miniChartInstances: Record<string, Chart> = {};

  function getChartData(lift: string) {
    const now = new Date();
    let startDate: Date;
    
    switch (timeframe) {
      case '3m':
        startDate = new Date(now.getTime() - 90 * 24 * 60 * 60 * 1000);
        break;
      case '6m':
        startDate = new Date(now.getTime() - 183 * 24 * 60 * 60 * 1000);
        break;
      case '1y':
        startDate = new Date(now.getTime() - 365 * 24 * 60 * 60 * 1000);
        break;
      case 'all':
        startDate = new Date(0);
        break;
    }
    
    return (results[lift] || []).filter(row => {
      const d = new Date(row[0] + 'T00:00:00');
      return d >= startDate && d <= now;
    });
  }

  function get1YearData(lift: string) {
    const now = new Date();
    const yearAgo = new Date(now.getTime() - 365 * 24 * 60 * 60 * 1000);
    return (results[lift] || []).filter(row => {
      const d = new Date(row[0] + 'T00:00:00');
      return d >= yearAgo && d <= now;
    });
  }

  function calculateMovingAverage(data: {x: number, y: number}[], windowDays: number): {x: number, y: number}[] {
    const result: {x: number, y: number}[] = [];
    const msInDay = 24 * 60 * 60 * 1000;
    
    for (let i = 0; i < data.length; i++) {
      const currentTime = data[i].x;
      const windowStart = currentTime - (windowDays * msInDay);
      
      const windowData = data.filter(d => d.x >= windowStart && d.x <= currentTime);
      
      if (windowData.length > 0) {
        const avg = windowData.reduce((sum, d) => sum + d.y, 0) / windowData.length;
        result.push({ x: currentTime, y: avg });
      }
    }
    
    return result;
  }

  function getTrendline(data: {x: number, y: number}[], lift?: string) {
    if (data.length < 2) {
      if (lift) trendlineEquations[lift] = '';
      return [];
    }
    const n = data.length;
    const sumX = data.reduce((a, p) => a + p.x, 0);
    const sumY = data.reduce((a, p) => a + p.y, 0);
    const sumXY = data.reduce((a, p) => a + p.x * p.y, 0);
    const sumXX = data.reduce((a, p) => a + p.x * p.x, 0);
    const slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX);
    const intercept = (sumY - slope * sumX) / n;
    
    // Calculate R^2
    const meanY = sumY / n;
    let ssTot = 0, ssRes = 0;
    for (const p of data) {
      const yPred = slope * p.x + intercept;
      ssTot += (p.y - meanY) ** 2;
      ssRes += (p.y - yPred) ** 2;
    }
    const r2 = ssTot === 0 ? 1 : 1 - ssRes / ssTot;
    
    const slopePerMonth = slope * 86400000 * 30;
    if (lift) {
      trendlineEquations[lift] = `a = ${slopePerMonth.toFixed(2)} /mo, R² = ${r2.toFixed(2)}`;
    }
    const x0 = data[0].x;
    const x1 = data[data.length - 1].x;
    return [
      { x: x0, y: slope * x0 + intercept },
      { x: x1, y: slope * x1 + intercept }
    ];
  }

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

  function updateChart(lift: string) {
    const canvas = document.getElementById('chart-' + lift) as HTMLCanvasElement;
    if (!canvas) return;
    
    // Destroy existing chart for this lift
    if (chartInstances[lift]) {
      chartInstances[lift].destroy();
      delete chartInstances[lift];
    }
    
    const dataRows = getChartData(lift);
    if (dataRows.length === 0) {
      trendlineEquations[lift] = '';
      return;
    }
    
    const points = dataRows.map(row => ({
      x: new Date(row[0] + 'T00:00:00').getTime(),
      y: row[8]
    }));
    
    const movingAvg = calculateMovingAverage(points, 30);
    const trend = getTrendline(points, lift);
    
    // Base color for consistency
    const baseColor = 'rgb(99, 102, 241)'; // Indigo
    
    chartInstances[lift] = new Chart(canvas, {
      type: 'scatter',
      data: {
        datasets: [
          {
            label: '6RM Data Points',
            data: points,
            backgroundColor: 'rgba(99, 102, 241, 0.4)',
            borderColor: 'rgba(99, 102, 241, 0.6)',
            pointRadius: 4,
            pointHoverRadius: 6,
          },
          {
            type: 'line' as const,
            label: '30-Day Moving Average',
            data: movingAvg,
            borderColor: 'rgba(99, 102, 241, 0.8)',
            backgroundColor: 'transparent',
            borderWidth: 2.5,
            pointRadius: 0,
            tension: 0.3,
            order: 1,
          },
          ...(trend.length === 2
            ? [{
                type: 'line' as const,
                label: 'Trendline',
                data: trend,
                borderColor: 'rgba(99, 102, 241, 1)',
                borderWidth: 2,
                borderDash: [8, 4],
                fill: false,
                pointRadius: 0,
                tension: 0,
                order: 2,
              }]
            : [])
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        animation: false,  // Disable animations to prevent spazzing
        plugins: {
          legend: { 
            display: true,
            position: 'top',
            labels: {
              usePointStyle: true,
              padding: 15,
              font: {
                size: 12
              }
            }
          },
          title: { display: false }
        },
        scales: {
          x: {
            type: 'time',
            time: { unit: 'month' },
            title: { 
              display: true, 
              text: 'Date',
              font: {
                size: 13,
                weight: 'bold'
              }
            },
            grid: {
              color: 'rgba(0, 0, 0, 0.05)'
            }
          },
          y: {
            title: { 
              display: true, 
              text: '6RM (lbs)',
              font: {
                size: 13,
                weight: 'bold'
              }
            },
            beginAtZero: false,
            grid: {
              color: 'rgba(0, 0, 0, 0.05)'
            }
          }
        }
      }
    });
  }

  function createMiniChart(lift: string) {
    const canvas = document.getElementById('mini-chart-' + lift) as HTMLCanvasElement;
    if (!canvas) return;
    
    // Destroy existing mini chart for this lift
    if (miniChartInstances[lift]) {
      miniChartInstances[lift].destroy();
      delete miniChartInstances[lift];
    }
    
    const dataRows = get1YearData(lift);
    if (dataRows.length === 0) return;
    
    const points = dataRows.map(row => ({
      x: new Date(row[0] + 'T00:00:00').getTime(),
      y: row[8]
    }));
    
    const movingAvg = calculateMovingAverage(points, 30);
    const trend = getTrendline(points);
    
    miniChartInstances[lift] = new Chart(canvas, {
      type: 'scatter',
      data: {
        datasets: [
          {
            label: '6RM',
            data: points,
            backgroundColor: 'rgba(99, 102, 241, 0.4)',
            borderColor: 'rgba(99, 102, 241, 0.6)',
            pointRadius: 2,
            pointHoverRadius: 3,
          },
          {
            type: 'line' as const,
            label: 'MA',
            data: movingAvg,
            borderColor: 'rgba(99, 102, 241, 0.8)',
            backgroundColor: 'transparent',
            borderWidth: 1.5,
            pointRadius: 0,
            tension: 0.3,
          },
          ...(trend.length === 2
            ? [{
                type: 'line' as const,
                label: 'Trend',
                data: trend,
                borderColor: 'rgba(99, 102, 241, 1)',
                borderWidth: 1.5,
                borderDash: [4, 2],
                fill: false,
                pointRadius: 0,
                tension: 0,
              }]
            : [])
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        animation: false,
        plugins: {
          legend: { display: false },
          title: { display: false },
          tooltip: { enabled: false }
        },
        scales: {
          x: {
            type: 'time',
            display: false
          },
          y: {
            display: true,
            beginAtZero: false,
            ticks: {
              font: {
                size: 9
              }
            },
            grid: {
              color: 'rgba(0, 0, 0, 0.05)'
            }
          }
        },
        onClick: () => {
          selectedLift = lift;
        }
      }
    });
  }

  function updateMiniCharts() {
    selectedLifts.forEach(lift => {
      setTimeout(() => {
        createMiniChart(lift);
      }, 10);
    });
  }

  // Single reactive statement to update chart when selectedLift or timeframe changes
  $: {
    const chartKey = selectedLift + '-' + timeframe;
    if (selectedLift && results[selectedLift] && chartKey !== currentChartKey) {
      currentChartKey = chartKey;
      
      // Destroy all existing charts to prevent memory leaks
      Object.keys(chartInstances).forEach(key => {
        if (chartInstances[key]) {
          chartInstances[key].destroy();
          delete chartInstances[key];
        }
      });
      
      // Use setTimeout to ensure DOM is ready after {#key} block recreation
      setTimeout(() => {
        updateChart(selectedLift);
      }, 50);
    }
  }

  onDestroy(() => {
    for (const k in chartInstances) {
      if (chartInstances[k]) chartInstances[k].destroy();
    }
    for (const k in miniChartInstances) {
      if (miniChartInstances[k]) miniChartInstances[k].destroy();
    }
  });

  $: if (selectedLifts.length > 0 && !selectedLift) {
    selectedLift = selectedLifts[0];
  }

  // Update mini charts when results are available
  $: if (Object.keys(results).length > 0 && selectedLifts.length > 0) {
    setTimeout(() => {
      updateMiniCharts();
    }, 100);
  }
</script>

<div class="max-w-7xl mx-auto px-2 sm:px-4">
  <!-- Timeframe Selector -->
    <label for="timeframe-select" class="block text-sm font-medium text-gray-700 mb-2">Timeframe</label>
    <select 
      id="timeframe-select"
      bind:value={timeframe}
      class="w-full md:w-64 px-2 py-1 mb-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent text-sm md:text-base">
      <option value="all">All Time</option>
      <option value="1y">Last Year</option>
      <option value="6m">Last 6 Months</option>
      <option value="3m">Last 3 Months</option>
    </select>

  <!-- Horizontally Scrollable Mini Charts -->
  <div class="mb-4 overflow-x-auto whitespace-nowrap pb-2 -mx-2 px-2">
    <div class="inline-flex gap-2">
      {#each selectedLifts as lift}
        <button 
          on:click={() => selectedLift = lift}
          class="inline-block bg-white rounded border p-2 hover:shadow-md transition-shadow cursor-pointer {selectedLift === lift ? 'ring-2 ring-blue-500 border-blue-500' : 'border-gray-200'}"
          style="width: max(15vw, 140px); min-width: 140px;">
          <div class="text-xs font-medium text-gray-700 mb-1 truncate">{liftNames[lift] || lift}</div>
          <div style="height: 100px;">
            <canvas id={'mini-chart-' + lift}></canvas>
          </div>
        </button>
      {/each}
    </div>
  </div>

  <!-- Charts View -->
  <div class="mb-6">

      <!-- Selected Lift Chart -->
      {#if selectedLift && results[selectedLift]}
        {@const liftStats = summaryTable.find(row => row.lift === selectedLift)}
        <div class="bg-white rounded-lg shadow-md p-3 sm:p-6">
          <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between mb-4 gap-2">
            <h2 class="text-xl sm:text-2xl font-bold text-gray-800">{liftNames[selectedLift] || selectedLift}</h2>
            {#if trendlineEquations[selectedLift]}
              <div class="text-xs sm:text-sm text-gray-600 bg-gray-100 px-2 sm:px-3 py-1 rounded-full">
                {trendlineEquations[selectedLift]}
              </div>
            {/if}
          </div>
          
          <!-- Stats Cards -->
          {#if liftStats}
            <div class="grid grid-cols-2 lg:grid-cols-4 gap-2 sm:gap-4 mb-4 sm:mb-6">
              <div class="bg-gradient-to-br from-blue-50 to-blue-100 rounded-lg p-3 sm:p-4 border border-blue-200">
                <div class="text-xs font-medium text-blue-600 mb-1">Current 6RM</div>
                <div class="text-lg sm:text-2xl font-bold text-blue-900">{liftStats.avg6RM_recent.toFixed(0)} lbs</div>
              </div>
              <div class="bg-gradient-to-br {liftStats.pct_increase > 0 ? 'from-green-50 to-green-100' : liftStats.pct_increase < 0 ? 'from-red-50 to-red-100' : 'from-gray-50 to-gray-100'} rounded-lg p-3 sm:p-4 border {liftStats.pct_increase > 0 ? 'border-green-200' : liftStats.pct_increase < 0 ? 'border-red-200' : 'border-gray-200'}">
                <div class="text-xs font-medium {liftStats.pct_increase > 0 ? 'text-green-600' : liftStats.pct_increase < 0 ? 'text-red-600' : 'text-gray-600'} mb-1">2-Week Change</div>
                <div class="text-lg sm:text-2xl font-bold {liftStats.pct_increase > 0 ? 'text-green-900' : liftStats.pct_increase < 0 ? 'text-red-900' : 'text-gray-900'}">
                  {liftStats.pct_increase > 0 ? '+' : ''}{liftStats.pct_increase.toFixed(1)}%
                </div>
              </div>
              <div class="bg-gradient-to-br {liftStats.pct_increase_6mo > 0 ? 'from-green-50 to-green-100' : liftStats.pct_increase_6mo < 0 ? 'from-red-50 to-red-100' : 'from-gray-50 to-gray-100'} rounded-lg p-3 sm:p-4 border {liftStats.pct_increase_6mo > 0 ? 'border-green-200' : liftStats.pct_increase_6mo < 0 ? 'border-red-200' : 'border-gray-200'}">
                <div class="text-xs font-medium {liftStats.pct_increase_6mo > 0 ? 'text-green-600' : liftStats.pct_increase_6mo < 0 ? 'text-red-600' : 'text-gray-600'} mb-1">6-Month Change</div>
                <div class="text-lg sm:text-2xl font-bold {liftStats.pct_increase_6mo > 0 ? 'text-green-900' : liftStats.pct_increase_6mo < 0 ? 'text-red-900' : 'text-gray-900'}">
                  {liftStats.pct_increase_6mo > 0 ? '+' : ''}{liftStats.pct_increase_6mo.toFixed(1)}%
                </div>
              </div>
              <div class="bg-gradient-to-br from-purple-50 to-purple-100 rounded-lg p-3 sm:p-4 border border-purple-200">
                <div class="text-xs font-medium text-purple-600 mb-1">6-Month Gain</div>
                <div class="text-lg sm:text-2xl font-bold text-purple-900">
                  {liftStats.raw_increase_6mo > 0 ? '+' : ''}{liftStats.raw_increase_6mo.toFixed(1)} lbs
                </div>
              </div>
            </div>
          {/if}

          {#key selectedLift + timeframe}
            <div class="h-64 sm:h-80 lg:h-96">
              <canvas id={'chart-' + selectedLift}></canvas>
            </div>
          {/key}
        </div>

          <!-- Detailed Data Table -->
          <div class="bg-white rounded-lg shadow-md p-3 sm:p-6 mt-4">
            <h3 class="text-base sm:text-lg font-bold text-gray-800 mb-4">Workout History</h3>
            <div class="overflow-x-auto">
              <table class="w-full text-xs sm:text-sm">
                <thead>
                  <tr class="bg-gray-100 border-b">
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-left font-semibold text-gray-700">Date</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700">Vol (kg)</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700">Reps</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700">Sets</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700 hidden sm:table-cell">Avg Wt (lbs)</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700 hidden sm:table-cell">Avg Reps</th>
                    <th class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-700">6RM (lbs)</th>
                  </tr>
                </thead>
                <tbody>
                  {#each results[selectedLift] as row}
                    <tr class="border-b hover:bg-gray-50">
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-gray-900 text-xs sm:text-sm">{row[0]}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right text-gray-700">{row[1].toFixed(1)}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right text-gray-700">{row[2]}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right text-gray-700">{row[3]}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right text-gray-700 hidden sm:table-cell">{row[5].toFixed(1)}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right text-gray-700 hidden sm:table-cell">{row[6].toFixed(1)}</td>
                      <td class="px-2 sm:px-4 py-2 sm:py-3 text-right font-semibold text-gray-900">{row[8].toFixed(1)}</td>
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>
          </div>
        {/if}
    </div>
</div>
