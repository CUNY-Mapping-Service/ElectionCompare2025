# ElectionCompare

## Quick Start

```
npm install
npm run dev
```

Pushing to main will run a github action to build and serve the new files!

## Updating settings and data

To update the data replace/overwrite [./src/stores/results.geojson](./src/stores/results.geojson)

In [./src/App.vue](./src/App.vue) you can update both COLOR_SCALE and SETTINGS to the new data schema in your geojson.
