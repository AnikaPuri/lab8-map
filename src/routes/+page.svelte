<script>
    import { onMount } from "svelte";

    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1IjoiYW5wdXJpIiwiYSI6ImNsdmlrZ2hraTAwdnQyanA5ano1aTRrdTcifQ.jnicnrkT06m-Bb2x1IwTEw";

    import * as d3 from "d3";

    // Variable to store the data fetched from the CSV
    let stations = [];

    let map;

    let trips = [];

    let departures;

    let arrivals;


    onMount(async () => {
        // Create the map
        map = new mapboxgl.Map({
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
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        // Add a layer to visualize the data (e.g., line layer for the route)
        map.addLayer({
            id: "boston_route_layer",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "#40b835",
                "line-width": 3,
                "line-opacity": 0.75
            }
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        //Add a layer to visualize the data (e.g., line layer for the route)
        map.addLayer({
            id: "cambridge_route_layer",
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "#40b835",
                "line-width": 3,
                "line-opacity": 0.75
            }
        });

        try {
            // Fetch the CSV data using d3.csv
            stations = await d3.csv(" https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
            console.log("Fetched station data:", stations);
        } catch (error) {
            console.error("Error fetching station data:", error);
        }


        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);

        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);


        stations = stations.map(station => {

            let id = station.Number;

            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.departures + station.arrivals;

            return station;
        });



        // Optional: Add navigation controls to the map (zoom buttons)
        //map.addControl(new mapboxgl.NavigationControl());
    });

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    let mapViewChanged = 0;

    $: map?.on("move", evt => mapViewChanged++);

    $: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range(timeFilter === -1 ? [0, 25] : [3, 50]);


    let timeFilter = -1;

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});

    function minutesSinceMidnight (date) {
	    return date.getHours() * 60 + date.getMinutes();    
    }
    
    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);

    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

    $: filteredStations = timeFilter === -1? stations : stations.map(station => {
        station = {...station};

        station.arrivals = filteredArrivals;
        station.departures = filteredDepartures;
        station.totalTraffic = station.arrivals + station.departures;

        return station

    });
    $: console.log(filteredStations)

</script>



<h1>Lab 8: Map</h1>

<header style="display: flex; gap: 1em; align-items: baseline;">
    <label style="margin-left: auto;">
        Filter by time:
        <input type="range" min="-1" max="1440" id="time-range" bind:value={timeFilter}/>
    </label>
    <time id="selected-time" style="display: block;">{timeFilterLabel}</time>
    <em id="any-time" style="display: block;"><i>(any time)</i></em>

</header>


<div id="map">
        <!-- SVG overlay to draw station circles -->
    <svg style="position: absolute; top: 0; left: 0; width: 100%; height: 400px;">
        {#key mapViewChanged}
            {#each filteredStations as station}
                <!-- Draw each station circle using the getCoords function -->
                <circle cx={ getCoords(station).cx }
                cy={ getCoords(station).cy }
                r={radiusScale(station.totalTraffic)} fill="steelblue" />
            {/each}
        {/key}

    </svg>
</div>


<style>

@import url("$lib/global.css");


#map {
	flex: 1;
    background-color: blue;
     height: 500px;
}

#map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
}

</style>
