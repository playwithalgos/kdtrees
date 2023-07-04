<script>
	import { draw } from "svelte/transition";
	let points = [];
	for (let i = 0; i < 100; i++) points.push(0);
	points = points.map(() => ({
		x: 10 + 1000 * Math.random(),
		y: 10 + 1000 * Math.random(),
	}));
	console.log(points);

	let edges = [];

	function addEdge(m, m1, coord, depth) {
		edges.push({ begin: m, end: m1, coord, depth });
	}

	let rects = [];
	let maxdepth = -2;

	function createKDTree(points, coord, depth, r) {
		function median(points, coord) {
			return points.sort((a, b) => a[coord] - b[coord])[
				Math.floor(points.length / 2)
			];
		}

		function nextCoord(coord) {
			return coord == "x" ? "y" : "x";
		}
		const m = median(points, coord);

		rects.push({ ...r, depth: depth });

		if (depth > maxdepth) return m;

		const p1 = points.filter((p) => p[coord] <= m[coord] && p != m);
		const p2 = points.filter((p) => p[coord] > m[coord]);

		if (p1.length > 0) {
			//{ x1: r.x1, y1: r.y1, x2: r.x2, y2: r.y2 }
			const r1 =
				coord == "x"
					? { x1: r.x1, y1: r.y1, x2: m.x, y2: r.y2 }
					: { x1: r.x1, y1: r.y1, x2: r.x2, y2: m.y };
			const m1 = createKDTree(p1, nextCoord(coord), depth + 1, r1);
			addEdge(m, m1, coord, depth);
		}

		if (p2.length > 0) {
			const r2 =
				coord == "x"
					? { x1: m.x, y1: r.y1, x2: r.x2, y2: r.y2 }
					: { x1: r.x1, y1: m.y, x2: r.x2, y2: r.y2 };
			const m2 = createKDTree(p2, nextCoord(coord), depth + 1, r2);
			addEdge(m, m2, coord, depth, r2);
		}
		return m;
	}

	function loop() {
		createKDTree(points, "x", 0, { x1: 0, y1: 0, x2: 1000, y2: 1000 });
		edges = edges;
		rects = rects;
		maxdepth++;
		if (maxdepth < Math.log2(points.length)) setTimeout(loop, 2000);
	}
	loop();


</script>

<svg width={1064} height={1064} viewBox="-32 -32 {1064} {1064}">
	{#each rects as r}
		<rect
			transition:draw={{ duration: 1000 }}
			x={"" + r.x1}
			y={"" + r.y1}
			width={"" + (r.x2 - r.x1)}
			height={"" + (r.y2 - r.y1)}
			stroke="lightgray"
			stroke-width={2 * (Math.log2(points.length) - 1 - r.depth)}
			fill="none"
		/>
	{/each}

	{#each edges as e}
		<line
			transition:draw={{ duration: 1000 }}
			x1={"" + e.begin.x}
			y1={"" + e.begin.y}
			x2={"" + e.end.x}
			y2={"" + e.end.y}
			stroke={e.coord == "x" ? "red" : "blue"}
			stroke-width={Math.log2(points.length) - 1 - e.depth}
		/>
	{/each}

	{#each points as p}
		<circle
			transition:draw={{ duration: 500 }}
			cx={"" + p.x}
			cy={"" + p.y}
			r={6}
			fill="black"
		/>
	{/each}
</svg>

<style>
</style>
