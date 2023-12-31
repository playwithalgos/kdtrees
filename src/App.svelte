<script>
	import { draw } from "svelte/transition";
	let points = [];
	const DELAY = 1000;
	const WIDTH = 1920;
	const HEIGHT = 1080;
	const NBPOINTS = 120;
	const NIGHTMODE = true;

	function alignedWithExistingPoints(pt) {
		function aligned(A, B, C) {
			const AB = { x: B.x - A.x, y: B.y - A.y };
			const AC = { x: C.x - A.x, y: C.y - A.y };

			return Math.abs(AB.x * AC.y - AC.x * AB.y) < 0.2;
		}

		for (const A of points)
			for (const B of points)
				if (A != B) {
					if (aligned(A, B, pt)) return true;
				}

		return false;
	}

	function closeFromExistingPoints(pt) {
		for (const A of points)
			if ((A.x - pt.x) ** 2 + (A.y - pt.y) ** 2 < 500) return true;

		return false;
	}

	function arrondi(x) {
		const grain = 8;
		return Math.round(x / grain) * grain;
	}

	function generateX() {
		const x = arrondi(10 + (WIDTH - 30) * Math.random());
		for (const A of points) if (A.x == x) return generateX();
		return x;
	}

	function generateY() {
		const y = arrondi(10 + (HEIGHT - 30) * Math.random());
		for (const A of points) if (A.y == y) return generateY();
		return y;
	}

	for (let i = 0; i < NBPOINTS; i++) {
		const pt = {
			x: generateX(),
			y: generateY(),
		};

		//	if (!alignedWithExistingPoints(pt) && !closeFromExistingPoints(pt))
		points.push(pt);
	}

	let edges = [];

	function addEdge(m, m1, coord, depth) {
		edges.push({ begin: m, end: m1, coord, depth });
	}

	let verticalLines = [];
	let horizontalLines = [];
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
		m.depth = depth;

		if (coord == "x")
			verticalLines.push({ x1: m.x, x2: m.x, y1: r.y1, y2: r.y2, depth });
		else
			horizontalLines.push({
				y1: m.y,
				y2: m.y,
				x1: r.x1,
				x2: r.x2,
				depth,
			});

		if (depth > maxdepth) return m;

		const p1 = points.filter((p) => p[coord] <= m[coord] && p != m);
		const p2 = points.filter((p) => p[coord] > m[coord]);

		if (p1.length > 0 || p2.length > 0) m.split = coord;
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
		createKDTree(points, "x", 0, { x1: 0, y1: 0, x2: WIDTH, y2: HEIGHT });
		points = points;
		edges = edges;
		verticalLines = verticalLines;
		horizontalLines = horizontalLines;
		maxdepth++;
		if (maxdepth < Math.log2(points.length)) setTimeout(loop, DELAY);
	}

	setTimeout(loop, DELAY);
</script>

<svg width={WIDTH} height={HEIGHT} viewBox="-32 -32 {WIDTH + 32} {HEIGHT + 32}">
	<defs>
		<marker
			id="arrowx"
			viewBox="0 0 10 10"
			refX="16"
			refY="5"
			markerWidth="6"
			markerHeight="6"
			orient="auto-start-reverse"
		>
			<path d="M 0 0 L 10 5 L 0 10" stroke="orange" fill="none" />
		</marker>
		<marker
			id="arrowy"
			viewBox="0 0 10 10"
			refX="16"
			refY="5"
			markerWidth="6"
			markerHeight="6"
			orient="auto-start-reverse"
		>
			<path d="M 0 0 L 10 5 L 0 10" stroke="skyblue" fill="none" />
		</marker>
	</defs>

	{#each horizontalLines as line}
		<line
			transition:draw={{ duration: 1000 }}
			x1={"" + line.x1}
			y1={"" + line.y1}
			x2={"" + line.x2}
			y2={"" + line.y2}
			stroke="skyblue"
			stroke-width={2 * (Math.log2(points.length) - 1 - line.depth)}
			fill="none"
		/>
	{/each}

	{#each verticalLines as line}
		<line
			transition:draw={{ duration: 1000 }}
			x1={"" + line.x1}
			y1={"" + line.y1}
			x2={"" + line.x2}
			y2={"" + line.y2}
			stroke="orange"
			stroke-width={2 * (Math.log2(points.length) - 1 - line.depth)}
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
			stroke={e.coord == "x" ? "orange" : "skyblue"}
			stroke-width={2 + 0.5 * (Math.log2(points.length) - 1 - e.depth)}
			marker-end={`url(#arrow${e.coord})`}
		/>
	{/each}

	{#each points as p}
		<circle
			transition:draw={{ duration: 500 }}
			cx={"" + p.x}
			cy={"" + p.y}
			r={p.depth == undefined
				? 10
				: 10 + 2 * (Math.log2(points.length) - 1 - p.depth)}
			stroke={NIGHTMODE ? "white" : "black"}
			stroke-width=2
			fill={p.split == "x"
				? "orange"
				: p.split == "y"
				? "skyblue"
				: NIGHTMODE
				? "white"
				: "black"}
		/>
	{/each}
</svg>

<style>
	:global(body) {
		background-color: black;
		text-align: center;
		vertical-align: middle;
	}
</style>
