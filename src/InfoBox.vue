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
}

const props = defineProps<Props>();

const edLabel = computed(() => {
    const id = props.hoveredData?.properties[props.idKey]
    if (id) {
        const ad = String(id).slice(0, 2)
        const ed = String(id).slice(2)
        return `ED ${ed} in AD ${ad}`
    }
    return ''
})

const isVisible = computed(() => props.hoveredData !== null);

// Chart refs
const chart25Ref = useTemplateRef('chart25');
const chartPrimaryRef = useTemplateRef('chartPrimary');
const chart21Ref = useTemplateRef('chart21');
const containerRef = useTemplateRef('container');

// Track container width for responsive sizing
const containerWidth = ref(320);

// Prepare data for 2025 election
const data25 = computed(() => {
    if (!props.hoveredData) return [];
    
    const candidates = props.COLOR_SCALE.candidates;
    const total = candidates.reduce((sum: number, c: any) => {
        return sum + (props.hoveredData!.properties[`${c.id.toLowerCase()}25`] || 0);
    }, 0);

    return candidates
        .map((c: any) => ({
            name: c.label,
            votes: props.hoveredData!.properties[`${c.id.toLowerCase()}25`] || 0,
            color: c.colors[4],
        }))
        .filter((d: any) => d.votes > 0)
        .map((d: any) => ({
            ...d,
            percentage: total > 0 ? (d.votes / total) * 100 : 0
        }));
});

// Prepare data for 2021 election
const data21 = computed(() => {
    if (!props.hoveredData) return [];
    
    const total = (props.hoveredData.properties.adams21 || 0) + 
                  (props.hoveredData.properties.sliwa21 || 0) + 
                  (props.hoveredData.properties.other21 || 0);

    return [
        {
            name: 'Adams',
            votes: props.hoveredData.properties.adams21 || 0,
            color: props.COLOR_SCALE.candidates[3].colors[4],
        },
        {
            name: 'Sliwa',
            votes: props.hoveredData.properties.sliwa21 || 0,
            color: props.COLOR_SCALE.candidates[2].colors[4],
        },
        {
            name: 'Other',
            votes: props.hoveredData.properties.other21 || 0,
            color: props.COLOR_SCALE.other.colors[4],
        }
    ]
    .filter((d: any) => d.votes > 0)
    .map((d: any) => ({
        ...d,
        percentage: total > 0 ? (d.votes / total) * 100 : 0
    }))
});

// Get winner labels
const winner25 = computed(() => props.hoveredData?.properties.winner25 || '');
const winner21 = computed(() => props.hoveredData?.properties.winner21 || '');

// Find winner color
const getWinnerColor = (winner: string, year: string) => {
    if (year === '2025') {
        const candidate = props.COLOR_SCALE.candidates.find((c: any) => c.label === winner);
        return candidate?.colors[4] || '#999';
    } else {
        if (winner === 'Adams') return props.COLOR_SCALE.candidates[3].colors[4];
        if (winner === 'Sliwa') return props.COLOR_SCALE.candidates[2].colors[4];
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

    // Calculate cumulative positions
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
        .data(segments.filter(d => d.name !== 'Other' || d.percentage >= 10))
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

    // Legend below bar
    const legendY = 60;
    const legendData = data;
    svg.selectAll('.legend-item')
        .data(legendData)
        .enter()
        .append('g')
        .attr('class', 'legend-item')
        .attr('transform', (d: any, i: number) => `translate(${(width / legendData.length) * i}, ${legendY})`)
        .each(function(d: any) {
            d3.select(this)
                .append('text')
                .attr('font-size', '11px')
                .attr('font-weight', '500')
                .text(`${d.name}: ${d.votes.toLocaleString()}`);
        });
};

// Update container width on mount and window resize
const updateContainerWidth = () => {
    if (containerRef.value) {
        containerWidth.value = (containerRef.value as HTMLElement).offsetWidth - 30; // 30 for padding
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

onMounted(() => {
    updateContainerWidth();
    
    // Debounced resize handler
    const debouncedResize = debounce(updateContainerWidth, 100);
    window.addEventListener('resize', debouncedResize);

    watch([data25, data21, containerWidth], () => {
        if (chart25Ref.value && data25.value.length > 0) {
            drawStackedChart(chart25Ref.value, data25.value, '2025', winner25.value);
        }
        if (chart21Ref.value && data21.value.length > 0) {
            drawStackedChart(chart21Ref.value, data21.value, '2021', winner21.value);
        }
    }, { immediate: true });

    return () => {
        window.removeEventListener('resize', debouncedResize);
    };
});
</script>

<template>
    <div ref="container" class="infobox-container" v-if="isVisible">
        <div class="header">
            <h3>{{ edLabel }}</h3>
        </div>

        <div class="charts-wrapper">
            <!-- 2025 Election Chart -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2025 Election</h3>
                   <p>
                     Winner: <div class="winner-pill" :style="{ backgroundColor: getWinnerColor(winner25, '2025') }">
                        {{ winner25 }}
                    </div>
                   </p>
                </div>
                <div ref="chart25"></div>
            </div>

            <!-- 2025 Primary Placeholder -->
            <div class="chart-section">
            </div>

            <!-- 2021 Election Chart -->
            <div class="chart-section">
                <div class="chart-title">
                    <h3>2021 Election</h3>
                    <p>
                       Winner:  <div class="winner-pill" :style="{ backgroundColor: getWinnerColor(winner21, '2021') }">
                        {{ winner21 }}
                    </div>
                    </p>
                </div>
                <div ref="chart21"></div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.infobox-container {
    position: absolute;
    right: 5px;
    bottom: 5px;
    z-index: 99;
    background-color: white;
    box-shadow: 2px 2px 5px rgba(94, 94, 94, 0.76);
    padding: 15px;
    border-radius: 4px;
    min-width: 200px;
    max-width: 400px;
}

.header {
    margin-bottom: 15px;
    border-bottom: 2px solid #dee2e6;
    padding-bottom: 10px;
}

.header h3 {
    margin: 0;
    font-size: 16px;
    font-weight: 600;
}

.charts-wrapper {
    display: flex;
    flex-direction: column;
    gap: 1rem
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
}

.winner-pill {
    display: inline-block;
    padding: 0px 8px;
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
    .infobox-container {
        max-width: 90vw;
        left: 5px;
        right: 5px;
    }
}
</style>