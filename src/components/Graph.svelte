<script>
        import * as d3 from 'd3';
        import { onMount } from 'svelte';
        import { writable } from 'svelte/store';
	import * as topojson from 'topojson-client';
	import { geoPath, geoAlbersUsa } from 'd3-geo';
	import { draw } from 'svelte/transition';
        import { geoOrthographic, geoGraticule10} from "d3-geo";
	import { json } from "d3-fetch";

        let svg;
        let gx;
        let gy;
        // grabbing the data
        export let volcanos;
        export let US_volcanos;

        const projection = geoAlbersUsa().scale(1300).translate([487.5, 305])
        const path = geoPath().projection(null);
        const yearCategories = ['pre-1800s', '1800s', '1900s', '2000s'];

        let states = [];
        let counties = [];
        let mesh;

        const currentIndex = writable('');
        let selected;
	var dots = 'temp';
	let eruptionIndex = 0;
        var nextEruptionMode = 1;
        let select = 1;
	let mapped = select;
        
        var scatterX, scatterY;
        let svgWidth, svgHeight;
        var margin = { top: 20, right: 50, bottom: 50, left: 20 };
        
        let selectedYear;
        let prevSelectedYear = null;
        
        let selectedLocation = null;
        let prevSelectedLocation = null;
        
        let filterState = {
                year: null, // Initialize with no year filter
                location: null // Initialize with no location filter
         };
        let filteredVolcanos = US_volcanos;
  	let slicing = filteredVolcanos;

        function filterByYear(year) {
                if (year === '1800s') {
                        filterState.year = 1800;
                } else if (year === '1900s') {
                        filterState.year = 1900;
                } else if (year === '2000s') {
                        filterState.year = 2000;
                } else if (year === 'pre-1800s') {
                        filterState.year = 'pre-1800s'; 
                } else {
                        filterState.year = null; // Reset filter in other cases
                }
                updateFilteredData(); 
        }

        function filterByYearAndUpdateClass(category, button) {
                // console.log(select, eruptionIndex);
                if (nextEruptionMode === 1){
                        select = 1;
                        eruptionIndex = 0; 
                } else {
                        select = 0;
                        eruptionIndex = 1;
                }

		// If same button pressed twice, deactivate it and change category to null
                if (selectedYear === category) {
                        selectedYear = null;
                        category = null;
                } else {
                // If not, change the selected year to the pressed button
                        selectedYear = category;
                }

                // Deactivates the previously pressed button
                if (prevSelectedYear && prevSelectedYear !== button) {
                        prevSelectedYear.setAttribute('class', 'btn btn-outline-primary');
                }
                
                // Set the prevSelectedYear to the current button
                if (button) {
                        button.setAttribute('class', 'btn btn-primary');
                        prevSelectedYear = button;
                };

                filterByYear(category);
        }

        function filterByLocationAndUpdateClass(location, button) {
                // If same button pressed twice, deactivate it
		select = 1;
    		eruptionIndex = 0; 
                if (selectedLocation === location) {
                        selectedLocation = null;
                        location = null;
                } else {
                // Else change selected location to the button's
                        selectedLocation = location;
                }

                 // Deactivates the previously pressed button
                 if (prevSelectedLocation && prevSelectedLocation !== button) {
                        prevSelectedLocation.setAttribute('class', 'btn btn-outline-primary');
                }

                if (button) {
                        button.setAttribute('class', 'btn btn-primary');
                        prevSelectedLocation = button;
                }

                filterState.location = location;
                updateFilteredData();
        }

        function updateFilteredData() {
                console.log("nextEruptionMode ", nextEruptionMode);
                filteredVolcanos = US_volcanos.filter(d => {
                        const yearMatches = filterState.year === 'pre-1800s' ? d.year < 1800 : filterState.year !== null ? d.year >= filterState.year && d.year <= filterState.year + 99 : true; 
                        const locationMatches = filterState.location ? d.location === filterState.location : true;
                        // console.log("Filtering by year:", filterState.year);
                        // console.log("Filtering by location:", filterState.location);
                        // console.log("Number of matches:", filteredVolcanos.length);
			slicing = filteredVolcanos;
                        return yearMatches && locationMatches;
                });
                currentIndex.set(findCurrentIndex());
                if (dots != 'temp') {
                        dots.remove();
                        populateScatter(svg);
                }
                if (nextEruptionMode == 0) {
                        eruptionIndex = 1;
                        nextEruptionMode = 0;
                        select = 0;
                        hideTooltip();
                }
                slicing = filteredVolcanos;
        }


        onMount(async () => {
		const us = await fetch('https://cdn.jsdelivr.net/npm/us-atlas@3/counties-albers-10m.json').then(d => d.json())
                while (US_volcanos.length === 0) {
                        // Pause for 100 ms
                        await new Promise(resolve => setTimeout(resolve, 100));
                }
                updateFilteredData();

		states = topojson.feature(us, us.objects.states).features;
		// console.log({ features })
		
		counties = topojson.feature(us, us.objects.counties).features;

		mesh = topojson.mesh(us, us.objects.states, (a, b) => a !== b);

                let min_year = filteredVolcanos.reduce( (min, obj) => (obj.year < min ? obj.year : min), filteredVolcanos[0].year);
                let max_year = filteredVolcanos.reduce( (max, obj) => (obj.year > max ? obj.year : max), filteredVolcanos[0].year);

                
                // Function to update SVG when user resizes window
                function updateSize() {

                        graphTable = d3.select('th').node().getBoundingClientRect();

                        svgWidth = parseFloat(graphTable.width)/3;
                        svgHeight = parseFloat(graphTable.height);


                        svg.attr("width", svgWidth + margin.left + margin.right)
                           .attr("height", svgWidth + margin.left + margin.right)

                        var fontSize = Math.max(12, (svgWidth - margin.left - margin.right) / 110);

                        // Update the x-axis scale
                        scatterX.range([0, svgWidth]);
                        scatterY.range([0, svgWidth * yScaleFactor]);

                        svg.select('.x-axis').call(d3.axisBottom(scatterX))
                           .selectAll("text").style('font-size', fontSize + "px");
                        svg.select('.y-axis').call(d3.axisLeft(scatterY))
                           .selectAll("text").style('font-size', fontSize + "px");

                        dots.attr('transform', 'translate(' + (margin.left*3) + ',' + ( svgWidth/4 + margin.top ) + ')');

                        // console.log("Update size", svgWidth, svgHeight);

                }

                // console.log("eruptionIndexing");
                
                let graphTable = d3.select('th').node().getBoundingClientRect();

                svgWidth = parseFloat(graphTable.width)/3;
                svgHeight = parseFloat(graphTable.height);

        


                let initialFontSize = Math.max(12, (svgWidth - margin.left - margin.right) / 110);
                // console.log(svgWidth, svgHeight, "Dimensions");

                svg = d3.select("#scatter")
                        .append("svg")
                        .attr("width", svgWidth + margin.left*3 + margin.right)
                        .attr("height", svgHeight)
                        .append("g")

                        
                // x-axis
                scatterX = d3.scaleLinear().domain([min_year, max_year]).range([0, svgWidth]);
                var x_axis = svg.append("g")
                        .attr("class", "x-axis")
                        .style('font-size', initialFontSize+'px')
                        .attr("transform",
                                "translate(" + margin.left*3 + "," + (svgWidth + margin.top) + ")")
                        .call(d3.axisBottom(scatterX));
                

                        const binWidth = 100;
                const numBins = Math.ceil((max_year - min_year + 1) / binWidth);
                const xDomain = d3.range(min_year, max_year + binWidth, binWidth);

                // // x-axis
                // scatterX = d3.scaleBand()
                //         .domain(xDomain)
                //         .range([0, svgWidth]);

                // const tickValues = [-2000, -1500, -1000, -500, 0, 500, 1000, 1500, 2000];
                // // Append the x-axis to the SVG
                // var x_axis = svg.append("g")
                // .attr("class", "x-axis")
                // .style('font-size', initialFontSize + 'px')
                // .attr("transform", "translate(" + margin.left*3 + "," + (svgWidth + margin.top*2) + ")")
                // .call(d3.axisBottom(scatterX)
                //         .tickValues(scatterX.domain().filter((d, i) => tickValues.includes(d)))
                // );


                let yScaleFactor = 3/4;

                // y-axis without tick marks
                scatterY = d3.scaleLinear().domain([8, 0]).range([0, svgWidth * yScaleFactor]);
        
                // tick marks
                var y_axis = svg.append("g")
                        .attr("class", "y-axis-marks")
                        .call(d3.axisLeft(scatterY))
                        .attr('text-anchor', 'end')
                        .style('font-size', initialFontSize+'px')
                        .attr('transform',
                                'translate(' + (margin.left*2.5) + ',' + (margin.top + svgWidth/4 ) + ')')
                // yMarks.select('.domain').style('stroke', 'none');

                // x-axis label
                svg.append('text')
                        .attr('class', 'x-label')
                        .text("Year")
                        .style('font-size', initialFontSize+2 + 'px')
                        .attr("transform",
                                "translate(" + (margin.left*3 + svgWidth/2 - (initialFontSize+7) ) + "," + (margin.top + svgWidth + margin.bottom) + ")");

                
                // y-axis label
                svg.append('text')
                        .attr('class', 'y-label')
                        .attr('text-anchor', 'middle')
                        .attr('text-align', 'center')
                        .attr('transform', 'translate(' + (margin.left) + ',' + (margin.top + svgWidth * (1-yScaleFactor) + (svgWidth*yScaleFactor/2)) + ') rotate(-90)')
                        .text("Explosivity")
                        .style('font-size', initialFontSize + 2 + 'px');

                // function kernelDensityEstimator(kernel, X) {
                //         return function(V) {
                //                 return X.map(function(x) {
                //                         return [x, d3.mean(V, function(v) { return kernel(x - v); })];
                //                 });
                //         };
                // }
                // function kernelEpanechnikov(k) {
                //         return function(v) {
                //                 return Math.abs(v /= k) <= 1 ? 0.75 * (1 - v * v) / k : 0;
                //         };
                // }

                // // Compute kernel density estimation for each column:
                // var kde = kernelDensityEstimator(kernelEpanechnikov(7), scatterX.ticks(20)) // increase this 40 for more accurate density.
                // var allDensity = []
                // var categories = [0, 1, 2, 3, 4, 5, 6, 7, 8];
                // var n = categories.length;
                // let i;
                // for (let i = 0; i < n; i++) {
                //         let category = categories[i];
                //         let values = filteredVolcanos.filter(d => d.Volcano_explosive_index == category).map(d => d.year);
                //         // console.log("Values for category " + category + ":", values); // Log values for debugging
                //         // console.log("Category", category, values);
                //         let density = kde(values);
                //         // console.log("Density for category " + category + ":", density); // Log density for debugging
                //         allDensity.push({ key: category, density: density });
                // }
                // console.log(allDensity);

                // // Define x-scale (assuming `scatterX` is already defined)
                // const xScale = scatterX;

                // // Define y-scale for the density curves
                // const yScale = scatterY;

                // // Add areas
                // svg.selectAll("areas")
                //         .data(allDensity)
                //         .enter()
                //         .append("path")
                //         .attr("transform", function(d) {
                //                 return "translate(" + (margin.left*3) + "," + ( -scatterY(d.key) + margin.top  + svgWidth/4 ) + ")"; 
                //         }) // Adjust this based on your y-axis scale
                //         .attr("opacity", 0.7)
                //         .attr("fill", "green")
                //         .attr("stroke", "#000")
                //         .attr("stroke-width", 0.1)
                //         .attr("d", function(d) { return d3.line()
                //                 .curve(d3.curveBasis)
                //                         .x(function(d) { 
                //                                 // console.log(d);
                //                                 return scatterX(d[0]);
                //                          }) // Adjust this based on your x-axis scale
                //                         .y(function(d) { 
                //                                 return scatterY(d[1]); 
                //                         })(d.density); 
                //                 }); // Adjust this based on your y-axis scale

              

                dots = svg.append('g')
                        .selectAll('dot')
                        .data(filteredVolcanos)
                        .enter()
                        .append('circle')
                        .attr('cx', function (d) { return scatterX(d.year); })
                        .attr('cy', function (d) { return scatterY(d.Volcano_explosive_index); })
                        .attr('r', 4 )
                        .style('fill', function (d) {return d.Volcano_explosive_index >= 5 ? 'red' : '#ffaa20'})
                        .attr('transform', 'translate(' + (margin.left*3) + ',' + ( svgWidth/4 + margin.top ) + ')');






                        
                // updateSize when user resizes window
                window.addEventListener("resize", updateSize);
        })

        function populateScatter(svgg) {
                // console.log("populate scatter", filteredVolcanos);
                dots = svgg.append('g')
                        .selectAll('dot')
                        .data(filteredVolcanos)
                        .enter()
                        .append('circle')
                        .attr('cx', function (d) { return scatterX(d.year); })
                        .attr('cy', function (d) { return scatterY(d.Volcano_explosive_index); })
                        .attr('r', 4 )
                        .style('fill', function (d) {return d.Volcano_explosive_index >= 5 ? 'red' : '#ffaa20'})
                        .attr('transform', 'translate(' + (margin.left*3) + ',' + ( svgWidth/4 + margin.top ) + ')');
                        
        }

        function calculateAverageExplosivity() {
                const total = filteredVolcanos.reduce( (sum, volcano) => sum + parseInt(volcano.Volcano_explosive_index), 0);
                const avg = total / filteredVolcanos.length;
                return avg.toFixed(2);
        }
        
        function coord_proj_cx(d) {
                let coords = [  
                        { lat: d['latitude'], long: d['longitude'] },
                        ].map(p => projection([p.long, p.lat]))
                //$: // console.log(coords[0][0])
                return coords[0][0]
        }

        function coord_proj_cy(d) {
                let coords = [
		        { lat: d['latitude'], long: d['longitude'] },
	                ].map(p => projection([p.long, p.lat]))
                //$: // console.log(d['longitude'])
                //$: // console.log(coords[0][1])

                return coords[0][1]
        }

        $: years = d3.group(volcanos, d => d.year);

        $: Tsunami = d3.group(volcanos, d => d.Tsunami_caused);
        $: earthquake = d3.group(volcanos, d => d.Earthquake_caused);

        $: deadly = [volcanos[277], volcanos[517], volcanos[584], volcanos[540], volcanos[291], volcanos[193], volcanos[822], volcanos[574], volcanos[302], volcanos[304]]

        $: US_years = d3.group(US_volcanos, d => d.year);

        $: US_Tsunami = d3.group(US_volcanos, d => d.Tsunami_caused);
        $: US_earthquake = d3.group(US_volcanos, d => d.Earthquake_caused);
        // $: // console.log(deadly)

        function explosive(groups) {
                
                return (2.5 * groups.Volcano_explosive_index)
        }

        //// console.log(points)
        // debugger;

        function showTooltip(d, newX=null, newY=null) {
                let x,y;
                if (newX === null && newY === null) {
                        x = event.pageX + 10;
                        y = event.pageY + 10;
                } else {
                        // Get the position of the erupt circle
                        const circle = document.querySelector('#highlighted');
                        const circleRect = circle.getBoundingClientRect();
                        // console.log(circleRect);
                        const circleX = parseFloat(circleRect.left) + 50;
                        const circleY = parseFloat(circleRect.top) + 700;
                        // Calculate the tooltip position
                        x = circleX;
                        y = circleY;
                }
                // console.log("showTooltip ", x, y);
                d3.select("#tooltip")
                  .style("visibility", "visible")
                  .html(`<b>${d.name}</b><br>Year: ${
                        d.year >= 0 ? d.year : d.year * -1 + " BC"
                  }<br>Location: ${d.location}<br>Explosivity: ${
                        d.Volcano_explosive_index === "" ? "Unknown" : d.Volcano_explosive_index
                  }<br>Destruction Rank (From 1 to 15): ${d.damage_caused_rank}`)
                  .style("left", x +'px' )
                  .style("top", y + 'px' );
                // console.log(d)
        }

        function hideTooltip(d) {
                d3.select("#tooltip")
                  .style('visibility', 'hidden')
        }

        function findCurrentIndex() {
                const total = filteredVolcanos.length;
                if (total == 0) {
                        return "No eruptions recorded";
                }
                const cur = select;
                // If in next eruption mode, show the current progress
                if (nextEruptionMode == 1) {
                        return cur + " out of " + total + " eruptions";
                } else {
                        return "Total " + total + " eruptions";
                }
        }

  	$: mapper = select;

  	$: if (eruptionIndex === 0) {
    		slicing = filteredVolcanos.slice(0, mapper);
  	} 
        
</script>       

<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous">

<div class="buttons">
        {#each yearCategories as category}
                <button
                        class="btn"
                        class:btn-outline-primary={selectedYear !== category}
                        class:btn-primary={selectedYear === category}
                        on:click={() => {
                                filterByYearAndUpdateClass(category);
                                hideTooltip();
                        }}
                >
                        {category}
                </button>
        {/each}
</div>

<div class="buttons">
        {#each Array.from(new Set(US_volcanos.map(d => d.location))) as location}
                <button 
                        class="btn"
                        class:btn-outline-primary={selectedLocation !== location}
                        class:btn-primary={selectedLocation === location}
                        on:click={() => {
                                filterByLocationAndUpdateClass(location);
                                hideTooltip();
                        }}>
                        {location}
                </button>
        {/each}
</div>

<button
  class="one"
  on:click={() => {
        select += select >= filteredVolcanos.length ? 0 : 1;
        eruptionIndex = 0;
        nextEruptionMode = 1;
        updateFilteredData();
        showTooltip(filteredVolcanos[select - 1]);

  }}
  on:mouseleave={() => hideTooltip()}
> Get Next Eruption </button>

<button
  class="all"
  on:click={() => {
        eruptionIndex = 1;
        nextEruptionMode = 0;
        select = 0;
        updateFilteredData();
        hideTooltip();
  }}
  on:mouseleave={() => hideTooltip()}
> Get All Eruptions </button>

<table class="map_plot_split">
        <tr>
                <th class="map_col">
                        <p> {$currentIndex} </p>
                        <div class="volcanos" id='volcanos'>  
                                <svg viewBox="-260 10 1500 650">
                                        <!--    If all the necessary data is loaded in  -->
                                        {#if states && counties && mesh}
                                                <g fill="white" stroke="black">
                                                        {#each states as feature, i}
                                                                <path d={path(feature)} 
                                                                 on:mouseover={() => selected = feature} 
                                                                 class="state" 
                                                                 />
                                                        {/each}
                                                </g>
 
                                                {#each slicing as d, i}
                                                                <!-- svelte-ignore a11y-interactive-supports-focus -->
                                                                <!-- svelte-ignore a11y-mouse-events-have-key-events -->
                                                                <circle
								 class="erupt"
                                                                 cx={coord_proj_cx(d)}
                                                                 cy={coord_proj_cy(d)}
                                                                 r={4 * ( d.Volcano_explosive_index ) + 10 }
                                                                 fill={d.Volcano_explosive_index >= 5 ? 'red' : '#ffca20'}
                                                                 opacity={0.6}
                                                                 stroke="gray"
                                                                 role="button"
                                                                 order={`${d.year}`}
                                                                 aria-label={`Volcano ${d.name}, Year: ${d.year}, Location: ${d.location}`}
                                                                 on:mouseover={() => showTooltip(d)}
                                                                 on:mouseleave={() => hideTooltip(d)}
                                                                />
							{#if i + 1 === select}
                						<circle
                                                                 class="erupt"
                                                                 id='highlighted'
                                                                 cx={coord_proj_cx(d)}
                                                                 cy={coord_proj_cy(d)}
                                                                 r={(4*(d.Volcano_explosive_index)+4)/2}
                                                                 fill='black'
                						/>
								<circle
                                                                 class="special"
                                                                 cx={coord_proj_cx(d)}
                                                                 cy={coord_proj_cy(d)}
                                                                 r={(30)}
                                                                 fill="blue"
								 on:mouseover={() => showTooltip(d)}
                                                                 on:mouseleave={() => hideTooltip(d)}
                                                                />
                					{/if}
                                                {/each}

                                                <circle cx="950" cy="500" r="30" fill="orange" opacity={0.6} />
                                                <text x="1000" y="500" font-size="20px"> Explosivity less than 5 </text>
                                                <circle cx="950" cy="600" r="45" fill="red" opacity={0.6} />
                                                <text x="1000" y="600" font-size="20px"> Explosivity greater </text>
                                                <text x="1000" y="620" font-size="20px"> than or equal to 5 </text>
                                        {/if}
                                </svg>
                        
                        </div>
                </th>
                <th class="plot_col">
                        <div id="scatter">
                        </div>
                        <p>Total # Eruptions: {filteredVolcanos.length}</p>
                        <p>Avg Explosivity: {calculateAverageExplosivity(filteredVolcanos)}</p>
                </th>
        </tr>
</table>

<div id='tooltip'/>
<style>
        #tooltip {
            background-color: white;
            padding: 8px;
            border: 1px solid black;
            border-radius: 4px;
            position: absolute;
            text-align:left;
            visibility: hidden;

        }

	.volcanos {
                animation: fadeIn 1s;
                animation-delay: 0s; 
        }

        @keyframes fadeIn {
                0% { opacity: 0; }
                100% { opacity: 1; }
        }

	@keyframes fadeOut {
                100% { opacity: 0; }
                0% { opacity: 1; }
        }
        .map_plot_split {       
                text-align: center;
                width: 100%;
        }
        .map_col {
                width: 60%;
        }
        .plot_col {
                width: 40%;
        }

	.all {
    		background-color: #8b0000;
    		border: white;    
    		transition-duration: 0.4s;
    		color: white;
    		margin:5px;  
    		text-align: center;  
  	}

  	.all:hover{
        	background-color: red;
        	border :#04AA6D;
        }
  
  	.one {
    		background-color: orange;
    		border: white;    
    		transition-duration: 0.4s;
    		color: white;    
  	}

  	.one:hover{
        	background-color: red;
        	border :#04AA6D;
    	}

 	.erupt {
    		animation: fadeIn 0.5s;
  	}

        .special {
                animation: fadeIn 2s;
                animation: fadeOut 5s;
                animation-fill-mode: forwards;
        } 
</style>
