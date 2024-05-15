<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import mapboxgl from 'mapbox-gl';
    import 'mapbox-gl/dist/mapbox-gl.css';

    mapboxgl.accessToken = "pk.eyJ1IjoiYzZsdW8iLCJhIjoiY2x3NGpsMWJ2MTUxNDJtcHZiaDlwenl0aCJ9.E1SLnII70oVnQcPYAizY_w";
    let map;
    let svg;
    let stopsData = { features: [] };
    let wheelchairFilter = false;
    let agencyFilter = "all";
    let pointCount = 0;
    let tooltip;
    let stops;

    onMount(async () => {
        map = new mapboxgl.Map({
            container: "map",
            style: 'mapbox://styles/mapbox/light-v11',
            center: [-117.1611, 32.7157],
            zoom: 12,
            minZoom: 10,
            maxZoom: 17,
        });

        map.on('load', async () => {
            svg = d3.select(map.getCanvasContainer()).append("svg")
                .attr("style", "position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: auto;");

            tooltip = d3.select("body").append("div")
                .attr("class", "tooltip")
                .style("position", "absolute")
                .style("visibility", "hidden")
                .style("background", "white")
                .style("border", "1px solid #ccc")
                .style("padding", "10px")
                .style("border-radius", "5px")
                .style("box-shadow", "0 0 10px rgba(0, 0, 0, 0.2)");

            const response = await fetch('transit_stops_datasd.geojson');
            stopsData = await response.json();
            drawStops();
            
            map.on("move", drawStops); 
            map.on("zoom", drawStops); 
        });

    });

    function drawStops() {
        if (!svg || !stopsData.features.length) return;

        let filteredData = stopsData.features;

        if (wheelchairFilter) {
            filteredData = filteredData.filter(d => d.properties.wheelchair === 1);
        }

        if (agencyFilter !== "all") {
            filteredDçata = filteredData.filter(d => d.properties.stop_agncy === agencyFilter);
        }

        pointCount = filteredData.length;

        const stops = svg.selectAll("circle")
            .data(filteredData, d => d.properties.stop_id)
            .join(
                enter => enter.append("circle")
                                .attr("r", map.getZoom() / 2.4)
                                .attr("fill", d => d.properties.wheelchair === 1 ? "#FFEB3B" : "#BA68C8") // Dark Blue for wheelchair accessible, Light Yellow otherwise
                                .attr("stroke", d => d.properties.stop_agncy === "MTS" ? "#B71C1C" : "#43A047")
                                .attr("stroke-width", 1.5)
                                .style("fill-opacity", 0.8)
                                .on("mouseover", (event, d) => {
                                    tooltip.style("visibility", "visible")
                                        .html(`<strong>Stop Name:</strong> ${d.properties.stop_name}<br><strong>Wheelchair Accessible:</strong> ${d.properties.wheelchair ? "Yes" : "No"}<br><strong>Agency:</strong> ${d.properties.stop_agncy}`);
                                })
                                .on("mousemove", (event) => {
                                    tooltip.style("top", (event.pageY - 10) + "px").style("left", (event.pageX + 10) + "px");
                                })
                                .on("mouseout", () => {
                                    tooltip.style("visibility", "hidden");
                                }),
                update => update
                    .attr("fill", d => d.properties.wheelchair === 1 ? "#FFEB3B" : "#BA68C8"),
                exit => exit.remove()
            )
            .attr("cx", d => project(d).x)
            .attr("cy", d => project(d).y);

        // console.log('Stops drawn:', filteredData.length);
    }

    function project(d) {
        const point = map.project(new mapboxgl.LngLat(+d.geometry.coordinates[0], +d.geometry.coordinates[1]));
        return { x: point.x, y: point.y };
    }

    $: {
        if (stopsData.features) {
            drawStops();
        }
    }
</script>

<main>
    <div class="overlay">
        <label>
            <span style="font-size: 14px">
                Wheelchair Accessible Only
            </span>
            <input type="checkbox" bind:checked={wheelchairFilter} style="display: inline-block;">
        </label>

        <div>
            <label>
                Agency:
            </label>
            {#if stopsData.features.length > 0}
                <select bind:value={agencyFilter}>
                    <option value="all">All</option>
                    {#each Array.from(new Set(stopsData.features.map(d => d.properties.stop_agncy))) as agency}
                        <option value={agency}>{agency}</option>
                    {/each}
                </select>
            {/if}
        </div>
        <div>
            Stations with feature support: {pointCount}
        </div>
        
        <div>
            <label>
                <hr>
                Legend
            </label>
        </div>

        <div>
            <svg width="200" height="110">
                <circle cx="10" cy="20" r="5" fill="#FFEB3B" />
                <text x="40" y="25" font-family="Arial" font-size="16" fill="black">
                    Wheelchair
                </text>
                <circle cx="10" cy="45" r="5" fill="#BA68C8" />
                <text x="40" y="50" font-family="Arial" font-size="16" fill="black">
                    Non-Wheelchair
                </text>
                <circle cx="10" cy="70" r="5" stroke="#B71C1C" stroke-width="1.5" fill="#FFFFFF"/>
                <text x="40" y="75" font-family="Arial" font-size="16" fill="black">
                    MTS
                </text>
                <circle cx="10" cy="95" r="5" stroke="#43A047" stroke-width="1.5" fill="#FFFFFF"/>
                <text x="40" y="100" font-family="Arial" font-size="16" fill="black">
                    NCTD
                </text>
            </svg>
        </div>
    </div>
</main>

<style>
    .tooltip {
        position: absolute;
        visibility: hidden;
        background: white;
        border: 1px solid #ccc;
        padding: 10px;
        border-radius: 5px;
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
    }
</style>
