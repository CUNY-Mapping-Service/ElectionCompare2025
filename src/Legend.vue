<script setup lang="ts">

const { COLOR_SCALE } = defineProps(['COLOR_SCALE'])
</script>

<template>
    <div class="legend">
        <div class="breakpoint" v-for="breakpoint in COLOR_SCALE" :key="breakpoint.label">
            <span class="swatch" :style="`background-color: ${breakpoint.color}`"></span>
            <p>{{ breakpoint.label }}</p>
        </div>
        <p class="subtitle">Blank areas = no votes and/or no population</p>
    </div>
</template>

<style scoped>
.legend{
    display: flex;
    flex-direction: column;
}
.breakpoint{
    display: flex;
    flex-direction: row;
    text-wrap: wrap;
    margin-top: 0.4rem;
    gap: 0.2rem;
}

.breakpoint p{
    margin-top: 0;
    line-height: 1.1;
    font-size: 0.9rem;
}
.swatch{
    display: block;
    width: 1rem;
    height: 1rem;
}
.subtitle{
    font-size: 0.8rem;
    letter-spacing: -0.5px;
}
</style>