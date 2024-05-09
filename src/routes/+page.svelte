<script>
    import { onMount } from "svelte";

    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1IjoiYW5wdXJpIiwiYSI6ImNsdmlrZ2hraTAwdnQyanA5ano1aTRrdTcifQ.jnicnrkT06m-Bb2x1IwTEw";

    onMount(async () => {
        // Create the map
        let map = new mapboxgl.Map({
            container: "map", // The ID of the HTML element that contains the map
            style: "mapbox://styles/mapbox/streets-v12", // Mapbox style URL
            center: [-71.0942, 42.3601], // Longitude and latitude (MIT HQ coordinates)
            zoom: 12 // Initial zoom level
        });

        // Wait for the map to load before adding the source
        await new Promise((resolve) => map.on("load", resolve));

        // Add the GeoJSON data source
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network.geojson"
        });

        // Add a layer to visualize the data (e.g., line layer for the route)
        map.addLayer({
            id: "boston_route_layer",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "#888",
                "line-width": 3
            }
        });

        // Optional: Add navigation controls to the map (zoom buttons)
        //map.addControl(new mapboxgl.NavigationControl());
    });

</script>



<h1>Lab 8: Map</h1>
<p>Paragraph Description of Lab 8</p>

<div id="map" />


<style>

@import url("$lib/global.css");


#map {
	flex: 1;
    background-color: blue;
    height: 500px;
}

</style>
