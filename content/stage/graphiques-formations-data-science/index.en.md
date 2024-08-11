+++
title = "Data Science Training Graphs"
+++

## 3. 2D Graph with D3.js and 3D Graph with Three.js for Data Science Training

The third task I had to complete was creating a 2D graph with D3.js and a 3D graph with Three.js for data science training.

Unlike the second task, this one was not performed locally on my machine but on [Google Colab](https://colab.google/).

### 1. 2D Graph with D3.js

For the 2D graph, I had to create a [3D Force Graph](https://github.com/vasturiano/3d-force-graph) in 2D. Since this is different from a tree of life, I couldn't reuse all of what I had done before, only some parts that were similar in both formats.

Once again, we use the <abbr title="Content Delivery Network">CDN</abbr> to import D3.js.

```js
    import * as d3 from 'https://cdn.skypack.dev/d3@7';
```

Next, we have a variable `timeoutId` that will allow us to update automatically at set intervals.

```js
    let timeoutId = undefined;
```

We then have the `draw` function, which is similar to the `updateTree` function, to display the graph. However, this time, we also have functions within this function to handle drag and drop events on the nodes.

```js
function draw({ source, data }) {
    // Define the container size
    const width = 800;
    const height = 600;

    d3.select("#visualization").select("svg").remove();

    // Create SVG container
    const svg = d3.select('#visualization')
        .append('svg')
        .attr('width', width)
        .attr('height', height);

    // Create a group for zooming
    const zoomGroup = svg.append('g');

    // Create force simulation
    const simulation = d3.forceSimulation(data.nodes)
        .force('link', d3.forceLink(data.links).id(d => d.id)) // Use d3.forceLink for the links
        .force('charge', d3.forceManyBody().strength(-50))
        .force('center', d3.forceCenter(width / 2, height / 2));

    // Append links
    const links = zoomGroup
        .selectAll('line')
        .data(data.links)
        .enter()
        .append('line')
        .attr('stroke', '#fff')
        .attr('stroke-width', 2);

    // Append group instead of node to add a text
    const node = zoomGroup
        .append("g")
        .attr("fill", "orange")
        .attr("stroke", "#fff")
        .attr("stroke-opacity", 1.5)
        .attr("stroke-width", 2)
        .selectAll("g")
        .data(data.nodes)
        .join("g")
        .call(drag(simulation));

    // Append circle to node "g"
    node
        .append('circle')
        .attr('r', 7.5);

    // Append text to node "g"
    node
        .append('text')
        .text(d => d.id)
        .attr("fill", "black")
        .attr("stroke", "none")
        .attr("font-size", "0.8em");

    // Add zoom behavior
    const zoom = d3.zoom()
        .scaleExtent([0.1, 4]) // Set the zoom scale range
        .on("zoom", (event) => {
            zoomGroup.attr("transform", event.transform);
        });

    // Apply the zoom behavior to the SVG
    svg.call(zoom);

    // Update simulation at each tick
    simulation.on('tick', () => {
        links
            .attr('x1', d => d.source.x)
            .attr('y1', d => d.source.y)
            .attr('x2', d => d.target.x)
            .attr('y2', d => d.target.y);

        node
            .attr('transform', d => `translate(${d.x},${d.y})`);
        });

    // Function to handle drags event
    function drag(simulation) {
        function dragstarted(event) {
            if (!event.active) simulation.alphaTarget(0.3).restart();
            event.subject.fx = event.subject.x;
            event.subject.fy = event.subject.y;
        }

        function dragged(event) {
            event.subject.fx = event.x;
            event.subject.fy = event.y;
        }

        function dragended(event) {
            if (!event.active) simulation.alphaTarget(0);
            event.subject.fx = null;
            event.subject.fy = null;
        }

        return d3.drag()
            .on("start", dragstarted)
            .on("drag", dragged)
            .on("end", dragended);
    }
}
```

After the `draw` function, we have the `dataFetch` function which makes a request to this route `http://localhost:8000/data` to fetch data from a local server on Google Colab and update the graph. The `timeoutId` variable is used to define the interval between each call.

```js
function dataFetch({source}) {
    console.log('dataFetch', {source});
    if(timeoutId !== undefined){
        clearTimeout(timeoutId);
    }
    fetch('http://localhost:8000/data')
        .then(response => response.json())
        .then(data => {
            draw({source, data});
            timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
        })
        .catch(error => {
            console.error('Error:', error);
            timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
        });
}
```

Finally, we have an `addEventListener` on a button to handle the button's action and call the `dataFetch` function. Then we call `dataFetch` to fetch the data and display the graph for the first time.

```js
document.getElementById('refreshButton').addEventListener(
    'click',
    () => dataFetch({source: 'refreshButton'})
);

dataFetch({source: 'initialization'});
```

<details>
    <summary>View full script</summary>

    import * as d3 from 'https://cdn.skypack.dev/d3@7';

    const visualizationElement = document.getElementById('visualization');
    let timeoutId = undefined;

    function draw({ source, data }) {
      // Define the container size
      const width = 800;
      const height = 600;

      d3.select("#visualization").select("svg").remove();

      // Create SVG container
      const svg = d3.select('#visualization')
        .append('svg')
        .attr('width', width)
        .attr('height', height);

      // Create a group for zooming
      const zoomGroup = svg.append('g');

      // Create force simulation
      const simulation = d3.forceSimulation(data.nodes)
        .force('link', d3.forceLink(data.links).id(d => d.id)) // Use d3.forceLink for the links
        .force('charge', d3.forceManyBody().strength(-50))
        .force('center', d3.forceCenter(width / 2, height / 2));

      // Append links
      const links = zoomGroup
        .selectAll('line')
        .data(data.links)
        .enter()
        .append('line')
        .attr('stroke', '#fff')
        .attr('stroke-width', 2);

      // Append group instead of node to add a text
      const node = zoomGroup
        .append("g")
        .attr("fill", "orange")
        .attr("stroke", "#fff")
        .attr("stroke-opacity", 1.5)
        .attr("stroke-width", 2)
        .selectAll("g")
        .data(data.nodes)
        .join("g")
        .call(drag(simulation));

      // Append circle to node "g"
      node
        .append('circle')
        .attr('r', 7.5);

      // Append text to node "g"
      node
        .append('text')
        .text(d => d.id)
        .attr("fill", "black")
        .attr("stroke", "none")
        .attr("font-size", "0.8em");

      // Add zoom behavior
      const zoom = d3.zoom()
        .scaleExtent([0.1, 4]) // Set the zoom scale range
        .on("zoom", (event) => {
          zoomGroup.attr("transform", event.transform);
        });

      // Apply the zoom behavior to the SVG
      svg.call(zoom);

      // Update simulation at each tick
      simulation.on('tick', () => {
        links
          .attr('x1', d => d.source.x)
          .attr('y1', d => d.source.y)
          .attr('x2', d => d.target.x)
          .attr('y2', d => d.target.y);

        node
          .attr('transform', d => `translate(${d.x},${d.y})`);
      });

      // Function to handle drags event
      function drag(simulation) {
        function dragstarted(event) {
          if (!event.active) simulation.alphaTarget(0.3).restart();
          event.subject.fx = event.subject.x;
          event.subject.fy = event.subject.y;
        }

        function dragged(event) {
          event.subject.fx = event.x;
          event.subject.fy = event.y;
        }

        function dragended(event) {
          if (!event.active) simulation.alphaTarget(0);
          event.subject.fx = null;
          event.subject.fy = null;
        }

        return d3.drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended);
      }
    }

    function dataFetch({source}) {
        console.log('dataFetch', {source});
        if(timeoutId !== undefined){
          clearTimeout(timeoutId);
        }
        fetch('http://localhost:8000/data')
            .then(response => response.json())
            .then(data => {
                draw({source, data});
                timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
            })
            .catch(error => {
                console.error('Error:', error);
                timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
            });
    }

    document.getElementById('refreshButton').addEventListener(
      'click',
      () => dataFetch({source: 'refreshButton'})
    );

    dataFetch({source: 'initialization'});

</details>

Here is the result of the 2D graph with D3.js:
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/d3-2d-force-graph-result.png?raw=true", alt="2D Graph D3.js Script Result", no_hover=true) }}

### 2. 3D Graph with Three.js

For Three.js, more links are needed to import everything required. That's why we have the code directly in an HTML file.

We start with the head of an HTML file, with the `DOCTYPE`, `html`, and `head` tags. Then, we have a style tag to apply CSS, particularly to the div with the id `visualization`, to set a height of 500px. I realized that the div was hidden in Google Colab, so I found this solution with CSS to force it to take a certain size.

We also have two `script` tags to import `three` and `3d-force-graph`.

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        #visualization {
            height: 500px;
        }
        body {
            margin: 0;
        }
        .node-label {
            font-size: 12px;
            padding: 1px 4px;
            border-radius: 4px;
            background-color: rgba(0,0,0,0.5);
            user-select: none;
        }
    </style>

    <script src="//unpkg.com/three"></script>
    <script src="//unpkg.com/3d-force-graph"></script>
</head>
```

In the body, we find the entire code, including the tags used to display the graph, like the div and the button to refresh the graph.

```html
<body>
    <div id="visualization"></div>
    <button id="refreshButton">Refresh</button>
```

We have two more `script` tags, one to import the build of three.js and another containing all the code to generate the graph.

```html
    <script type="importmap">{ "imports": { "three": "https://unpkg.com/three/build/three.module.js" }}</script>
    <script type="module">
```

The same logic as the 2D graph made with D3.js is followed, with the `threeDataFetch` function to fetch data and initialize the `timeoutId` variable.

```js
    function threeDataFetch({source}) {
        console.log('threeDataFetch', {source});
        if(timeoutId !== undefined){
            clearTimeout(timeoutId);
        }
        fetch('http://localhost:8000/data')
        .then(response => response.json())
        .then(data => {
            draw({source, data});
            timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
        })
        .catch(error => {
            console.error('Error:', error);
            timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
        });
    }
```

We then have the `draw` function to draw the graph. This function is different since we are no longer using D3.js but Three.js, so the methods and functioning are different.

Thanks to the comments, we understand which part affects which aspect of the graph.

- `.graphData`, from its name, we understand it is used to take the data to be displayed.
- The comment `Styling links` lets us know that the two methods following it are for styling the links between the nodes.
- The comment `Styling nodes` lets us know that the methods following it are for styling the nodes.

```js
    function draw({ source, data }) {
        const container = document.getElementById("visualization");

        let hoveredNode;

        const Graph = ForceGraph3D({
            extraRenderers: [new CSS2DRenderer()]
        })(container)
        .graphData(data)

        // Styling links
        .linkWidth(1)
        .linkColor("grey")

        // Styling nodes
        .nodeRelSize(3)
        .nodeOpacity(1)
        .nodeColor(node => {
            if (node == hoveredNode) {
                return "yellow";
            }
            return "grey";
        })
        .onNodeHover((node) => {
            hoveredNode = node;
            Graph.nodeColor(Graph.nodeColor())
        })
        .nodeThreeObject(node => {
            const nodeEl = document.createElement('div');
            nodeEl.textContent = node.id;
            nodeEl.style.color = "white";
            nodeEl.className = 'node-label';
            return new CSS2DObject(nodeEl);
        })
        .nodeThreeObjectExtend(true);
    }
```

<details>
    <summary>View full code</summary>

    <!DOCTYPE html>
    <html>
    <head>
        <style>
          #visualization {
            height: 500px;
          }
          body {
              margin: 0;
          }
          .node-label {
            font-size: 12px;
            padding: 1px 4px;
            border-radius: 4px;
            background-color: rgba(0,0,0,0.5);
            user-select: none;
          }
        </style>

        <script src="//unpkg.com/three"></script>
        <script src="//unpkg.com/3d-force-graph"></script>
        <!--<script src="../../dist/3d-force-graph.js"></script>-->
    </head>
    <body>
      <div id="visualization"></div>
      <button id="refreshButton">Refresh</button>

      <script type="importmap">{ "imports": { "three": "https://unpkg.com/three/build/three.module.js" }}</script>
      <script type="module">
        import { CSS2DRenderer, CSS2DObject } from '//unpkg.com/three/examples/jsm/renderers/CSS2DRenderer.js';

        let timeoutId = undefined;

        function draw({ source, data }) {
            const container = document.getElementById("visualization");

            let hoveredNode;

            const Graph = ForceGraph3D({
              extraRenderers: [new CSS2DRenderer()]
            })(container)
            .graphData(data)

            // Styling links
            .linkWidth(1)
            .linkColor("grey")

            // Styling nodes
            .nodeRelSize(3)
            .nodeOpacity(1)
            .nodeColor(node => {
              if (node == hoveredNode) {
                return "yellow";
              }
              return "grey";
            })
            .onNodeHover((node) => {
              hoveredNode = node;
              Graph.nodeColor(Graph.nodeColor())
            })
            .nodeThreeObject(node => {
              const nodeEl = document.createElement('div');
              nodeEl.textContent = node.id;
              nodeEl.style.color = "white";
              nodeEl.className = 'node-label';
              return new CSS2DObject(nodeEl);
            })
            .nodeThreeObjectExtend(true);
        }

        function threeDataFetch({source}) {
          console.log('threeDataFetch', {source});
          if(timeoutId !== undefined){
            clearTimeout(timeoutId);
          }
          fetch('http://localhost:8000/data')
            .then(response => response.json())
            .then(data => {
                draw({source, data});
                timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
            })
            .catch(error => {
                console.error('Error:', error);
                timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
            });
        }

        document.getElementById('refreshButton').addEventListener(
          'click',
          () => threeDataFetch({source: 'refreshButton'})
        );

        threeDataFetch({source: 'initialization'});
      </script>
    </body>
    </html>

</details>

Here is the result of this graph:
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/three-3d-force-graph-result.png?raw=true", alt="3D Graph Three.js Script Result", no_hover=true) }}

The yellow node you see is not grey because I deliberately hovered over it with the mouse to show the color change effect when hovering over a node.

<section class="task-nav">
    <a href="/en/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">All tasks</span>
        </div>
    </a>
    <a href="/en/stage/exportation-godot">
        <div>
            <span class="label">First task</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
