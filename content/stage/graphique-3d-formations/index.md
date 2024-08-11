+++
title = "Graphique 3D formations"
+++

## 2. Graphique D3.js pour afficher les futures formations

La deuxième tâche qui m'a été attribuée est la suivante : faire un graphique avec D3.js pour afficher les futures sessions de formations de l'entreprise.

Il m'a été imposé d'utiliser [tree of life](https://observablehq.com/@d3/tree-of-life) pour réaliser ce graphique.

J'ai commencé par comprendre les concepts de D3.js puis à comprendre le fonctionnement du tree of life.

### 1. Mise en place d'un serveur node.js

Pour afficher le graphique dans mon navigateur, j'ai mis en place un serveur avec node.js.

```js
import express from 'express';
const app = express()
const port = 3000

app.use(express.static('assets/html'));
app.use(express.static('assets/js'));
app.use(express.static('assets/json'));

app.get('/', (req, res) => {
  res.sendFile('index.html')
})

app.listen(port, () => {
  console.log(`Listening on port ${port}`)
})
```

L'objectif de ce serveur est juste de rendre accessible les ressources : html, js et autres fichiers externes utiles au graphique.

### 2. Création du graphique

Afin d'afficher le graphique, j'ai eu besoin d'un fichier html pour appeler le code javascript dans le navigateur.

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>d3js - Bashroom</title>

    <script type="module" src="index.js"></script>
</head>
<body>
</body>
</html>
```

Maintenant que tout était prêt pour afficher le graphique dans mon navigateur, il était temps de le réaliser.

Pour utiliser D3.js, j'ai utilisé le <abbr title="Content Delivery Network">CDN</abbr> pour importer D3 dans mon script.

```js
import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";
```

Ensuite, pour plus de simplicité, j'ai placé les données en brut dans le script. Voici un exemple de ce à quoi ressemblaient les données :

```js
let jsonData = [
    {
        "category": "front-end",
        "technos": [
            {
                "name": "angular",
                "difficulty": [
                    {
                        "name": "speed-run",
                        "duration": 1,
                        "dates": ["2024-01-08"]
                    },
                    {
                        "name": "basic",
                        "duration": 3,
                        "dates": ["2024-01-17", "2024-01-18", "2024-01-19"]
                    },
                    {
                        "name": "intermediate",
                        "duration": 4,
                        "dates": ["2024-02-05", "2024-02-06", "2024-02-07", "2024-02-08"]
                    },
                    {
                        "name": "advanced",
                        "duration": 5,
                        "dates": ["2024-02-26", "2024-02-27", "2024-02-28", "2024-02-29", "2024-02-30"]
                    }
                ]
            },
        ]
    }
];
```

C'est donc un tableau contenant des objets avec comme propriété `category` et `technos`, qui est un tableau contenant des objets pour définir la formation avec `name` et la difficulté défini par `difficulty` qui contient la difficulté, les dates et la durée des formations.

Ensuite, il a fallu créer une fonction pour transformer ces données au format <abbr title="JavaScript Object Notation">JSON</abbr> en un format compréhensible pour D3.js.

La fonction s'appelle `transformData` et prend `inputData` en paramètre, qui correspond à nos données sous format <abbr title="JavaScript Object Notation">JSON</abbr>.

```js
function transformData(inputData) {
```

Nous commençons par initialiser `result`, qui est un objet contenant le nom du nœud, un tableau contenant les nœuds enfants de ce nœud, et un état collapsed pour savoir si le nœud en question est replié ou non.

{% alert(tip=true) %}
Il faut savoir qu'un tree of life est constitué de noeud (node) et de liens (links).
{% end %}

```js
const result = { "name": "Formations", "children": [], "collapsed": true };
```

Maintenant que notre format d'objet est défini avec notre premier nœud central s'appelant "Formations".

Nous pouvons itérer avec des boucles sur chacun des tableaux que nous avons dans nos données, donc `inputData`, `technos`, `difficulty` et `dates`. Cela nous permet d'ajouter en tant que noeud enfant chaque valeur à son noeud parent.

```js
    for (const categoryData of inputData) {
        const categoryNode = { "name": categoryData.category, "children": [], "collapsed": true };

        for (const techno of categoryData.technos) {
            const technoNode = { "name": techno.name, "children": [], "collapsed": true };

            for (const difficulty of techno.difficulty) {
                const difficultyNode = { "name": difficulty.name, "value": difficulty.duration, "children": [], "collapsed": true };

                for (const date of difficulty.dates) {
                    const dateNode = { "name": date };

                    difficultyNode.children.push(dateNode);
                }
                technoNode.children.push(difficultyNode);
            }

            categoryNode.children.push(technoNode);
        }

        result.children.push(categoryNode);
    }

    return result;
}
```

Nous initialisons ensuite deux variables pour stocker les données transformées. Une première constante qui contient le résultat de notre fonction `transformData` afin d'avoir accès aux données transformées intactes. Puis une deuxième variable qui contient notre première valeur, mais qui sera modifiable pour agir sur les nœuds.

```js
const globalData = transformData(jsonData);
let treeData = globalData;
```

La fonction `updateTree`, comme son nom l'indique, permet de mettre à jour l'arbre du graphique. Cette fonction génère, à partir des données de la variable treeData, les nœuds, les liens et les textes pour afficher le nom des nœuds.

```js
function updateTree() {
    d3.select("body").select("svg").remove();

    // Constant for tree length
    const width = 928;
    const height = width;
    const cx = width * 0.5;
    const cy = height * 0.59;
    const radius = Math.min(width, height) / 2 - 120;

    // Tree instance with custom options and our data
    const tree = d3.tree()
        .size([2 * Math.PI, radius])
        .separation((a, b) => (a.parent == b.parent ? 1 : 2) / a.depth);

    const root = tree(d3.hierarchy(treeData)
        .sort((a, b) => d3.ascending(a.data.name, b.data.name)));

    const svg = d3.select("body")
        .append("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [-cx, -cy, width, height])
        .attr("style", "width: 75%; height: auto; font: 10px sans-serif;");

    // Append links between each node
    svg.append("g")
        .attr("fill", "none")
        .attr("stroke", "#156")
        .attr("stroke-opacity", 0.4)
        .attr("stroke-width", 1.5)
        .selectAll()
        .data(root.links())
        .join("path")
        .attr("d", d3.linkRadial()
            .angle(d => d.x)
            .radius(d => d.y));

    // Append each node with mouse handlers
    svg.append("g")
        .selectAll()
        .data(root.descendants())
        .join("circle")
        .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0)`)
        .attr("fill", d => d.children ? "#555" : "#999")
        .attr("r", 2.5)
        .on("click", (e, d) => {
            d.data.collapsed ? expand(d.data) : collapse(d.data);
            updateTree();
        });

    // Append text for formation's name with mouse handlers
    svg.append("g")
        .attr("stroke-linejoin", "round")
        .attr("stroke-width", 3)
        .selectAll()
        .data(root.descendants())
        .join("text")
        .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0) rotate(${d.x >= Math.PI ? 180 : 0})`)
        .attr("dy", "0.31em")
        .attr("x", d => d.x < Math.PI === !d.children ? 6 : -6)
        .attr("text-anchor", d => d.x < Math.PI === !d.children ? "start" : "end")
        .attr("paint-order", "stroke")
        .attr("stroke", "white")
        .attr("fill", "currentColor")
        .text(d => d.data.name)
        .on("click", (e, d) => {
            d.data.collapsed ? expand(d.data) : collapse(d.data);
            updateTree();
        });
}
```

Après la fonction pour mettre à jour l'arbre, nous avons les deux fonctions permettant de développer et de réduire les nœuds, appelées respectivement `expand` et `collapse` en anglais. Elles permettent de mettre à jour l'arbre en modifiant les nœuds développés et réduits.

```js
function expand(d) {
    if (d._children) {
        d.children = d._children;
        d._children = null;
    } else if (d.children) {
        d._children = d.children;
        d.children = null;
    }
    if (d._children) d._children.forEach(expand);
}

function collapse(d) {
    if (d._children) {
        d.children = d._children;
        d._children = null;
    }
    if (d.children) d.children.forEach(collapse);
}
```

À la toute fin du script, nous trouvons une ligne pour appeler la fonction `updateTree` et afficher le graphique pour la première fois.

```js
updateTree()
```

<details>
    <summary>Voir le script en entier</summary>

    import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";

    let jsonData = [
        {
            "category": "front-end",
            "technos": [
                {
                    "name": "angular",
                    "difficulty": [
                        {
                            "name": "speed-run",
                            "duration": 1,
                            "dates": ["2024-01-08"]
                        },
                        {
                            "name": "basic",
                            "duration": 3,
                            "dates": ["2024-01-17", "2024-01-18", "2024-01-19"]
                        },
                        {
                            "name": "intermediate",
                            "duration": 4,
                            "dates": ["2024-02-05", "2024-02-06", "2024-02-07", "2024-02-08"]
                        },
                        {
                            "name": "advanced",
                            "duration": 5,
                            "dates": ["2024-02-26", "2024-02-27", "2024-02-28", "2024-02-29", "2024-02-30"]
                        }
                    ]
                },
            ]
        }
    ];

    /**
     * Transform JSON data into an object for the tree
     * @param {JSON} inputData
     * @returns An object for the tree
     */
    function transformData(inputData) {
        const result = { "name": "Formations", "children": [], "collapsed": true };

        for (const categoryData of inputData) {
            const categoryNode = { "name": categoryData.category, "children": [], "collapsed": true };

            for (const techno of categoryData.technos) {
                const technoNode = { "name": techno.name, "children": [], "collapsed": true };

                for (const difficulty of techno.difficulty) {
                    const difficultyNode = { "name": difficulty.name, "value": difficulty.duration, "children": [], "collapsed": true };

                    for (const date of difficulty.dates) {
                        const dateNode = { "name": date };

                        difficultyNode.children.push(dateNode);
                    }
                    technoNode.children.push(difficultyNode);
                }

                categoryNode.children.push(technoNode);
            }

            result.children.push(categoryNode);
        }

        return result;
    }

    const globalData = transformData(jsonData);
    let treeData = globalData;

    function updateTree() {
        d3.select("body").select("svg").remove();

        // Constant for tree length
        const width = 928;
        const height = width;
        const cx = width * 0.5;
        const cy = height * 0.59;
        const radius = Math.min(width, height) / 2 - 120;

        // Tree instance with custom options and our data
        const tree = d3.tree()
            .size([2 * Math.PI, radius])
            .separation((a, b) => (a.parent == b.parent ? 1 : 2) / a.depth);

        const root = tree(d3.hierarchy(treeData)
            .sort((a, b) => d3.ascending(a.data.name, b.data.name)));

        const svg = d3.select("body")
            .append("svg")
            .attr("width", width)
            .attr("height", height)
            .attr("viewBox", [-cx, -cy, width, height])
            .attr("style", "width: 75%; height: auto; font: 10px sans-serif;");

        // Append links between each node
        svg.append("g")
            .attr("fill", "none")
            .attr("stroke", "#156")
            .attr("stroke-opacity", 0.4)
            .attr("stroke-width", 1.5)
            .selectAll()
            .data(root.links())
            .join("path")
            .attr("d", d3.linkRadial()
                .angle(d => d.x)
                .radius(d => d.y));

        // Append each node with mouse handlers
        svg.append("g")
            .selectAll()
            .data(root.descendants())
            .join("circle")
            .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0)`)
            .attr("fill", d => d.children ? "#555" : "#999")
            .attr("r", 2.5)
            .on("click", (e, d) => {
                d.data.collapsed ? expand(d.data) : collapse(d.data);
                updateTree();
            });

        // Append text for formation's name with mouse handlers
        svg.append("g")
            .attr("stroke-linejoin", "round")
            .attr("stroke-width", 3)
            .selectAll()
            .data(root.descendants())
            .join("text")
            .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0) rotate(${d.x >= Math.PI ? 180 : 0})`)
            .attr("dy", "0.31em")
            .attr("x", d => d.x < Math.PI === !d.children ? 6 : -6)
            .attr("text-anchor", d => d.x < Math.PI === !d.children ? "start" : "end")
            .attr("paint-order", "stroke")
            .attr("stroke", "white")
            .attr("fill", "currentColor")
            .text(d => d.data.name)
            .on("click", (e, d) => {
                d.data.collapsed ? expand(d.data) : collapse(d.data);
                updateTree();
            });
    }

    function expand(d) {
        if (d._children) {
            d.children = d._children;
            d._children = null;
        } else if (d.children) {
            d._children = d.children;
            d.children = null;
        }
        if (d._children) d._children.forEach(expand);
    }

    function collapse(d) {
        if (d._children) {
            d.children = d._children;
            d._children = null;
        }
        if (d.children) d.children.forEach(collapse);
    }

    updateTree()

</details>

Voici le résultat de ce script dans le navigateur :
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/d3-formations-result.png?raw=true", alt="Resultat script graphique D3.js", no_hover=true) }}

Nous pouvons voir que certains nœuds sont repliés, comme : nestjs, rust, typescript, ansible, aws et cloud. Nous pouvons aussi voir que des dates sont disponibles uniquement pour les formations en bas à droite du graphique.

<section class="task-nav">
    <a href="/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">Toutes les tâches</span>
        </div>
    </a>
    <a href="/stage/graphiques-formations-data-science">
        <div>
            <span class="label">Troisième tâche</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
