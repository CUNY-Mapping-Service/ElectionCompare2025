<script setup lang="ts">
import { computed, onMounted, ref, useTemplateRef, watch } from 'vue';
import * as d3 from 'd3';

const formatPercent = d3.format('.1f');
const formatVotes = d3.format(',.0f');
const formatVotesCompact = d3.format('.3~s');

interface FeatureProperties {
    [key: string]: any;
}

interface Props {
    hoveredData: {
        id: string | number;
        properties: FeatureProperties;
    } | null;
    COLOR_SCALE: any;
    idKey: string;
    filteredFeatures?: any[]; // Array of all filtered district features
    selectedFilters?: Array<{ column: string; label: string; short_label: string }>;
    allFeatures?: any[]; // Array of all features for citywide calculations
}

const scannerPercent = 97.2
const props = defineProps<Props>();
const emit = defineEmits<{
    close: []
}>();

const edLabel = computed(() => {
    if (!props.hoveredData) return '';

    const id = props.hoveredData?.properties[props.idKey]
    const ntaName = props.hoveredData?.properties.ntashort

    if (id) {
        const ad = String(id).slice(0, 2)
        const ed = Number(String(id).slice(2))
        const baseLabel = `ED ${ed} in AD ${ad}`
        // return ntaName ? `${baseLabel} (${ntaName})` : baseLabel
        return ntaName ? `${ntaName}: ${baseLabel}` : baseLabel

    }
    return ''
})

const isVisible = computed(() => true); // Always show - display citywide by default
const hasFilteredData = computed(() => props.filteredFeatures && props.filteredFeatures.length > 0);
const hasCitywideData = computed(() => props.allFeatures && props.allFeatures.length > 0);
const showCitywideByDefault = computed(() => !props.hoveredData && !hasFilteredData.value && hasCitywideData.value);

// Compute filter labels for display
const filterLabelsText = computed(() => {
    if (!props.selectedFilters || props.selectedFilters.length === 0) return '';
    return props.selectedFilters.map(f => f.label).join(' AND ');
});

// Chart refs
const chart25Ref = useTemplateRef('chart25');
const chart21Ref = useTemplateRef('chart21');
const aggChart25Ref = useTemplateRef('aggChart25');
const aggChart21Ref = useTemplateRef('aggChart21');
const containerRef = useTemplateRef('container');

// Track container width for responsive sizing
const containerWidth = ref(320);
const isMobile = ref(false);

// Track scale mode: 'percentage' or 'votes'
const scaleMode = ref<'percentage' | 'votes'>('percentage');

// Aggregate data computation for filtered districts
const aggregateData25 = computed(() => {
    if (!hasFilteredData.value) return [];

    const candidates = props.COLOR_SCALE.candidates;
    const totals: { [key: string]: { votes: number, name: string, color: string } } = {};

    candidates.forEach((c: any) => {
        totals[c.id] = { votes: 0, name: c.label, color: c.colors[3] };
    });

    // Add 'other' to totals
    const otherScale = props.COLOR_SCALE.other;

    totals[otherScale.id] = {
        votes: 0,
        name: otherScale.label,
        color: otherScale.colors[3]
    };


    props.filteredFeatures!.forEach(feature => {
        candidates.forEach((c: any) => {
            if (totals[c.id]) {
                // @ts-ignore
                totals[c.id].votes += feature.properties[c.id] || 0;
            }
        });
        // Add other votes
        // @ts-ignore
        totals[otherScale.id].votes += feature.properties[otherScale.id] || 0;
    });

    const totalVotes = Object.values(totals).reduce((sum, c) => sum + c.votes, 0);

    return Object.values(totals)
        .map(c => ({
            name: c.name,
            votes: c.votes,
            percentage: totalVotes > 0 ? (c.votes / totalVotes) * 100 : 0,
            color: c.color
        }))
        .filter(d => d.votes > 0)
        .sort((a, b) => b.percentage - a.percentage);
});

const aggregateData21 = computed(() => {
    if (!hasFilteredData.value) return [];

    const totals = {
        ea: { votes: 0, name: 'Eric Adams', color: props.COLOR_SCALE.candidates[3].colors[3] },
        cs: { votes: 0, name: 'Curtis Sliwa', color: props.COLOR_SCALE.candidates[2].colors[3] },
        other: { votes: 0, name: 'Other', color: props.COLOR_SCALE.other.colors[3] }
    };

    props.filteredFeatures!.forEach(feature => {
        totals.ea.votes += feature.properties.gen21ea || 0;
        totals.cs.votes += feature.properties.gen21cs || 0;
        totals.other.votes += feature.properties.gen21othr || 0;
    });

    const totalVotes = Object.values(totals).reduce((sum, c) => sum + c.votes, 0);

    return Object.values(totals)
        .map(c => ({
            name: c.name,
            votes: c.votes,
            percentage: totalVotes > 0 ? (c.votes / totalVotes) * 100 : 0,
            color: c.color
        }))
        .filter(d => d.votes > 0)
        .sort((a, b) => b.percentage - a.percentage);
});

// Citywide data computation for all features
const citywideData25 = computed(() => {
    if (!props.allFeatures || props.allFeatures.length === 0) return [];

    const candidates = props.COLOR_SCALE.candidates;
    const totals: { [key: string]: { votes: number, name: string, color: string } } = {};

    candidates.forEach((c: any) => {
        totals[c.id] = { votes: 0, name: c.label, color: c.colors[3] };
    });

    // Add 'other' to totals
    const otherScale = props.COLOR_SCALE.other;
    totals[otherScale.id] = {
        votes: 0,
        name: props.COLOR_SCALE.other.label,
        color: props.COLOR_SCALE.other.colors[3]
    };

    props.allFeatures.forEach(feature => {
        candidates.forEach((c: any) => {
            if (totals[c.id]) {
                // @ts-ignore
                totals[c.id].votes += feature.properties[c.id] || 0;
            }
        });
        // Add other votes
        // @ts-ignore
        totals[otherScale.id].votes += feature.properties[props.COLOR_SCALE.other.id] || 0;
    });

    const totalVotes = Object.values(totals).reduce((sum, c) => sum + c.votes, 0);

    return Object.values(totals)
        .map(c => ({
            name: c.name,
            votes: c.votes,
            percentage: totalVotes > 0 ? (c.votes / totalVotes) * 100 : 0,
            color: c.color
        }))
        .filter(d => d.votes > 0)
        .sort((a, b) => b.percentage - a.percentage);
});

const citywideData21 = computed(() => {
    if (!props.allFeatures || props.allFeatures.length === 0) return [];

    const totals = {
        ea: { votes: 0, name: 'Eric Adams', color: props.COLOR_SCALE.candidates[3].colors[3] },
        cs: { votes: 0, name: 'Curtis Sliwa', color: props.COLOR_SCALE.candidates[2].colors[3] },
        other: { votes: 0, name: 'Other', color: props.COLOR_SCALE.other.colors[3] }
    };

    props.allFeatures.forEach(feature => {
        totals.ea.votes += feature.properties.gen21ea || 0;
        totals.cs.votes += feature.properties.gen21cs || 0;
        totals.other.votes += feature.properties.gen21othr || 0;
    });

    const totalVotes = Object.values(totals).reduce((sum, c) => sum + c.votes, 0);

    return Object.values(totals)
        .map(c => ({
            name: c.name,
            votes: c.votes,
            percentage: totalVotes > 0 ? (c.votes / totalVotes) * 100 : 0,
            color: c.color
        }))
        .filter(d => d.votes > 0)
        .sort((a, b) => b.percentage - a.percentage);
});

// Prepare data for 2025 election - using percentage data
const data25 = computed(() => {
    if (!props.hoveredData) return [];

    const candidates = props.COLOR_SCALE.candidates;

    return [...candidates
        .map((c: any) => ({
            name: c.label,
            percentage: props.hoveredData!.properties[`${c.id}_pct`] || 0,
            votes: props.hoveredData!.properties[c.id] || 0,
            color: c.colors[3],
        })),
    {
        name: props.COLOR_SCALE.other.label,
        percentage: props.hoveredData!.properties[`${props.COLOR_SCALE.other.id}_pct`] || 0,
        votes: props.hoveredData!.properties[props.COLOR_SCALE.other.id] || 0,
        color: props.COLOR_SCALE.other.colors[3],
    }]
        .filter((d: any) => d.votes > 0)
        .sort((a: any, b: any) => b.percentage - a.percentage);
});

// Prepare data for 2021 election - using percentage data
const data21 = computed(() => {
    if (!props.hoveredData) return [];

    return [
        {
            name: 'Eric Adams',
            percentage: props.hoveredData.properties.gen21ea_pct || 0,
            votes: props.hoveredData.properties.gen21ea || 0,
            color: props.COLOR_SCALE.candidates[3].colors[3],
        },
        {
            name: 'Curtis Sliwa',
            percentage: props.hoveredData.properties.gen21cs_pct || 0,
            votes: props.hoveredData.properties.gen21cs || 0,
            color: props.COLOR_SCALE.candidates[2].colors[3],
        },
        {
            name: 'Other',
            percentage: props.hoveredData.properties.gen21othr_pct || 0,
            votes: props.hoveredData.properties.gen21othr || 0,
            color: props.COLOR_SCALE.other.colors[3],
        }
    ]
        .filter((d: any) => d.votes > 0)
        .sort((a: any, b: any) => b.percentage - a.percentage);
});

// Copy table to clipboard
const copyTableToClipboard = async (data: any[], includeCitywide: boolean = false) => {
    let headers: string[];
    let rows: string[];

    if (includeCitywide) {
        // Determine view type based on filtered data
        const viewType = hasFilteredData.value ? 'Filtered %' : 'Citywide %';
        headers = ['Candidate', viewType, 'Votes'];
        rows = data.map(d => {
            return `${d.name}\t${formatPercent(d.percentage)}%\t${formatVotes(d.votes)}`;
        });
    } else {
        // For citywide only view (shouldn't be used anymore)
        headers = ['Candidate', 'Citywide %'];
        rows = data.map(d => `${d.name}\t${formatPercent(d.percentage)}%`);
    }

    const text = [headers.join('\t'), ...rows].join('\n');
    await navigator.clipboard.writeText(text);
    // include allow="clipboard-write" in the <iframe> tag

};

// Draw stacked bar chart with responsive width
const drawStackedChart = (ref: any, data: any[], maxVotes: number) => {
    if (!ref) return;

    // Clear previous content
    d3.select(ref).selectAll("*").remove();

    // If no data, exit early after clearing
    if (data.length === 0) return;

    const margin = { top: 0, right: 30, bottom: 0, left: 0 };
    const width = containerWidth.value - margin.left - margin.right;
    const height = 12 * 2;

    const svg = d3.select(ref)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

    // Prepare segments based on scale mode
    let segments;
    let x;

    if (scaleMode.value === 'percentage') {
        // Use percentages
        let cumulativePercent = 0;
        segments = data.map((d: any) => {
            const start = cumulativePercent;
            cumulativePercent += d.percentage;
            return { ...d, start, value: d.percentage };
        });

        x = d3.scaleLinear()
            .domain([0, 100])
            .range([0, width]);
    } else {
        // Use votes
        let cumulativeVotes = 0;
        segments = data.map((d: any) => {
            const start = cumulativeVotes;
            cumulativeVotes += d.votes;
            return { ...d, start, value: d.votes };
        });

        x = d3.scaleLinear()
            .domain([0, maxVotes])
            .range([0, width]);
    }

    // Draw stacked bar
    svg.selectAll('.segment')
        .data(segments)
        .enter()
        .append('rect')
        .attr('class', 'segment')
        .attr('x', (d: any) => x(d.start))
        .attr('y', 0)
        .attr('width', (d: any) => x(d.value))
        .attr('height', height)
        .attr('fill', (d: any) => d3.rgb(d.color).brighter(0.4).toString())
        .attr('stroke', '#333')
        .attr('stroke-width', 1)
        .append('title')
        .text((d: any, i: number) => {
            if (scaleMode.value === 'percentage') {
                return d.name + ' : '  + formatPercent(d.percentage) + (i >= 0 ? '%' : '');
            } else {
                return d.name + ' has votes: ' + formatVotes(d.votes);
            }
        });

    // Labels below bar
    svg.selectAll('.bar-label')
        .data(segments)
        .enter()
        .append('text')
        .attr('class', 'bar-label')
        .attr('x', (d: any) => x(d.start) + 1)
        .attr('y', (d: any, i: number) => 12)
        .attr('text-anchor', 'start')
        .attr('dy', '0.35em')
        .attr('font-size', '12px')
        .attr('font-family', 'monospace')
        .text((d: any, i: number) => {
            // hide label if less than 5%
            if (i !== 0 && d.percentage < 5) return '';
            if (scaleMode.value === 'percentage') {
                return formatPercent(d.percentage) + (i === 0 ? '%' : '');
            } else {
                return i === 0 ? 'Votes:' + formatVotesCompact(d.votes) : formatVotesCompact(d.votes);
            }
        });
};

// Update container width on mount and window resize
const updateContainerWidth = () => {
    if (containerRef.value) {
        const element = containerRef.value as HTMLElement;
        containerWidth.value = element.offsetWidth - 30; // 30 for padding
        isMobile.value = window.innerWidth <= 600;
    }
};

// Debounce function
const debounce = (func: () => void, wait: number) => {
    let timeout: ReturnType<typeof setTimeout>;
    return () => {
        clearTimeout(timeout);
        timeout = setTimeout(func, wait);
    };
};

// Redraw all charts
const redrawCharts = () => {
    updateContainerWidth();

    // Citywide default charts (when nothing is selected)
    if (showCitywideByDefault.value) {
        const maxVotes25 = d3.sum(citywideData25.value, (d: any) => d.votes);
        const maxVotes21 = d3.sum(citywideData21.value, (d: any) => d.votes);
        const maxVotesCitywide = Math.max(maxVotes25, maxVotes21);

        // Always call drawStackedChart
        if (aggChart25Ref.value) {
            drawStackedChart(aggChart25Ref.value, citywideData25.value, maxVotesCitywide);
        }
        if (aggChart21Ref.value) {
            drawStackedChart(aggChart21Ref.value, citywideData21.value, maxVotesCitywide);
        }
        return;
    }

    // Calculate max votes for ED charts
    const maxVotes25ED = data25.value.length > 0 ? d3.sum(data25.value, (d: any) => d.votes) : 0;
    const maxVotes21ED = data21.value.length > 0 ? d3.sum(data21.value, (d: any) => d.votes) : 0;
    const maxVotesED = Math.max(maxVotes25ED, maxVotes21ED);

    // Calculate max votes for aggregate charts
    const maxVotes25Agg = aggregateData25.value.length > 0 ? d3.sum(aggregateData25.value, (d: any) => d.votes) : 0;
    const maxVotes21Agg = aggregateData21.value.length > 0 ? d3.sum(aggregateData21.value, (d: any) => d.votes) : 0;
    const maxVotesAgg = Math.max(maxVotes25Agg, maxVotes21Agg);

    // ED charts
    if (chart25Ref.value) {
        drawStackedChart(chart25Ref.value, data25.value, maxVotesED);
    }
    if (chart21Ref.value) {
        drawStackedChart(chart21Ref.value, data21.value, maxVotesED);
    }

    // Aggregate charts
    if (hasFilteredData.value) {
        if (aggChart25Ref.value) {
            drawStackedChart(aggChart25Ref.value, aggregateData25.value, maxVotesAgg);
        }
        if (aggChart21Ref.value) {
            drawStackedChart(aggChart21Ref.value, aggregateData21.value, maxVotesAgg);
        }
    }
};

onMounted(() => {
    updateContainerWidth();

    // Debounced resize handler
    const debouncedResize = debounce(() => {
        redrawCharts();
    }, 100);

    window.addEventListener('resize', debouncedResize);

    // Cleanup
    return () => {
        window.removeEventListener('resize', debouncedResize);
    };
});

watch([data25, data21, aggregateData25, aggregateData21, citywideData25, citywideData21, containerWidth, isVisible, showCitywideByDefault, scaleMode], () => {
    if (!isVisible.value) return;

    // Wait a tick 
    requestAnimationFrame(() => {
        redrawCharts();
    });
}, { immediate: true });
</script>

<template>
    <div ref="container" class="infobox-container" v-if="isVisible">
        <!-- Citywide Default View - show when nothing is selected -->
        <div v-if="showCitywideByDefault" class="citywide-section">
            <div class="section-header">
                <p class="section-subtitle-text-body-1">Citywide Results - All Election Districts</p>
            </div>

            <!-- 2025 Citywide -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2025 General (<a href="https://vote.nyc/page/election-results-summary" target="_blank">certified results</a>)<br />
                        <!-- <span
                            style="font-weight: 200;">{{ scannerPercent }}% of scanners reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span> -->
                            </h3>
                <!--     <h3>2025 General (unofficial results)<br /><span
                            style="font-weight: 200;">More than {{ scannerPercent }} votes reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span></h3> -->
                </div>
                <div ref="aggChart25"></div>
                <div class="results-table-container">
                    <table class="results-table">
                        <thead>
                            <tr>
                                <th>Candidate</th>
                                <th>Citywide %</th>
                                <th>Votes</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(item, index) in citywideData25" :key="index">
                                <td>
                                    <div class="candidate-cell">
                                        <span class="color-indicator" :style="{ backgroundColor: item.color }"></span>
                                        <span>{{ item.name }}</span>
                                        <span v-if="index === 0" class="margin-indicator">
                                            +{{ Math.round((citywideData25[0]?.percentage || 0) -
                                                (citywideData25[1]?.percentage || 0)) }} points
                                        </span>
                                    </div>
                                </td>
                                <td class="percentage-cell citywide-text">{{ formatPercent(item.percentage) }}%</td>
                                <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                            </tr>
                        </tbody>
                        <tfoot>
                            <tr class="total-row">
                                <td>
                                    <button class="copy-button" @click="copyTableToClipboard(citywideData25, true)"
                                        title="Copy table to clipboard">
                                        📋
                                    </button>
                                    <strong>Total</strong>
                                </td>
                                <td></td>
                                <td class="votes-cell"><strong>{{formatVotes(d3.sum(citywideData25, (d: any) =>
                                    d.votes))}}</strong></td>
                            </tr>
                        </tfoot>
                    </table>

                </div>
            </div>

            <!-- 2021 Citywide -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2021 General (allocated to 2025 EDs)</h3>
                </div>
                <div ref="aggChart21"></div>
                <div class="results-table-container">
                    <table class="results-table">
                        <thead>
                            <tr>
                                <th>Candidate</th>
                                <th>Citywide %</th>
                                <th>Votes</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(item, index) in citywideData21" :key="index">
                                <td>
                                    <div class="candidate-cell">
                                        <span class="color-indicator" :style="{ backgroundColor: item.color }"></span>
                                        <span>{{ item.name }}</span>
                                        <span v-if="index === 0" class="margin-indicator">
                                            +{{ Math.round((citywideData21[0]?.percentage || 0) -
                                                (citywideData21[1]?.percentage || 0)) }} points
                                        </span>
                                    </div>
                                </td>
                                <td class="percentage-cell citywide-text">{{ formatPercent(item.percentage) }}%</td>
                                <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                            </tr>
                        </tbody>
                        <tfoot>
                            <tr class="total-row">
                                <td>
                                    <button class="copy-button" @click="copyTableToClipboard(citywideData21, true)"
                                        title="Copy table to clipboard">
                                        📋
                                    </button>
                                    <strong>Total</strong>
                                </td>
                                <td></td>
                                <td class="votes-cell"><strong>{{formatVotes(d3.sum(citywideData21, (d: any) =>
                                    d.votes))}}</strong></td>
                            </tr>
                            <tr class="footernote">
                                <td colspan="3">Totals may not add to 100% due to rounding.</td>
                            </tr>
                            <tr class="footernote">
                                <td colspan="3">Results omit write-in votes.</td>
                            </tr>
                        </tfoot>
                    </table>

                </div>
            </div>
        </div>

        <div class="header" v-if="hoveredData">
            <div class="header-content">
                <div>
                    <h3>SELECTED DISTRICT<br />{{ hasFilteredData && !hoveredData ? 'Filtered Districts' :
                        (hasFilteredData
                            ?
                            edLabel :
                            edLabel) }}</h3>
                </div>
                <button v-if="hoveredData" class="close-button" @click="emit('close')" aria-label="Close">
                    ×
                </button>
            </div>
            <!-- Individual ED Charts - show when a district is selected -->
            <div class="ed-section">
                <!-- 2025 Election Chart -->
                <div class="chart-section">
                    <div class="chart-title">
                    <h3>2025 General (<a href="https://vote.nyc/page/election-results-summary" target="_blank">certified results</a>)<br />
                        <!-- <span
                            style="font-weight: 200;">{{ scannerPercent }}% of scanners reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span> <br />-->
                            
                    <span style="font-weight: 100; font-size:.8em;">Districts with fewer than 5 votes omitted.</span>
                    </h3>
                    <!-- <h3>2025 General (unofficial results)<br /><span
                            style="font-weight: 200;">More than {{ scannerPercent }} votes reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span></h3> -->
                    </div>
                    <div ref="chart25"></div>
                    <div class="results-table-container">
                        <table class="results-table">
                            <thead>
                                <tr>
                                    <th>Candidate</th>
                                    <th>ED %</th>
                                    <th>Votes</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr v-for="(item, index) in data25" :key="index">
                                    <td>
                                        <div class="candidate-cell">
                                            <span class="color-indicator"
                                                :style="{ backgroundColor: item.color }"></span>
                                            <span>{{ item.name }}</span>
                                            <span v-if="index === 0 && data25.length > 1" class="margin-indicator">
                                                +{{ Math.round((data25[0]?.percentage || 0) - (data25[1]?.percentage ||
                                                    0)) }} points
                                            </span>
                                        </div>
                                    </td>
                                    <td class="percentage-cell">{{ formatPercent(item.percentage) }}%</td>
                                    <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                                </tr>
                            </tbody>
                            <tfoot>
                                <tr class="total-row">
                                    <td>
                                        <button class="copy-button" @click="copyTableToClipboard(data25, true)"
                                            title="Copy table to clipboard">
                                            📋
                                        </button>
                                        <strong>Total</strong>
                                    </td>
                                    <td></td>
                                    <td class="votes-cell"><strong>{{formatVotes(d3.sum(data25, (d: any) => d.votes))
                                            }}</strong></td>
                                </tr>
                            </tfoot>
                        </table>
                    </div>
                </div>

                <!-- 2021 Election Chart -->
                <div class="chart-section">
                    <div class="chart-title">
                        <h3>2021 General (allocated to 2025 EDs)</h3>
                    </div>
                    <div ref="chart21"></div>
                    <div class="results-table-container">
                        <table class="results-table">
                            <thead>
                                <tr>
                                    <th>Candidate</th>
                                    <th>ED %</th>
                                    <th>Votes</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr v-for="(item, index) in data21" :key="index">
                                    <td>
                                        <div class="candidate-cell">
                                            <span class="color-indicator"
                                                :style="{ backgroundColor: item.color }"></span>
                                            <span>{{ item.name }}</span>
                                            <span v-if="index === 0 && data21.length > 1" class="margin-indicator">
                                                +{{ Math.round((data21[0]?.percentage || 0) - (data21[1]?.percentage ||
                                                    0)) }} points
                                            </span>
                                        </div>
                                    </td>
                                    <td class="percentage-cell">{{ formatPercent(item.percentage) }}%</td>
                                    <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                                </tr>
                            </tbody>
                            <tfoot>
                                <tr class="total-row">
                                    <td>
                                        <button class="copy-button" @click="copyTableToClipboard(data21, true)"
                                            title="Copy table to clipboard">
                                            📋
                                        </button>
                                        <strong>Total</strong>
                                    </td>
                                    <td></td>
                                    <td class="votes-cell"><strong>{{formatVotes(d3.sum(data21, (d: any) => d.votes))
                                            }}</strong></td>
                                </tr>
                                <tr class="footernote">
                                    <td colspan="3">Totals may not add to 100% due to rounding.</td>
                                </tr>
                                <tr class="footernote">
                                    <td colspan="3">Results omit write-in votes.</td>
                                </tr>
                            </tfoot>
                        </table>
                    </div>
                </div>
            </div>
        </div>

        <!-- Aggregate Charts - show when filters are active -->
        <div v-if="hasFilteredData" class="aggregate-section">
            <div class="section-header">
                <p class="section-subtitle-text-body-1">Filtered citywide results ({{
                    filteredFeatures?.length.toLocaleString() }} election districts):</p>
                <p class="section-subtitle-text-body-2">{{ filterLabelsText }}</p>
            </div>

            <!-- Empty state message -->
            <div v-if="filteredFeatures && filteredFeatures.length === 0" class="empty-state">
                <p>No election districts match this selection.</p>
            </div>

            <!-- 2025 Aggregate -->
            <template v-if="filteredFeatures && filteredFeatures.length > 0">
                <div class="chart-section">
                    <div class="chart-title">
                    <h3>2025 General (<a href="https://vote.nyc/page/election-results-summary" target="_blank">certified results</a>)<br />
                        <!-- <span
                            style="font-weight: 200;">{{ scannerPercent }}% of scanners reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span> -->
                            </h3>
                   <!--  <h3>2025 General (unofficial results)<br /><span
                            style="font-weight: 200;">{{ scannerPercent }} votes reported, per <a href="https://web.enrboenyc.us/index.html" target="_blank">NYC BOE</a></span></h3> -->
                    </div>
                    <div ref="aggChart25"></div>
                    <div class="results-table-container">
                        <table class="results-table">
                            <thead>
                                <tr>
                                    <th>Candidate</th>
                                    <th>Filtered %</th>
                                    <th>Votes</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr v-for="(item, index) in aggregateData25" :key="index">
                                    <td>
                                        <div class="candidate-cell">
                                            <span class="color-indicator"
                                                :style="{ backgroundColor: item.color }"></span>
                                            <span>{{ item.name }}</span>
                                            <span v-if="index === 0 && aggregateData25.length > 1"
                                                class="margin-indicator">
                                                +{{ Math.round((aggregateData25[0]?.percentage || 0) -
                                                    (aggregateData25[1]?.percentage || 0)) }} points
                                            </span>
                                        </div>
                                    </td>
                                    <td class="percentage-cell">{{ formatPercent(item.percentage) }}%</td>
                                    <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                                </tr>
                            </tbody>
                            <tfoot>
                                <tr class="total-row">
                                    <td>
                                        <button class="copy-button" @click="copyTableToClipboard(aggregateData25, true)"
                                            title="Copy table to clipboard">
                                            📋
                                        </button>
                                        <strong>Total</strong>
                                    </td>
                                    <td></td>
                                    <td class="votes-cell"><strong>{{formatVotes(d3.sum(aggregateData25, (d: any) =>
                                        d.votes))}}</strong></td>
                                </tr>
                            </tfoot>
                        </table>

                    </div>
                </div>

                <!-- 2021 Aggregate -->
                <div class="chart-section">
                    <div class="chart-title">
                        <h3>2021 General (allocated to 2025 EDs)</h3>
                    </div>
                    <div ref="aggChart21"></div>
                    <div class="results-table-container">
                        <table class="results-table">
                            <thead>
                                <tr>
                                    <th>Candidate</th>
                                    <th>Filtered %</th>
                                    <th>Votes</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr v-for="(item, index) in aggregateData21" :key="index">
                                    <td>
                                        <div class="candidate-cell">
                                            <span class="color-indicator"
                                                :style="{ backgroundColor: item.color }"></span>
                                            <span>{{ item.name }}</span>
                                            <span v-if="index === 0 && aggregateData21.length > 1"
                                                class="margin-indicator">
                                                +{{ Math.round((aggregateData21[0]?.percentage || 0) -
                                                    (aggregateData21[1]?.percentage || 0)) }} points
                                            </span>
                                        </div>
                                    </td>
                                    <td class="percentage-cell">{{ formatPercent(item.percentage) }}%</td>
                                    <td class="percentage-cell votes-cell">{{ formatVotes(item.votes) }}</td>
                                </tr>
                            </tbody>
                            <tfoot>
                                <tr class="total-row">
                                    <td>
                                        <button class="copy-button" @click="copyTableToClipboard(aggregateData21, true)"
                                            title="Copy table to clipboard">
                                            📋
                                        </button>
                                        <strong>Total</strong>
                                    </td>
                                    <td></td>
                                    <td class="votes-cell"><strong>{{formatVotes(d3.sum(aggregateData21, (d: any) =>
                                        d.votes))}}</strong></td>
                                </tr>
                                <tr class="footernote">
                                    <td colspan="3">Totals may not add to 100% due to rounding.</td>
                                </tr>
                                <tr class="footernote">
                                    <td colspan="3">Results omit write-in votes.</td>
                                </tr>
                            </tfoot>
                        </table>

                    </div>
                </div>
            </template>
        </div>

        <!-- Scale Mode Toggle -->
        <div class="scale-toggle-container">
            <label class="scale-toggle-label">
                <input type="checkbox" v-model="scaleMode" true-value="votes" false-value="percentage"
                    class="scale-toggle-checkbox" />
                <span class="scale-toggle-text">
                    Scale bars by vote count
                </span>
            </label>
        </div>
    </div>
</template>

<style scoped>
.infobox-container {
    background-color: white;
    flex: 1;
    overflow-y: auto;
    display: flex;
    padding-right: 8px;
    flex-direction: column;
    font-family: Roboto, Helvetica, Arial, sans-serif;
}

.header {
    margin: 1rem 2px;
    padding: 0.2rem 0.5rem;
    outline: dashed rgb(0, 112, 240) 2px;
}

.header-content {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 12px;
}

.header h3 {
    margin: 0;
    font-size: 16px;
    font-weight: 600;
    padding-bottom: 5px;
}

.close-button {
    background: none;
    border: none;
    font-size: 28px;
    line-height: 1;
    cursor: pointer;
    padding: 0;
    color: #666;
    width: 28px;
    height: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 4px;
    transition: all 0.2s ease;
    flex-shrink: 0;
}

.close-button:hover {
    background-color: #f0f0f0;
    color: #333;
}

.close-button:active {
    background-color: #e0e0e0;
}

.filter-count {
    margin: 4px 0 0 0;
    font-size: 12px;
    color: #666;
}

.aggregate-section {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
}

.empty-state {
    padding: 2rem 1rem;
    text-align: center;
    color: #999;
    font-size: 14px;
    background-color: #f9f9f9;
    border-radius: 4px;
    border: 1px dashed #ddd;
}

.citywide-section {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
}

.ed-section {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
}

.section-header h2 {
    margin: 0;
    font-size: 15px;
    font-weight: 600;
    color: #333;
}

.section-subtitle-text-body-1 {
    margin: 4px 0 0 0;
    padding-top: 5px;
    color: #666;
    font-size: 1rem;
    font-weight: 600;
    line-height: 0.9;
    letter-spacing: 0.03125em;
}

.section-subtitle-text-body-2 {
    margin: 4px 0 0 0;
    color: #666;
    font-size: 0.875rem;
    font-weight: 400;
    line-height: 1;
    letter-spacing: 0.0178571429em;
}

.divider {
    height: 2px;
    background: linear-gradient(to right, #dee2e6, transparent);
    margin: 0.5rem 0;
}

.chart-section {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.chart-title {
    display: flex;
    align-items: center;
    gap: 10px;
}

.chart-title h3 {
    margin: 0;
    font-size: 14px;
    font-weight: 600;
}

.chart-title a, a:active {
  color: blue;
}

.chart-title a:visited {
  color: purple;
}

.total-votes {
    color: #666;
    font-weight: 500;
    font-family: monospace;
}

.chart-title p {
    margin: 0;
    display: flex;
    align-items: center;
    gap: 6px;
}

.winner-pill {
    display: inline-block;
    padding: 2px 8px;
    border-radius: 8px;
    font-size: 12px;
    font-weight: 600;
    color: white;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Results Table Styles */
.results-table-container {
    position: relative;
}

.copy-button {
    border: none;
    font-size: 0.8rem;
    cursor: pointer;
    border-radius: 8px;
    position: absolute;
    left: 0;
    font-size: 0.9rem;
    vertical-align: middle;
}


.copy-button:hover {
    background-color: #e8e8e8;
    border-color: #ccc;
}

.copy-button:active {
    background-color: #ddd;
}

.results-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 13px;
    background-color: white;
    border-left: none;
    border-right: none;
    border-top: 1px solid #ddd;
    border-bottom: 1px solid #ddd;
}

.results-table thead {
    background-color: #f5f5f5;
}

.results-table th {
    padding: 6px 8px;
    text-align: left;
    font-weight: 600;
    color: #333;
    border-bottom: 1px solid #ddd;
    font-size: 12px;
}

.results-table :not(:first-child) {
    text-align: right;
}


.results-table tbody tr {
    border-bottom: 1px solid #eee;
}

.results-table tbody tr:last-child {
    border-bottom: none;
}

.results-table tfoot {
    border-top: 2px solid #333;
}

.results-table tfoot .total-row td {
    font-family: monospace;
    background-color: #f5f5f5;
    position: relative;
}

.results-table tfoot .footernote {
    text-align: left;
    font-size: 11px;
}

.results-table tbody tr:hover {
    background-color: #fafafa;
}

.results-table td {
    padding: 2px 4px;
    font-family: monospace;
}

.results-table td.votes-cell {
    background-color: #f9f9f9;
}

.candidate-cell {
    display: flex;
    align-items: center;
    gap: 6px;
}

.color-indicator {
    width: 10px;
    height: 10px;
    border-radius: 2px;
    flex-shrink: 0;
}

.margin-indicator {
    font-weight: 700;
    font-size: 12px;
    color: #333;
    margin-left: 4px;
}

.percentage-cell {
    font-weight: 500;
    color: #333;
    text-align: right;
}

.citywide-text {
    color: #666;
}

:deep(.segment) {
    stroke: #333;
}

:deep(.segment:hover) {
    opacity: 0.8;
    cursor: pointer;
}

:deep(.vote-count) {
    fill: #666;
}

:deep(.bar-label) {
    fill: #ffffff;
}

/* Scale Toggle Styles */
.scale-toggle-container {
    position: sticky;
    bottom: 0;
    padding-top: 0.2rem;
    border-top: 1px solid #e0e0e0;
    background: linear-gradient(to top, white 80%, transparent);
    margin-top: auto;
}

.scale-toggle-label {
    display: flex;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    user-select: none;
    font-family: monospace;
}

.scale-toggle-text {
    font-size: 0.8rem;
    color: #555;
    font-weight: 500;
}

@media screen and (max-width: 600px) {
    .infobox-container {
        padding-right: 0.2rem;
        /* scroll bar */
    }

    .header h3 {
        font-size: 14px;
    }

    .chart-title {
        flex-direction: column;
        align-items: flex-start;
        gap: 4px;
    }

    .chart-title h3 {
        font-size: 13px;
    }

    .chart-title p {
        font-size: 12px;
    }

    .section-header h2 {
        font-size: 14px;
    }
}
</style>