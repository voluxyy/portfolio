+++
title = "3D Training Graph"
+++

## 2. D3.js Graph to Display Upcoming Trainings

The second task assigned to me was to create a graph with D3.js to display the company's upcoming training sessions.

I was required to use [tree of life](https://observablehq.com/@d3/tree-of-life) to create this graph.

I started by understanding the concepts of D3.js and then understanding how the tree of life works.

### 1. Setting up a Node.js Server

To display the graph in my browser, I set up a server with Node.js.

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

The purpose of this server is simply to make resources accessible: HTML, JS, and other external files useful for the graph.

### 2. Creating the Graph

To display the graph, I needed an HTML file to call the JavaScript code in the browser.

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

Now that everything was ready to display the graph in my browser, it was time to create it.

To use D3.js, I used the <abbr title="Content Delivery Network">CDN</abbr> to import D3 into my script.

```js
import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";
```

Next, for simplicity, I placed the raw data in the script. Here is an example of what the data looked like:

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

It’s a table containing objects with properties like `category` and `technos`, which is an array containing objects that define the training with `name` and the difficulty defined by `difficulty`, which contains the difficulty, dates, and duration of the training sessions.

Next, a function was needed to transform this data from <abbr title="JavaScript Object Notation">JSON</abbr> format into a format understandable by D3.js.

The function is called `transformData` and takes `inputData` as a parameter, which corresponds to our data in <abbr title="JavaScript Object Notation">JSON</abbr> format.

```js
function transformData(inputData) {
```

We start by initializing `result`, which is an object containing the node name, an array containing the child nodes of this node, and a collapsed state to determine if the node is collapsed or not.

{% alert(tip=true) %}
It’s important to know that a tree of life is made up of nodes and links.
{% end %}

```js
const result = { "name": "Formations", "children": [], "collapsed": true };
```

Now that our object format is defined with our first central node named "Formations".

We can iterate through each of the arrays we have in our data, such as `inputData`, `technos`, `difficulty`, and `dates`. This allows us to add each value as a child node to its parent node.

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

We then initialize two variables to store the transformed data. The first constant contains the result of our `transformData` function to have access to the intact transformed data. The second variable contains our first value but will be modifiable to act on the nodes.

```js
const globalData = transformData(jsonData);
let treeData = globalData;
```

The `updateTree` function, as its name suggests, updates the tree of the graph. This function uses the data from the `treeData` variable to generate the nodes, links, and text to display the node names.

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

After the function to update the tree, we have two functions that enable the functionality to expand and collapse nodes. They allow the tree to be updated by modifying the expanded and collapsed nodes.

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

At the very end of the script, we find a line to call the `updateTree` function to display the graph for the first time.

```js
updateTree()
```

<details>
    <summary>View full script</summary>

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

Here is the result of this script in the browser:
{{ image(url="https://github.com/voluxyy/portfolio/blob/main/static/stage/d3-formations-result.png?raw=true", alt="D3.js Graph Script Result", no_hover=true) }}

We can see that some nodes are collapsed, such as: nestjs, rust, typescript, ansible, aws, and cloud. We can also see that dates are available only for the training sessions in the bottom right of the graph.

<section class="task-nav">
    <a href="/portfolio/en/stage">
        <div>
            <span class="previous">← </span>
            <span class="label">All tasks</span>
        </div>
    </a>
    <a href="/portfolio/en/stage/graphiques-formations-data-science">
        <div>
            <span class="label">Third task</span>
            <span class="next">→</span>
        </div>
    </a>
</section>
