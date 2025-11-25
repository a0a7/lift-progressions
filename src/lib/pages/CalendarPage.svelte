<script lang="ts">
    import { onMount } from 'svelte';
    import { format, eachDayOfInterval, startOfWeek, endOfWeek, isSameDay, subWeeks, getYear, startOfYear, endOfYear } from 'date-fns';

    export let activities: any[] = [];
    export let splitConfig: any;
    export let determineSplitDay: (exerciseSets: any[]) => { splitType: string | null; confidence: number };

    let heatmapDataByYear: { [year: number]: { date: Date, splitType: string | null, confidence: number, totalSets: number, totalMinutes: number }[] } = {};
    let weeksByYear: { [year: number]: Date[][] } = {};
    let monthLabelsByYear: { [year: number]: { month: string, index: number }[] } = {};
    let past52Weeks: Date[][] = [];
    let past52WeeksMonthLabels: { month: string, index: number }[] = [];
    const weekdays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

    let showAllYears = false;
    let years: number[] = [];
    let hoveredDate: Date | null = null;
    let tooltipPosition = { x: 0, y: 0 };
    let viewMode: 'split' | 'volume' | 'time' = 'split';

    // Transform activities to match expected format
    let transformedActivities: any[] = [];

    $: {
        transformedActivities = activities.map(activity => ({
            startTime: activity.start_time,
            exerciseSets: activity.exercise_sets || [],
        }));
        
        // Regenerate calendar data when activities change
        if (transformedActivities.length > 0) {
            years = Array.from(new Set(transformedActivities.map(a => getYear(new Date(a.startTime)))));
            if (years.length > 0) {
                generateHeatmapDataForYear(years[0]);
            }
            generatePast52WeeksData();
        }
    }

    onMount(() => {
        if (transformedActivities.length > 0) {
            years = Array.from(new Set(transformedActivities.map(a => getYear(new Date(a.startTime)))));
            if (years.length > 0) {
                generateHeatmapDataForYear(years[0]);
            }
            generatePast52WeeksData();
        }
    });

    function generateHeatmapDataForYear(year: number) {
        const startDate = startOfYear(new Date(year, 0, 1));
        const endDate = endOfYear(new Date(year, 0, 1));
        const dates = eachDayOfInterval({ start: startDate, end: endDate });

        heatmapDataByYear[year] = dates.map(date => {
            const dayActivities = transformedActivities.filter(a => isSameDay(new Date(a.startTime), date));
            const allSets = dayActivities.flatMap(a => a.exerciseSets || []);
            const result = determineSplitDay(allSets);
            const totalSets = allSets.length;
            
            // Calculate workout duration in minutes
            let totalMinutes = 0;
            if (allSets.length > 0) {
                const times = allSets.map(s => new Date(s.start_time).getTime()).sort((a, b) => a - b);
                totalMinutes = Math.round((times[times.length - 1] - times[0]) / 60000);
            }
            
            return {
                date,
                splitType: result.splitType,
                confidence: result.confidence,
                totalSets,
                totalMinutes
            };
        });

        weeksByYear[year] = [];
        let currentWeek: Date[] = [];
        dates.forEach(date => {
            if (currentWeek.length === 0 || isSameDay(startOfWeek(date), startOfWeek(currentWeek[0]))) {
                currentWeek.push(date);
            } else {
                weeksByYear[year].push(currentWeek);
                currentWeek = [date];
            }
        });
        if (currentWeek.length > 0) {
            weeksByYear[year].push(currentWeek);
        }

        monthLabelsByYear[year] = [];
        dates.forEach((date, index) => {
            if (date.getDate() === 1) {
                monthLabelsByYear[year].push({ month: format(date, 'MMM'), index });
            }
        });
    }

    function generatePast52WeeksData() {
        const endDate = endOfWeek(new Date());
        const startDate = subWeeks(endDate, 51);
        const dates = eachDayOfInterval({ start: startDate, end: endDate });

        past52Weeks = [];
        let currentWeek: Date[] = [];
        dates.forEach(date => {
            if (currentWeek.length === 0 || isSameDay(startOfWeek(date), startOfWeek(currentWeek[0]))) {
                currentWeek.push(date);
            } else {
                past52Weeks.push(currentWeek);
                currentWeek = [date];
            }
        });
        if (currentWeek.length > 0) {
            past52Weeks.push(currentWeek);
        }

        past52WeeksMonthLabels = [];
        dates.forEach((date, index) => {
            if (date.getDate() === 1) {
                past52WeeksMonthLabels.push({ month: format(date, 'MMM'), index });
            }
        });
    }

    function getColor(splitType: string | null, totalSets: number = 0): string {
        // Rest day (no sets at all)
        if (totalSets === 0) {
            return 'bg-gray-100';
        }
        
        // Has sets but no recognized split type (imperfect data)
        if (!splitType) {
            return 'bg-gray-400';
        }
        
        const color = splitConfig[splitType]?.color;
        
        // Map color names to Tailwind classes
        const colorMap: Record<string, string> = {
            'red': 'bg-red-600',
            'blue': 'bg-blue-600',
            'green': 'bg-green-600',
            'green-yellow': 'bg-lime-500',
            'yellow': 'bg-yellow-500',
            'teal': 'bg-teal-600',
            'orange': 'bg-orange-600',
            'orange-yellow': 'bg-amber-500',
            'purple': 'bg-purple-600',
            'pink': 'bg-pink-600',
            'indigo': 'bg-indigo-600',
            'cyan': 'bg-cyan-600',
            'lime': 'bg-lime-600',
        };
        
        return colorMap[color] || 'bg-white';
    }

    function getVolumeColor(totalSets: number): string {
        // Rest day (no sets)
        if (totalSets === 0) {
            return 'white';
        }
        
        // Create continuous color gradient: dark blue -> red -> yellow
        // Max out at 25 sets for color purposes
        const maxSets = 25;
        const normalizedSets = Math.min(totalSets / maxSets, 1);
        
        let r: number, g: number, b: number;
        
        if (normalizedSets < 0.6) {
            // Dark blue (30, 58, 138) to Red (220, 38, 38) (0-0.6)
            const t = normalizedSets / 0.6;
            r = Math.round(56 + (220 - 56) * t);
            g = Math.round(78 + (38 - 78) * t);
            b = Math.round(220 + (38 - 220) * t);
        } else {
            // Red (220, 38, 38) to Yellow (250, 204, 21) (0.6-1.0)
            const t = (normalizedSets - 0.6) / 0.4;
            r = Math.round(220 + (250 - 220) * t);
            g = Math.round(38 + (204 - 38) * t);
            b = Math.round(38 + (21 - 38) * t);
        }
        
        return `rgb(${r}, ${g}, ${b})`;
    }

    function getTimeColor(totalMinutes: number): string {
        // Rest day (no time)
        if (totalMinutes === 0) {
            return 'white';
        }
        
        // Create continuous color gradient: dark blue -> red -> yellow
        // Max out at 120 minutes (2 hours) for color purposes
        const maxMinutes = 120;
        const normalizedMinutes = Math.min(totalMinutes / maxMinutes, 1);
        
        let r: number, g: number, b: number;
        
        if (normalizedMinutes < 0.6) {
            // Dark blue (56, 78, 220) to Red (220, 38, 38) (0-0.6)
            const t = normalizedMinutes / 0.6;
            r = Math.round(56 + (220 - 56) * t);
            g = Math.round(78 + (38 - 78) * t);
            b = Math.round(220 + (38 - 220) * t);
        } else {
            // Red (220, 38, 38) to Yellow (250, 204, 21) (0.6-1.0)
            const t = (normalizedMinutes - 0.6) / 0.4;
            r = Math.round(220 + (250 - 220) * t);
            g = Math.round(38 + (204 - 38) * t);
            b = Math.round(38 + (21 - 38) * t);
        }
        
        return `rgb(${r}, ${g}, ${b})`;
    }

    function getOpacity(totalSets: number, confidence: number = 100): number {
        if (totalSets === 0) return 10;
        
        const maxSets = 40;
        let opacity = Math.round(Math.min(((totalSets / maxSets) * 10), 10)) * 10;
        opacity = Math.max(opacity, 30); // Minimum 30% opacity if there are sets
                
        return opacity;
    }

    function getVolumeOpacity(totalSets: number): number {
        if (totalSets === 0) return 10;
        
        // For volume view, use full opacity to show the color gradient clearly
        const maxSets = 30;
        let opacity = Math.round(Math.min(((totalSets / maxSets) * 10), 10)) * 10;
        opacity = Math.max(opacity, 60); // Higher minimum opacity for volume view
        
        return opacity;
    }

    function loadMoreYears() {
        years.slice(1,).forEach(year => {
            generateHeatmapDataForYear(year);
        });
        showAllYears = true;
    }

    function handleMouseOver(event: MouseEvent, date: Date) {
        hoveredDate = date;
        tooltipPosition = {
            x: event.clientX + 10,
            y: event.clientY + 10
        };
    }

    function handleMouseOut() {
        hoveredDate = null;
    }
</script>

<div class="pb-10">
    <!-- View Mode Toggle -->
    <div class="flex justify-center mb-4">
        <div class="inline-flex rounded-md shadow-sm" role="group">
            <button
                class="px-4 py-2 text-sm font-medium {viewMode === 'split' ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 hover:bg-gray-50'} border border-gray-300 rounded-l-md"
                on:click={() => viewMode = 'split'}
            >
                Split Type
            </button>
            <button
                class="px-4 py-2 text-sm font-medium {viewMode === 'volume' ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 hover:bg-gray-50'} border-t border-b border-gray-300"
                on:click={() => viewMode = 'volume'}
            >
                Volume (Sets)
            </button>
            <button
                class="px-4 py-2 text-sm font-medium {viewMode === 'time' ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 hover:bg-gray-50'} border border-gray-300 rounded-r-md"
                on:click={() => viewMode = 'time'}
            >
                Time
            </button>
        </div>
    </div>
    
    <div class="mx-auto w-fit">
        {#if viewMode === 'split'}
            <div class="flex justify-center flex-wrap gap-x-4 gap-y-2 pt-4 mb-4 max-w-4xl">
                {#each Object.entries(splitConfig) as [splitName, splitData]}
                    <div class="flex items-center">
                        <div class="w-[12px] h-[12px] {getColor(splitName, 1)} rounded-[2px] mr-2"></div>
                        <span class="text-sm">{splitName.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase())}</span>
                    </div>
                {/each}
                <div class="flex items-center">
                    <div class="w-[12px] h-[12px] bg-black rounded-[2px] mr-2"></div>
                    <span class="text-sm">No Split</span>
                </div>
                <div class="flex items-center">
                    <div class="w-[12px] h-[12px] bg-gray-200 rounded-[2px] mr-2"></div>
                    <span class="text-sm">Rest</span>
                </div>
            </div>
        {:else if viewMode === 'volume'}
            <div class="flex justify-center items-center gap-x-4 gap-y-2 pt-4 mb-4 max-w-4xl">
                <div class="flex items-center gap-2">
                    <span class="text-sm">0 sets</span>
                    <div class="w-[200px] h-[12px] rounded-[2px]" 
                         style="background: linear-gradient(to right, rgb(56,78,220), rgb(220,38,38), rgb(250,204,21))">
                    </div>
                    <span class="text-sm">25+ sets</span>
                </div>
                <div class="flex items-center">
                    <div class="w-[12px] h-[12px] bg-white rounded-[2px] mr-2 border border-gray-300"></div>
                    <span class="text-sm">Rest Day</span>
                </div>
            </div>
        {:else if viewMode === 'time'}
            <div class="flex justify-center items-center gap-x-4 gap-y-2 pt-4 mb-4 max-w-4xl">
                <div class="flex items-center gap-2">
                    <span class="text-sm">0 min</span>
                    <div class="w-[200px] h-[12px] rounded-[2px]" 
                         style="background: linear-gradient(to right, rgb(56,78,220), rgb(220,38,38), rgb(250,204,21))">
                    </div>
                    <span class="text-sm">120+ min</span>
                </div>
                <div class="flex items-center">
                    <div class="w-[12px] h-[12px] bg-white rounded-[2px] mr-2 border border-gray-300"></div>
                    <span class="text-sm">Rest Day</span>
                </div>
            </div>
        {/if}

        <h2 class="text-xl sm:text-2xl text-center font-black pb-3 pt-4">Last Year</h2>
        <div class="overflow-x-auto pb-2 -mx-2 px-2">
            <div class="inline-block min-w-fit">
                <div class="flex scale-90 sm:scale-95 origin-top-left">
                    <div class="mt-[1px] text-right pr-1">   
                        {#each weekdays as weekday}
                            <div class="text-[9.5px] -my-[1.1px]">{weekday}</div>
                        {/each}
                    </div>
                    {#each past52Weeks as week}
                        <div class="mx-[1px] {week === past52Weeks[0] ? 'mt-auto' : ''}">
                            {#each week as date}
                                {@const dayActivities = transformedActivities.filter(a => isSameDay(new Date(a.startTime), date))}
                                {@const allSets = dayActivities.flatMap(a => a.exerciseSets || [])}
                                {@const result = determineSplitDay(allSets)}
                                {@const times = allSets.map(s => new Date(s.start_time).getTime()).sort((a, b) => a - b)}
                                {@const totalMinutes = times.length > 0 ? Math.round((times[times.length - 1] - times[0]) / 60000) : 0}
                                {@const volumeColor = getVolumeColor(allSets.length)}
                                {@const timeColor = getTimeColor(totalMinutes)}
                                <div 
                                    class="w-[12px] h-[12px] my-[1px] rounded-[2px] shadow-slate-900/10 dark:shadow-slate-900/50 shadow-inner {viewMode === 'split' ? getColor(result.splitType, allSets.length) : ''}"
                                    style="{viewMode === 'volume' ? `background-color: ${volumeColor}` : viewMode === 'time' ? `background-color: ${timeColor}` : ''}"
                                    title="{format(date, 'yyyy-MM-dd')}"
                                    on:mouseover={(e) => handleMouseOver(e, date)}
                                    on:mouseout={handleMouseOut}
                                    role="button"
                                    tabindex="0">
                                </div>
                            {/each}
                        </div>
                    {/each}
                </div>
                
                <div class="flex px-1 scale-90 sm:scale-95 origin-top-left">
                    {#each past52WeeksMonthLabels as { month, index }}
                        <div class="mx-[25px] flex w-[13px] text-[9px]" style="grid-column-start: {index + 2}">{month}</div>
                    {/each}
                </div>
            </div>
        </div>
    </div>
    <div class="pt-4">
        <h2 class="text-xl sm:text-2xl font-black text-center pb-2">By Year</h2>
        {#each (Object.keys(heatmapDataByYear)).reverse() as year}
            {#if String(year) === String(years[0]) || showAllYears}
                <div class="mb-4">
                    <h3 class="text-xs font-black pt-2 pb-1 text-center sm:text-left sm:pl-8">{year}</h3>
                    <div class="overflow-x-auto pb-2 -mx-2 px-2">
                        <div class="inline-block min-w-fit">
                            <div class="flex scale-90 sm:scale-95 origin-top-left">
                                <div class="mt-[1px] text-right pr-1">   
                                    {#each weekdays as weekday}
                                        <div class="text-[9.5px] -my-[1.1px]">{weekday}</div>
                                    {/each}
                                </div>
                                {#each weeksByYear[year] as week}
                                    <div class="mx-[1px] {week === weeksByYear[year][0] ? 'mt-auto' : ''}">
                                        {#each week as date}
                                            {@const activity = heatmapDataByYear[year].find(d => isSameDay(d.date, date))}
                                            {@const volumeColor = getVolumeColor(activity?.totalSets || 0)}
                                            {@const timeColor = getTimeColor(activity?.totalMinutes || 0)}
                                            <div 
                                                class="w-[12px] h-[12px] my-[1px] rounded-[2px] shadow-slate-900/10 dark:shadow-slate-900/50 shadow-inner {viewMode === 'split' ? getColor(activity?.splitType, activity?.totalSets || 0) : ''}"
                                                style="{viewMode === 'volume' ? `background-color: ${volumeColor}` : viewMode === 'time' ? `background-color: ${timeColor}` : ''}"
                                                title="{format(date, 'yyyy-MM-dd')}"
                                                on:mouseover={(e) => handleMouseOver(e, date)}
                                                on:mouseout={handleMouseOut}
                                                role="button"
                                                tabindex="0">
                                            </div>
                                        {/each}
                                    </div>
                                {/each}
                            </div>
                            <div class="flex scale-90 sm:scale-95 origin-top-left">
                                {#each monthLabelsByYear[year] as { month, index }}
                                    <div class="mx-[24.5px] flex w-[13px] text-[9px]" style="grid-column-start: {index + 2}">{month}</div>
                                {/each}
                            </div>
                        </div>
                    </div>
                </div>
            {/if}
        {/each}
        {#if !showAllYears && years.length > 1}
            <div class="flex justify-center pt-4">
                <button 
                    class="px-4 py-2 border border-gray-300 rounded hover:bg-gray-100"
                    on:click={loadMoreYears}>
                    Load More
                </button>
            </div>
        {/if}
    </div>
</div>

{#if hoveredDate}
    {@const dayActivities = transformedActivities.filter(a => isSameDay(new Date(a.startTime), hoveredDate || new Date()))}
    {@const allSets = dayActivities.flatMap(a => a.exerciseSets || [])}
    {@const result = determineSplitDay(allSets)}
    {@const times = allSets.map(s => new Date(s.start_time).getTime()).sort((a, b) => a - b)}
    {@const totalMinutes = times.length > 0 ? Math.round((times[times.length - 1] - times[0]) / 60000) : 0}
    <div
        class="fixed bg-white bg-opacity-90 p-2 rounded shadow-lg z-50 text-md font-medium text-gray-800 whitespace-nowrap"
        style="top: {tooltipPosition.y}px; left: {tooltipPosition.x}px;">
        <h4 class="m-0 mb-1 font-bold">{hoveredDate.toLocaleDateString('en-US', {weekday: "long",year: "numeric",month: "long",day: "numeric",})}</h4>
        {#if allSets.length > 0}
            <p class="m-0">
                <b>{result.splitType ? result.splitType.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase()) : 'Unknown'} Day</b>
                {#if result.confidence < 100}
                    <span class="text-gray-600">({result.confidence}% confidence)</span>
                {/if}
                <br>
                <b>{allSets.length} sets</b> • <b>{totalMinutes} min</b>
            </p>
        {:else}
            <p class="m-0">Rest Day</p>
        {/if}
    </div>
{/if}
