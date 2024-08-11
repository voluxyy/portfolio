+++
title = "Graphiques formations data science"
+++

## 3. Graphique 2D avec D3.js et graphique 3D avec three.js pour la formation data science

La troisième tâche que j'ai eu à faire fut la suivante : réaliser un graphique 2D avec D3.js et un graphique 3D avec three.js pour la formation data science.

Contrairement à la deuxième tâche, celle-ci n'a pas été réalisée en local sur ma machine mais sur [Google Colab](https://colab.google/).

### 1. Graphique 2D avec D3.js

Pour le graphique 2D, j'ai dû faire un [3D Force Graph](https://github.com/vasturiano/3d-force-graph) en 2D. Puisque c'est différent d'un tree of life, je n'ai pas pu récupérer l'intégralité de ce que j'avais fait, seulement quelques parties similaires aux deux formats.

Nous retrouvons une fois de plus, le <abbr title="Content Delivery Network">CDN</abbr> pour importer D3.js.

```js
    import * as d3 from 'https://cdn.skypack.dev/d3@7';
```

Ensuite nous avons une variable `timeoutId` qui va nous permettre de mettre à jour automatiquement avec un intervalle de temps.

```js
    let timeoutId = undefined;
```

Nous retrouvons la fonction `draw`, qui s'apparente à la fonction `updateTree`, pour afficher le graphique sauf que cette fois-ci nous retrouvons également des fonctions dans cette fonction pour gérer les événements de drag and drop sur les noeuds.

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

Après la fonction `draw`, nous avons la fonction `dataFetch` qui permet de faire une requête sur cette route `http://localhost:8000/data` pour récupérer les données depuis un serveur local sur le Google Colab et mettre à jour le graphique. Tout en utilisant la variable `timeoutId` afin de définir l'intervalle de temps entre chaque appelle.

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

Pour terminer, nous retrouvons un `addEventListener` sur un bouton pour gérer l'action du bouton et appeler la fonction `dataFetch`. Puis nous appelons `dataFetch` pour récupérer les données et afficher le graphique une première fois.

```js
document.getElementById('refreshButton').addEventListener(
    'click',
    () => dataFetch({source: 'refreshButton'})
);

dataFetch({source: 'initialization'});
```

<details>
    <summary>Voir le script en entier</summary>

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

Voici le résultat du script pour le graphique 2D avec D3.js :
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/d3-2d-force-graph-result.png?raw=true", alt="Resultat script graphique 2D D3.js", no_hover=true) }}

### 2. Graphique 3D avec three.js

Pour three.js, il y a besoin de beaucoup plus de liens pour importer tout ce dont nous avons besoin. C'est pour ça que nous avons le code directement dans un fichier html.

Nous avons donc l'en tête d'un fichier html avec la balise `DOCTYPE`, `html` et `head`. Ensuite nous avons une balise style pour appliquer du css, notamment à la balise div avec l'id `visualization`, pour appliquer une hauteur de 500px. Je me suis rendu compte que la div était cachée dans Google Colab, donc j'ai trouvé cette solution avec le CSS pour la forcer à prendre une certaine taille.

Nous avons également deux balises `script` pour importer `three` et `3d-force-graph`.

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

Dans le body nous retrouvons l'intégralité du code y compris les balises utilisés pour afficher le graphique comme la div et le bouton pour refresh le graphique.

```html
<body>
    <div id="visualization"></div>
    <button id="refreshButton">Refresh</button>
```

Nous avons encore deux balises `script`, une pour importer le build de three.js et l'autre qui contient tout le code pour générer le graphique.

```html
    <script type="importmap">{ "imports": { "three": "https://unpkg.com/three/build/three.module.js" }}</script>
    <script type="module">
```

Nous retrouvons la même logique que le graphique 2D fait avec D3.js avec la fonction `threeDataFetch` pour récupérer les données et initialiser la variable `timeoutId`.

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

Nous avons de nouveau, la fonction `draw` pour dessiner le graphique. Cette fonction est différente puisque nous n'utilisons plus D3.js mais three.js donc les méthodes et le fonctionnement est différent.

Grâce au commentaire, nous comprenons quelle partie agit sur quelle aspect du graphique.

- `.graphData`, de son nom, nous comprenons qu'elle sert à prendre la data à afficher.
- Le commentaire `Styling links` nous fait comprendre que les deux méthodes à sa suite sont pour styliser les liens entre les noeuds.
- Le commentaire `Styling nodes` nous fait comprendre que les méthodes à sa suite sont pour styliser les noeuds.

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
    <summary>Voir le code en entier</summary>

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

Voici le résultat de ce graphique :
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/three-3d-force-graph-result.png?raw=true", alt="Resultat script graphique 3D three.js", no_hover=true) }}

Le nœud jaune que nous apercevons n'est pas gris car je l'ai volontairement survolé avec la souris pour montrer l'effet de changement de couleur lors du survol d'un nœud.

<section class="task-nav">
    <a href="/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">Toutes les tâches</span>
        </div>
    </a>
    <a href="./exportation-godot">
        <div>
            <span class="label">Première tâche</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
