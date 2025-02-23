<script>
  
  import { onMount } from "svelte";
  import * as d3 from 'd3';
  export let graph;

  let canvas;
  let width = 500;
  let height = 600;
  let max = 1000;
  const nodeRadius = 5;
  let activeNode = false;
  const padding = { top: 20, right: 40, bottom: 40, left: 25 };

  $: xScale = d3.scaleLinear()
    .domain([0, 20])
    .range([padding.left, width - padding.right]);

  $: yScale = d3.scaleLinear()
    .domain([0, 12])
    .range([height - padding.bottom, padding.top]);

  $: xTicks = width > 180 ? [0, 4, 8, 12, 16, 20] : [0, 10, 20];

  $: yTicks = height > 180 ? [0, 2, 4, 6, 8, 10, 12] : [0, 4, 8, 12];

  $: d3yScale = d3.scaleLinear().domain([0, height]).range([height, 0]);

  $: links = graph.links.map((d) => Object.create(d));
  $: nodes = graph.nodes.map((d) => {
    d.size = Math.pow(graph.links
      .filter((link) => link.source == d.id || link.target == d.id)
      .map((link) => link.value)
      .reduce((a, b) => a + b), 2);
    d.connections = graph.links
      .filter((link) => link.source == d.id || link.target == d.id)
      .map((link) => 1)
      .reduce((a, b) => a + b);
    if (d.id == "u") {
      max = d.size;
      d.details.messages = max;
    }
    return Object.create(d);
  });
  

  function groupColour(context, d) {
    let nodesize = 2 + Math.sqrt(d.size) / 1;
    let radgrads = ["#9b5fe0", "#16a4d8", "#60dbe8", "#8bd346", "#efdf48", "#f9a52c",  "#d64e12"];
    return radgrads[d.group];
  }
  let showCard;
  let transform = d3.zoomIdentity;
  let simulation, context;
  let dpi = 1;
  let initialZoomScale = 0.06; 
  let searchQuery = '';
  let activeLinks = []; 

  $: filteredNodes = nodes.filter(node => node.name.toLowerCase().includes(searchQuery.toLowerCase()));

  onMount(() => {
    dpi = window.devicePixelRatio || 1;
    context = canvas.getContext("2d");
    resize();

    simulation = d3
      .forceSimulation(nodes)
      //.alphaDecay(0.3)
      //.velocityDecay(0.5)
      .alphaTarget(0) // stay hot
      //.velocityDecay(0.9)
      .force(
        "link",
        d3
          .forceLink(links)
          .id((d) => d.id)
          .distance(
            (d) => Math.pow(2, -d.value / 1) / 10//10
            //(d) => 2 + Math.sqrt(max) / 4 + 130 * Math.pow(2, -d.value / 1)
          )
      )
      .force("charge", d3.forceManyBody())
      .force("center", d3.forceCenter(width / 2, height / 2))
      .force('collision', d3.forceCollide().radius((d) => Math.sqrt(d.size)/3))
      .on("tick", simulationUpdate);

    // title
    d3.select(context.canvas).on("mousemove", (event) => {
      const d = simulation.find(
        transform.invertX(event.offsetX * dpi),
        transform.invertY(event.offsetY * dpi),
        1000
      );
      if (d) activeNode = d;
      else activeNode = false;

      simulationUpdate();
    });

    d3.select(context.canvas).on("click", () => {
      if (activeNode) {
        showCard = JSON.parse(
          JSON.stringify({ id: activeNode.id, details: activeNode.details })
        );
      }
      //each {}
    });

    let zoom = d3.zoom()
      .scaleExtent([0, 10])
      .on("zoom", zoomed);
    let initialTranslate = [width / 2.5, height / 2.5];
    d3.select(canvas)
      .call(
        d3
          .drag()
          .container(canvas)
          .subject(dragsubject)
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended)
      )
      .call(zoom)
      .call(zoom.transform, d3.zoomIdentity.translate(initialTranslate[0], initialTranslate[1]).scale(initialZoomScale));

  
      d3.select(canvas).dispatch('click');

  });

  function zoomed(currentEvent) {
    transform = currentEvent.transform;
  simulationUpdate();
}

function simulationUpdate() {
    context.save();
    context.clearRect(0, 0, context.canvas.width, context.canvas.height);
    context.translate(transform.x, transform.y);
    context.scale(transform.k, transform.k);
    activeLinks = [];

    if (activeNode) {
        activeLinks = links.filter(link => link.source.id === activeNode.id || link.target.id === activeNode.id);
    }

    links.forEach((d) => {
      context.beginPath();
      context.moveTo(d.source.x, d.source.y);
      context.lineTo(d.target.x, d.target.y);
      context.globalAlpha = activeLinks.some(link => link === d) ? 1 : 0.3;
      context.strokeStyle = "#999";
      context.lineWidth = Math.cbrt(d.value) / (activeLinks.some(link => link === d) ? 0.25 : 1);
      context.stroke();
      context.globalAlpha = 0.9;
    });

    nodes.forEach((d, i) => {
      if (activeNode !== false) {
        if (activeNode && (d.id === activeNode.id || activeLinks.some(link => link.source.id === d.id || link.target.id === d.id))) {
          context.beginPath();
          context.arc(d.x, d.y, 2 + Math.sqrt(d.size) / 5, 0, 2 * Math.PI);
          context.strokeStyle = "solid";
          context.lineWidth = 1.5;
          context.stroke();
          context.fillStyle = groupColour(context, d);
          context.fill();
        }
        else {
          context.beginPath();
          context.arc(d.x, d.y, 2 + Math.sqrt(d.size) / 5, 0, 2 * Math.PI);
          context.strokeStyle = "solid";
          context.lineWidth = 1.5;
          context.stroke();
          context.fillStyle = "#555"; // change to groupColour(context, d) for color instead
          context.globalAlpha = 0.7; // Adjust transparency here
          context.fill();
          context.globalAlpha = 0.9;
        }
      }
      else {
        context.beginPath();
        context.arc(d.x, d.y, 2 + Math.sqrt(d.size) / 5, 0, 2 * Math.PI);
        context.strokeStyle = "solid";
        context.lineWidth = 1.5;
        context.stroke();
        context.fillStyle = groupColour(context, d);
        context.fill();
      }
      
      context.font = 'lighter 100px sans-serif';
      if (d.size > 5000000) {
        context.fillStyle = "black";
        d.name
          .split(/(?=[A-Z])/)
          .forEach((word, index) =>
            context.fillText(d.name, d.x - (d.name.length * 25), d.y)
          );
      }
      else {
        context.font = 'lighter 20px sans-serif';
        context.fillStyle = "black";
        d.name
          .split(/(?=[A-Z])/)
          .forEach((word, index) =>
            context.fillText(d.name, d.x - (d.name.length * 5), d.y)
          );
      }
    });
    context.restore();
}


  // Use the d3-force simulation to locate the node
  
  function dragsubject(currentEvent) {
    const node = simulation.find(
      transform.invertX(currentEvent.x * dpi),
      transform.invertY(currentEvent.y * dpi),
      800
    );
    if (node) {
      node.x = transform.applyX(node.x);
      node.y = transform.applyY(node.y);
    }
    return node;
  }

  function dragstarted(currentEvent) {
    if (!currentEvent.active) simulation.alphaTarget(0.01).restart();
    currentEvent.subject.fx = transform.invertX(currentEvent.subject.x);
    currentEvent.subject.fy = transform.invertY(currentEvent.subject.y);
  }

  function dragged(currentEvent) {
    currentEvent.subject.fx = transform.invertX(currentEvent.x);
    currentEvent.subject.fy = transform.invertY(currentEvent.y);
  }

  function dragended(currentEvent) {
    if (!currentEvent.active) simulation.alphaTarget(0);
    currentEvent.subject.fx = null;
    currentEvent.subject.fy = null;
  }
  
  function resize() {
    ({ width, height } = canvas);
  }
  function fitToContainer(node) {
    dpi = window.devicePixelRatio || 1;
    // Make it visually fill the positioned parent
    node.style.width = "100%";
    node.style.height = "100%";
    // ...then set the internal size to match
    node.width = node.offsetWidth * dpi;
    node.height = node.offsetHeight * dpi;
    width = node.offsetWidth * dpi;
    height = node.offsetHeight * dpi;
  }

  // function for search box, stops working after a while?
  function updateSearch(event) {
  const query = event.target.value.trim().toLowerCase();
  const matchedNodes = nodes.filter(node => node.name.toLowerCase().includes(query));

  if (matchedNodes.length > 0) {
    // Find the node with the largest size among duplicates
    activeNode = matchedNodes.reduce((maxNode, node) => {
      return maxNode.size > node.size ? maxNode : node;
    });
    // Update activeLinks
    activeLinks = links.filter(link => link.source.id === activeNode.id || link.target.id === activeNode.id);
  } else if (query === '') {
    activeNode = false; 
    activeLinks = [];
  }

  simulationUpdate();
}
  

</script>

<svelte:window on:resize={resize} />

<div on:resize={resize} class="container">
  <input type="text" placeholder="Type to search..." on:input={updateSearch} class="search-box" />
  {#if activeNode}
    <breadcrumb id="nodeDetails">
      <strong><b>{activeNode.name}</b></strong>
      <br>
      Interactions: {Math.sqrt(activeNode.size)}
      <br>
      Connections: {activeNode.connections}
      {#if activeNode.details}
        {#each Object.entries(activeNode.details) as detail}
          {detail[0]}:
          {detail[1]}
          <br />
        {/each}
      {/if}
    </breadcrumb>
  {/if}
  <canvas use:fitToContainer bind:this={canvas} />
</div>

<style>
	:global(body){background-color: #ffffff; font-family: sans-serif; font-weight: lighter;}
  canvas {
    float: right;
  }
  .container {
    width: 80%;
    height: 100%;
    position: relative;
    border-color: #000000;
    border-style: solid;
    border: 10px;
    background-color: light-grey;
  }
  #nodeDetails {
    position: absolute;
    top: 1%;
    left: 1%;
    width: unset;
		color:#000000;
  }
  .search-box {
    position: absolute;
    bottom: 10px;
    left: 10px;
    width: 200px; /* Adjust width as needed */
    height: 30px; /* Adjust height as needed */
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }
</style>