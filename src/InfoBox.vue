<script setup lang="ts">
import { computed, onMounted, ref, useTemplateRef, watch } from 'vue';
import * as d3 from 'd3';

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
}

const props = defineProps<Props>();
const emit = defineEmits<{
    close: []
}>();

const edLabel = computed(() => {
    const id = props.hoveredData?.properties[props.idKey]
    if (id) {
        const ad = String(id).slice(0, 2)
        const ed = String(id).slice(2)
        return `ED ${ed} in AD ${ad}`
    }
    return ''
})

const isVisible = computed(() => props.hoveredData !== null || hasFilteredData.value);
const hasFilteredData = computed(() => props.filteredFeatures && props.filteredFeatures.length > 0);

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

// Aggregate data computation for filtered districts
const aggregateData25 = computed(() => {
    if (!hasFilteredData.value) return [];

    const candidates = props.COLOR_SCALE.candidates;
    const totals: { [key: string]: { votes: number, name: string, color: string } } = {};

    candidates.forEach((c: any) => {
        totals[c.id] = { votes: 0, name: c.label, color: c.colors[4] };
    });

    props.filteredFeatures!.forEach(feature => {
        candidates.forEach((c: any) => {
            if (totals[c.id]) {
                // @ts-ignore
                totals[c.id].votes += feature.properties[c.id] || 0;
            }
        });
    });

    const totalVotes = Object.values(totals).reduce((sum, c) => sum + c.votes, 0);

    return Object.values(totals)
        .map(c => ({
            name: c.name,
            votes: c.votes,
            percentage: totalVotes > 0 ? (c.votes / totalVotes) * 100 : 0,
            color: c.color
        }))
        .filter(d => d.votes > 0);
});

const aggregateData21 = computed(() => {
    if (!hasFilteredData.value) return [];

    const totals = {
        ea: { votes: 0, name: 'Eric Adams', color: props.COLOR_SCALE.candidates[3].colors[4] },
        cs: { votes: 0, name: 'Curtis Sliwa', color: props.COLOR_SCALE.candidates[2].colors[4] },
        other: { votes: 0, name: 'Other', color: props.COLOR_SCALE.other.colors[4] }
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
        .filter(d => d.votes > 0);
});

// Prepare data for 2025 election - using percentage data
const data25 = computed(() => {
    if (!props.hoveredData) return [];

    const candidates = props.COLOR_SCALE.candidates;

    return candidates
        .map((c: any) => ({
            name: c.label,
            percentage: props.hoveredData!.properties[`${c.id}_pct`] || 0,
            votes: props.hoveredData!.properties[c.id] || 0,
            color: c.colors[4],
        }))
        .filter((d: any) => d.votes > 0);
});

// Prepare data for 2021 election - using percentage data
const data21 = computed(() => {
    if (!props.hoveredData) return [];

    return [
        {
            name: 'Eric Adams',
            percentage: props.hoveredData.properties.gen21ea_pct || 0,
            votes: props.hoveredData.properties.gen21ea || 0,
            color: props.COLOR_SCALE.candidates[3].colors[4],
        },
        {
            name: 'Curtis Sliwa',
            percentage: props.hoveredData.properties.gen21cs_pct || 0,
            votes: props.hoveredData.properties.gen21cs || 0,
            color: props.COLOR_SCALE.candidates[2].colors[4],
        },
        {
            name: 'Other',
            percentage: props.hoveredData.properties.gen21othr_pct || 0,
            votes: props.hoveredData.properties.gen21othr || 0,
            color: props.COLOR_SCALE.other.colors[4],
        }
    ]
        .filter((d: any) => d.votes > 0);
});

// Get winner labels
const winner25 = computed(() => props.hoveredData?.properties.win25 || '');
const winner21 = computed(() => props.hoveredData?.properties.win21 || '');

// Get aggregate winners
const aggregateWinner25 = computed(() => {
    if (aggregateData25.value.length === 0) return '';
    return aggregateData25.value.reduce((a, b) => a.votes > b.votes ? a : b).name;
});

const aggregateWinner21 = computed(() => {
    if (aggregateData21.value.length === 0) return '';
    return aggregateData21.value.reduce((a, b) => a.votes > b.votes ? a : b).name;
});

// Find winner color
const getWinnerColor = (winner: string, year: string) => {
    if (year === '2025') {
        const candidate = props.COLOR_SCALE.candidates.find((c: any) => c.label === winner);
        return candidate?.colors[4] || '#999';
    } else {
        if (winner === 'Eric Adams') return props.COLOR_SCALE.candidates[3].colors[4];
        if (winner === 'Curtis Sliwa') return props.COLOR_SCALE.candidates[2].colors[4];
        return '#999';
    }
};

// Draw stacked bar chart with responsive width
const drawStackedChart = (ref: any, data: any[], year: string, winner: string) => {
    if (!ref || data.length === 0) return;

    d3.select(ref).selectAll("*").remove();

    const margin = { top: 0, right: 20, bottom: 20, left: 20 };
    const width = containerWidth.value - margin.left - margin.right;
    const height = 40;

    const svg = d3.select(ref)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

    // Calculate cumulative positions using percentages
    const total = d3.sum(data, (d: any) => d.percentage);
    let cumulativePosition = 0;
    const segments = data.map((d: any) => {
        const segWidth = (d.percentage / total) * 100;
        const start = cumulativePosition;
        cumulativePosition += segWidth;
        return { ...d, start, width: segWidth };
    });

    // X scale (0-100 for percentages)
    const x = d3.scaleLinear()
        .domain([0, 100])
        .range([0, width]);

    // Draw stacked bar
    svg.selectAll('.segment')
        .data(segments)
        .enter()
        .append('rect')
        .attr('class', 'segment')
        .attr('x', (d: any) => x(d.start))
        .attr('y', 0)
        .attr('width', (d: any) => x(d.width))
        .attr('height', 40)
        .attr('fill', (d: any) => d3.rgb(d.color).brighter(0.4).toString())
        .attr('stroke', '#333')
        .attr('stroke-width', 1);

    // Labels on bar
    svg.selectAll('.bar-label')
        .data(segments.filter(d => d.percentage >= 5))
        .enter()
        .append('text')
        .attr('class', 'bar-label')
        .attr('x', (d: any) => x(d.start) + x(d.width) / 2)
        .attr('y', 20)
        .attr('text-anchor', 'middle')
        .attr('dy', '0.35em')
        .attr('font-size', '11px')
        .attr('font-weight', 'bold')
        .attr('fill', '#000')
        .text((d: any) => `${d.percentage.toFixed(1)}%`);

    // Legend below bar - showing both percentage and vote count
    const legendY = 60;
    const legendData = data;
    svg.selectAll('.legend-item')
        .data(legendData)
        .enter()
        .append('g')
        .attr('class', 'legend-item')
        .attr('transform', (d: any, i: number) => `translate(${(width / legendData.length) * i}, ${legendY})`)
        .each(function (d: any) {
            d3.select(this)
                .append('text')
                .attr('font-size', '11px')
                .attr('font-weight', '500')
                .text(`${d?.name?.split(' ')[1] ?? 'Other'} (${Math.round(d.votes).toLocaleString()})`);
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

    // ED charts
    if (chart25Ref.value && data25.value.length > 0) {
        drawStackedChart(chart25Ref.value, data25.value, '2025', winner25.value);
    }
    if (chart21Ref.value && data21.value.length > 0) {
        drawStackedChart(chart21Ref.value, data21.value, '2021', winner21.value);
    }

    // Aggregate charts
    if (hasFilteredData.value) {
        if (aggChart25Ref.value && aggregateData25.value.length > 0) {
            drawStackedChart(aggChart25Ref.value, aggregateData25.value, '2025', aggregateWinner25.value);
        }
        if (aggChart21Ref.value && aggregateData21.value.length > 0) {
            drawStackedChart(aggChart21Ref.value, aggregateData21.value, '2021', aggregateWinner21.value);
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

watch([data25, data21, aggregateData25, aggregateData21, containerWidth, isVisible], () => {
    if (!isVisible.value) return;

    // Wait a tick 
    requestAnimationFrame(() => {
        redrawCharts();
    });
}, { immediate: true });
</script>

<template>
    <div ref="container" class="infobox-container" v-if="isVisible">
        <!-- Aggregate Charts - show when filters are active -->
        <div v-if="hasFilteredData" class="aggregate-section">
            <div class="section-header">
                <p class="section-subtitle-text-body-1">Combined results from {{ filteredFeatures?.length.toLocaleString() }} filtered election districts:</p>
                <p class="section-subtitle-text-body-2">{{ filterLabelsText }}</p>
            </div>

            <!-- 2025 Aggregate -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2025 General</h3>
                    <p>
                        Winner: <span class="winner-pill"
                            :style="{ backgroundColor: getWinnerColor(aggregateWinner25, '2025') }">
                            {{ aggregateWinner25 }}
                        </span>
                    </p>
                </div>
                <div ref="aggChart25"></div>
            </div>

            <!-- 2021 Aggregate -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2021 General</h3>
                    <p>
                        Winner: <span class="winner-pill"
                            :style="{ backgroundColor: getWinnerColor(aggregateWinner21, '2021') }">
                            {{ aggregateWinner21 }}
                        </span>
                    </p>
                </div>
                <div ref="aggChart21"></div>
            </div>
        </div>
        <div class="header" v-if="hoveredData">
            <div class="header-content">
                <div>
                    <h3>You have selected: {{ hasFilteredData && !hoveredData ? 'Filtered Districts' : (hasFilteredData
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
                        <h3>2025 General</h3>
                        <p>
                            ED winner: <span class="winner-pill"
                                :style="{ backgroundColor: getWinnerColor(winner25, '2025') }">
                                {{ winner25 }}
                            </span>
                        </p>
                    </div>
                    <div ref="chart25"></div>
                </div>

                <!-- 2021 Election Chart -->
                <div class="chart-section">
                    <div class="chart-title">
                        <h3>2021 General</h3>
                        <p>
                            ED winner: <span class="winner-pill"
                                :style="{ backgroundColor: getWinnerColor(winner21, '2021') }">
                                {{ winner21 }}
                            </span>
                        </p>
                    </div>
                    <div ref="chart21"></div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.infobox-container {
    background-color: white;
    flex: 1;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    font-family: Roboto, Helvetica, Arial, sans-serif;
}

.header {
    margin: 0.2rem 0.1rem;
    padding: 1rem;
    outline: dashed rgb(0, 112, 240) 0.5px;
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
/*     font-size: 12px;
    font-style: italic; */
    color: #666;
font-size: 1rem;
font-weight: 600;
line-height: 1.5;
letter-spacing: 0.03125em;
}

.section-subtitle-text-body-2 {
    margin: 4px 0 0 0;
/*     font-size: 12px;
    font-style: italic; */
    color: #666;
font-size: 0.875rem;
font-weight: 400;
line-height: 1.425;
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

:deep(.segment) {
    stroke: #333;
}

:deep(.segment:hover) {
    opacity: 0.8;
    cursor: pointer;
}

:deep(.bar-label) {
    fill: #000;
    font-weight: bold;
}

:deep(.legend-item) {
    font-size: 11px;
}

@media screen and (max-width: 600px) {
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