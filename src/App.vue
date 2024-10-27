<!-- eslint-disable no-unused-vars -->
<template>
  <div class="flex flex-col items-center h-screen">
    <div class="flex flex-row items-center justify-center w-2/3 gap-3 p-4">
      <button @click="importGraph" class="p-1.5 text-white bg-purple-500 rounded">Import Graph</button>
      <button @click="exportGraph" class="p-1.5 text-white bg-purple-500 rounded">Export Graph</button>
      <button @click="addNode" class="p-1.5 text-white bg-purple-500 rounded">Add Node</button>
      <button @click="animate" class="p-1.5 text-white bg-purple-500 rounded">Animate</button>
    </div>
    <input type="file" ref="fileInput" accept=".json" class="hidden" @change="handleFileChange" />
    <div id="graph-container" class="w-full h-full">
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Swal from 'sweetalert2'
import * as d3 from 'd3'

const fileInput = ref(null)

//!Uncomment if need sample data
const graph = ref( [
  { id: 1, name: "node 1", edges: [2, 3, 4], color: "#FFFFFF" },
  { id: 2, name: "node 2", edges: [1, 3], color: "#FFFFFF" },
  { id: 3, name: "node 3", edges: [1, 2, 4, 5], color: "#FFFFFF" },
  { id: 4, name: "node 4", edges: [1, 3], color: "#FFFFFF" },
  { id: 5, name: "node 5", edges: [3], color: "#FFFFFF" }
] )

const createNode = (name, id) => {
  const newNode = {
    id: id,
    name: name,
    edges: [],
    color: "#FFFFFF"
  }
  graph.value.push(newNode)
}

const linkNode = (nodeId, nodes) => {
  const mainNode = graph.value.find(node => node.id === nodeId)
  if (mainNode) {
    nodes.forEach(edge => {
      if (!mainNode.edges.includes(edge)) {
        mainNode.edges.push(edge)
      }
    })
  }
  nodes.forEach(edgeNodeId => {
    const edgeNode = graph.value.find(node => node.id === edgeNodeId)
    if (edgeNode)
      if (!edgeNode.edges.includes(nodeId))
        edgeNode.edges.push(nodeId)
  })
}

const addNode = async () => {
  const { value: nodeName } = await Swal.fire({
    title: 'Nuevo nodo',
    input: "text",
    inputLabel: "Ingresa el nombre del nuevo nodo"
  })
    
  const nodeId = graph.value.length + 1
  createNode(nodeName, nodeId)

  const html = graph.value.map(node =>
    `<label><input type="checkbox" name="nodes" value="${node.id}" /> ${node.name}</label><br>`
  ).join('')

  const result = await Swal.fire({
    title: 'Enlaces al nuevo nodo',
    html,
    showCancelButton: false,
    preConfirm: () => {
      const selectedNodeIds = []
      const checkboxes = document.querySelectorAll('input[name="nodes"]:checked')
      checkboxes.forEach(checkbox => { selectedNodeIds.push(parseInt(checkbox.value)) })
      return selectedNodeIds
    }
  })
  if (!result.isConfirmed)
    return

  const nodeConnection = result.value || []
  linkNode(nodeId, nodeConnection)
  renderGraph()
  console.log(graph)
}


const renderGraph = () => {
  //d3.select("#graph-container").selectAll("svg").remove()

  const container=d3.select("#graph-container");
  const transform = container.select("svg").empty() 
    ? d3.zoomIdentity 
    : d3.zoomTransform(container.select("svg").node()); // Si hay, usar la actual

  container.selectAll("svg").remove()
  const svg = container
    .append("svg")
    .attr("width", "100%")
    .attr("height", "100%")
    .call(d3.zoom().on("zoom", (event) => {
      svg.attr("transform", event.transform)
    }))
    .append("g")
    .attr("transform",transform);

  const width = document.getElementById('graph-container').clientWidth;
  const height = document.getElementById('graph-container').clientHeight;

  const simulation = d3.forceSimulation(graph.value)
    .force("link", d3.forceLink().id(d => d.id))
    .force("charge", d3.forceManyBody().strength(-300))
    .force("center", d3.forceCenter(width / 2, height / 2))

  const link = svg.append("g")
    .attr("class", "links")
    .selectAll("line")
    .data(graph.value.flatMap(node => node.edges.map(e => ({ source: node.id, target: e }))))
    .enter()
    .append("line")
    .attr("stroke", d => {
      const source = graph.value.find(node => node.id === d.source);
      const target = graph.value.find(node => node.id === d.target);
      return source.color === "#FF0000" && target.color === "#FF0000" ? "#FF0000" : "#000";
    })
    .attr("stroke-width", 5)

  const node = svg.append("g")
    .attr("class", "nodes")
    .selectAll("g")
    .data(graph.value)
    .enter()
    .append("g")
    .call(d3.drag()
      .on("start", (event, d) => {
        event.sourceEvent.stopPropagation()
        if (!event.active) simulation.alphaTarget(0.3).restart()
        d.fx = d.x
        d.fy = d.y
      })
      .on("drag", (event, d) => {
        event.sourceEvent.stopPropagation()
        d.fx = event.x
        d.fy = event.y
      })
      .on("end", (event, d) => {
        event.sourceEvent.stopPropagation()
        if (!event.active) simulation.alphaTarget(0)
        d.fx = null
        d.fy = null
      }))

  node.append("circle")
    .attr("r", 15)
    .attr("fill", d => d.color || "#FFFFFF")
    .attr("stroke", "#000000")
    .attr("stroke-width", 5)

  node.append("text")
    .attr("dy", ".35em")
    .attr("x", 0)
    .attr("y", 0)
    .attr("text-anchor", "middle")
    .attr("font-size", "2em")
    .attr("fill", "#000000")
    .text(d => d.name)

  simulation.on("tick", () => {
    link
      .attr("x1", d => graph.value.find(node => node.id === d.source).x)
      .attr("y1", d => graph.value.find(node => node.id === d.source).y)
      .attr("x2", d => graph.value.find(node => node.id === d.target).x)
      .attr("y2", d => graph.value.find(node => node.id === d.target).y)

    node
      .attr("transform", d => `translate(${d.x},${d.y})`)
  })
}

const importGraph = () => {
  fileInput.value.click()
}

const handleFileChange = (event) => {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      try {
        const json = JSON.parse(e.target.result)
        if (checkJSONFormat(json)) {
          graph.value = json
          renderGraph()
          Swal.fire('Success', 'Graph imported successfully!', 'success')
        } else {
          Swal.fire('Error', 'Invalid graph format!', 'error')
        }
      } catch (error) {
        Swal.fire('Error', `Invalid JSON file! ${error}`, 'error')
      }
    }
    reader.readAsText(file)
  }
}

const checkJSONFormat = (json) => {
  if (!Array.isArray(json)) return false

  for (const node of json) {
    if (typeof node.id !== 'number' ||
      typeof node.name !== 'string' ||
      !Array.isArray(node.edges) ||
      typeof node.color !== 'string') {
      return false
    }
  }
  return true
}

const exportGraph = () => {
  const jsonData = JSON.stringify(graph.value, null, 2)

  const blob = new Blob([jsonData], { type: 'application/json' })

  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = 'graph.json'

  link.click()

  URL.revokeObjectURL(url)
}

const animate = () => { bfs(graph.value[0].id) }

const bfs = async (startNodeId) => {
  const visited = new Set();
  const queue = [];
  const animationDelay = 1000;


  const startNode = graph.value.find(node => node.id === startNodeId);
  if (!startNode) {
    console.log("Start node not found");
    return;
  }

  graph.value.forEach(node =>{
    node.fx=node.x;
    node.fy=node.y;
  });

  visited.add(startNode.id);
  queue.push(startNode);

  
  while (queue.length > 0) {
    const currentNode = queue.shift();


    currentNode.color = "#FF0000";
    console.log(`Visited node: ${currentNode.name}`);


    renderGraph();
    await delay(animationDelay);


    for (const neighborId of currentNode.edges) {
      const neighborNode = graph.value.find(node => node.id === neighborId);

      if (neighborNode && !visited.has(neighborNode.id)) {
        visited.add(neighborNode.id);
        queue.push(neighborNode);


        d3.selectAll("line")
          .filter(d => (d.source === currentNode.id && d.target === neighborNode.id) ||
            (d.source === neighborNode.id && d.target === currentNode.id))
          .attr("stroke", "#FF0000");

        renderGraph();
        await delay(animationDelay);
      }
    }
  }

  graph.value.forEach(node=>{
    node.fx=null;
    node.fy=null;
  })
};


const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));

onMounted(() => {
  renderGraph()
})
</script>
