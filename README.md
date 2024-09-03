
<html><head><base href="https://kubevis.app/">
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kubernetes Cluster Visualizer with Monitoring</title>
    <script src="https://d3js.org/d3.v7.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f0f5ff;
        }
        #container {
            max-width: 1600px;
            margin: 0 auto;
        }
        h1 {
            color: #326ce5;
            text-align: center;
        }
        textarea {
            width: 100%;
            height: 200px;
            margin-bottom: 20px;
            font-family: monospace;
        }
        #visualize {
            display: block;
            margin: 0 auto 20px;
            padding: 10px 20px;
            background-color: #326ce5;
            color: white;
            border: none;
            cursor: pointer;
        }
        #visualization {
            display: flex;
            justify-content: space-between;
        }
        #graph, #monitoring {
            width: 48%;
            background-color: white;
            border: 1px solid #ddd;
            border-radius: 5px;
            height: 600px;
        }
        .node circle {
            stroke: #fff;
            stroke-width: 2px;
        }
        .node text {
            font-size: 12px;
        }
        .link {
            fill: none;
            stroke: #ccc;
            stroke-width: 2px;
        }
        .chart-container {
            position: relative;
            height: 200px;
            width: 100%;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div id="container">
        <h1>Kubernetes Cluster Visualizer with Monitoring</h1>
        <textarea id="yamlInput" placeholder="Paste your Kubernetes YAML here...">
apiVersion: v1
kind: Node
metadata:
  name: node-1
---
apiVersion: v1
kind: Node
metadata:
  name: node-2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
        </textarea>
        <button id="visualize">Visualize</button>
        <div id="visualization">
            <div id="graph"></div>
            <div id="monitoring">
                <div class="chart-container">
                    <canvas id="cpuChart"></canvas>
                </div>
                <div class="chart-container">
                    <canvas id="memoryChart"></canvas>
                </div>
                <div class="chart-container">
                    <canvas id="networkChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <script>
        const width = 700;
        const height = 600;

        const svg = d3.select("#graph")
            .append("svg")
            .attr("width", width)
            .attr("height", height);

        const g = svg.append("g")
            .attr("transform", `translate(${width / 2},${height / 2})`);

        function visualizeCluster(data) {
            g.selectAll("*").remove();

            const root = d3.hierarchy(data);
            const links = root.links();
            const nodes = root.descendants();

            const simulation = d3.forceSimulation(nodes)
                .force("link", d3.forceLink(links).id(d => d.id).distance(100))
                .force("charge", d3.forceManyBody().strength(-1000))
                .force("x", d3.forceX())
                .force("y", d3.forceY());

            const link = g.selectAll(".link")
                .data(links)
                .enter().append("line")
                .attr("class", "link");

            const node = g.selectAll(".node")
                .data(nodes)
                .enter().append("g")
                .attr("class", "node")
                .call(d3.drag()
                    .on("start", dragstarted)
                    .on("drag", dragged)
                    .on("end", dragended));

            node.append("circle")
                .attr("r", d => getNodeSize(d))
                .style("fill", d => getNodeColor(d));

            node.append("text")
                .attr("dy", "0.31em")
                .attr("x", d => d.children ? -8 : 8)
                .attr("text-anchor", d => d.children ? "end" : "start")
                .text(d => d.data.name)
                .clone(true).lower()
                .attr("fill", "none")
                .attr("stroke", "white")
                .attr("stroke-width", 3);

            node.append("title")
                .text(d => getNodeTooltip(d));

            simulation.on("tick", () => {
                link
                    .attr("x1", d => d.source.x)
                    .attr("y1", d => d.source.y)
                    .attr("x2", d => d.target.x)
                    .attr("y2", d => d.target.y);

                node
                    .attr("transform", d => `translate(${d.x},${d.y})`);
            });

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
        }

        function getNodeSize(d) {
            switch (d.data.type) {
                case 'cluster': return 30;
                case 'node': return 20;
                case 'deployment': return 15;
                case 'pod': return 10;
                default: return 8;
            }
        }

        function getNodeColor(d) {
            switch (d.data.type) {
                case 'cluster': return '#326ce5';
                case 'node': return '#fd7e14';
                case 'deployment': return '#28a745';
                case 'pod': return '#17a2b8';
                default: return '#6c757d';
            }
        }

        function getNodeTooltip(d) {
            return `${d.data.type}: ${d.data.name}`;
        }

        function parseYAML(yamlString) {
            try {
                const documents = jsyaml.loadAll(yamlString);
                return buildClusterHierarchy(documents);
            } catch (e) {
                console.error("Error parsing YAML:", e);
                return { name: "Error parsing YAML" };
            }
        }

        function buildClusterHierarchy(documents) {
            const cluster = { name: 'Cluster', type: 'cluster', children: [] };
            const nodes = [];
            const deployments = [];

            documents.forEach(doc => {
                switch (doc.kind) {
                    case 'Node':
                        nodes.push({ name: doc.metadata.name, type: 'node', children: [] });
                        break;
                    case 'Deployment':
                        const deployment = { name: doc.metadata.name, type: 'deployment', children: [] };
                        for (let i = 0; i < doc.spec.replicas; i++) {
                            deployment.children.push({ name: `${doc.metadata.name}-pod-${i}`, type: 'pod' });
                        }
                        deployments.push(deployment);
                        break;
                }
            });

            cluster.children = [...nodes, ...deployments];
            return cluster;
        }

        // Monitoring charts
        const cpuChart = new Chart(document.getElementById('cpuChart').getContext('2d'), {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: 'CPU Usage',
                    data: [],
                    borderColor: 'rgb(75, 192, 192)',
                    tension: 0.1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        beginAtZero: true,
                        max: 100
                    }
                }
            }
        });

        const memoryChart = new Chart(document.getElementById('memoryChart').getContext('2d'), {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: 'Memory Usage',
                    data: [],
                    borderColor: 'rgb(255, 99, 132)',
                    tension: 0.1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        beginAtZero: true,
                        max: 100
                    }
                }
            }
        });

        const networkChart = new Chart(document.getElementById('networkChart').getContext('2d'), {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: 'Network I/O',
                    data: [],
                    borderColor: 'rgb(54, 162, 235)',
                    tension: 0.1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        beginAtZero: true
                    }
                }
            }
        });

        function updateCharts() {
            const now = new Date();
            const timeStr = now.toLocaleTimeString();

            cpuChart.data.labels.push(timeStr);
            cpuChart.data.datasets[0].data.push(Math.random() * 100);

            memoryChart.data.labels.push(timeStr);
            memoryChart.data.datasets[0].data.push(Math.random() * 100);

            networkChart.data.labels.push(timeStr);
            networkChart.data.datasets[0].data.push(Math.random() * 1000);

            if (cpuChart.data.labels.length > 10) {
                cpuChart.data.labels.shift();
                cpuChart.data.datasets[0].data.shift();
                memoryChart.data.labels.shift();
                memoryChart.data.datasets[0].data.shift();
                networkChart.data.labels.shift();
                networkChart.data.datasets[0].data.shift();
            }

            cpuChart.update();
            memoryChart.update();
            networkChart.update();
        }

        setInterval(updateCharts, 2000);

        document.getElementById("visualize").addEventListener("click", () => {
            const yamlInput = document.getElementById("yamlInput").value;
            const parsedCluster = parseYAML(yamlInput);
            visualizeCluster(parsedCluster);
        });

        // Initial visualization
        const initialYAML = document.getElementById("yamlInput").value;
        const initialParsedCluster = parseYAML(initialYAML);
        visualizeCluster(initialParsedCluster);
    </script>
</body></html>
