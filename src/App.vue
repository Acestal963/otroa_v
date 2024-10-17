<template>
  <div class="flex flex-col items-center h-screen">
    <div class="flex flex-row items-center justify-center w-2/3 gap-3 p-4">
      <button @click="importGraph" class="p-1.5 text-white bg-purple-500 rounded">Import Graph</button>
      <button @click="exportGraph" class="p-1.5 text-white bg-purple-500 rounded">Export Graph</button>
      <button @click="addNode" class="p-1.5 text-white bg-purple-500 rounded">Add Node</button>
      <button @click="animate" class="p-1.5 text-white bg-purple-500 rounded">Animate</button>
    </div>
    <div id="graph-container" class="w-full h-full">
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, render } from 'vue'
import Swal from 'sweetalert2'
import * as d3 from 'd3'

const graph = ref([
  { id: 1, name: "node 1", edges: [2, 3, 4], color: "#FFFFFF" },
  { id: 2, name: "node 2", edges: [1, 3], color: "#FFFFFF" },
  { id: 3, name: "node 3", edges: [1, 2, 4, 5], color: "#FFFFFF" },
  { id: 4, name: "node 4", edges: [1, 3], color: "#FFFFFF" },
  { id: 5, name: "node 5", edges: [3], color: "#FFFFFF" }
])

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
  // Remove any existing SVG elements before rendering the updated graph
  d3.select("#graph-container").selectAll("svg").remove()

  const svg = d3.select("#graph-container")
    .append("svg")
    .attr("width", "100%")
    .attr("height", "100%")
    .call(d3.zoom().on("zoom", (event) => {
      svg.attr("transform", event.transform)
    }))
    .append("g")

  const width = 800
  const height = 600

  const simulation = d3.forceSimulation(graph.value)
    .force("link", d3.forceLink().id(d => d.id))
    .force("charge", d3.forceManyBody().strength(-500))
    .force("center", d3.forceCenter(width / 2, height / 2))

  // Create links (edges)
  const link = svg.append("g")
    .attr("class", "links")
    .selectAll("line")
    .data(graph.value.flatMap(node => node.edges.map(e => ({ source: node.id, target: e }))))
    .enter()
    .append("line")
    .attr("stroke", "#999")
    .attr("stroke-width", 2)

  // Create nodes
  const node = svg.append("g")
    .attr("class", "nodes")
    .selectAll("g")
    .data(graph.value)
    .enter()
    .append("g")
    .call(d3.drag()
      .on("start", (event, d) => {
        event.sourceEvent.stopPropagation() // Prevent zoom during drag
        if (!event.active) simulation.alphaTarget(0.3).restart()
        d.fx = d.x
        d.fy = d.y
      })
      .on("drag", (event, d) => {
        event.sourceEvent.stopPropagation() // Prevent zoom during drag
        d.fx = event.x
        d.fy = event.y
      })
      .on("end", (event, d) => {
        event.sourceEvent.stopPropagation() // Prevent zoom during drag
        if (!event.active) simulation.alphaTarget(0)
        d.fx = null
        d.fy = null
      }))

  // Append circles for nodes
  node.append("circle")
    .attr("r", 15)
    .attr("fill", d => d.color || "#FFFFFF")
    .attr("stroke", "#000000")
    .attr("stroke-width", 2)

  // Append text to each node
  node.append("text")
    .attr("dy", ".35em")
    .attr("x", d => 0)
    .attr("y", d => 0)
    .attr("text-anchor", "middle")
    .attr("font-size", "10px")
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


const importGraph = () => {}

const exportGraph = () => {}

const animate = () => {}

onMounted(() => {
  renderGraph()
})
</script>

