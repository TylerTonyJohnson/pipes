<script>
	import { T, useTask } from '@threlte/core';
	import { OrbitControls, Gizmo } from '@threlte/extras';
	import { Button, Pane, Slider, Checkbox, Folder, Stepper } from 'svelte-tweakpane-ui';
	// import { Spring } from 'svelte/motion';
	import {
		AmbientLight,
		Vector3,
		SplineCurve,
		CatmullRomCurve3,
		QuadraticBezierCurve3,
		CurvePath,
		LineCurve3
	} from 'three';

	// interactivity();
	const PI = Math.PI;
	const HALF_PI = PI / 2;

	// --- CONFIG ---

	const PIPE_RADIUS = 1;
	const PIPE_SEGMENTS = 8;
	const BEND_SEGMENTS = 4;

	let BEND_RADIUS = $state(1);
	let MIN_LENGTH = $state(4);
	let MAX_LENGTH = $state(4);

	let MAX_ANGLE_DEVIATION = $state((PI * 3) / 4);

	let MAX_LENGTH_TOTAL = $state(100);
	let STRAIGHT_BIAS = $state(0.5);
	// const BEND_ARC = PI / 2;
	// const BEND_RESOLUTION = 8;
	let POINTS_PER_LENGTH = $state(2);

	let growSpeed = $state(40); // units per second

	//  --- GENERATION ---

	let isOrtho = $state(true);

	// --- RUNTIME ---

	let showPipes = $state(false);
	let showBends = $state(false);
	let showDebug = $state(true);

	let pipes = $state();

	const directions = [
		{ x: 1, y: 0, z: 0 },
		{ x: 0, y: 1, z: 0 },
		{ x: 0, y: 0, z: 1 },
		{ x: -1, y: 0, z: 0 },
		{ x: 0, y: -1, z: 0 },
		{ x: 0, y: 0, z: -1 }
	];

	let isPlaying = $state(false);

	let elapsed = $state(1);
	let progress = $state(1);
	// $inspect(progress);

	let curvePoints = $derived(getCurvePoints());
	// $inspect(curvePoints);

	// --- FUNCTIONS ---

	function generatePipes() {
		// Generate an array of points at right angles, random length
		const points = [];
		let remainingLength = MAX_LENGTH_TOTAL;
		let skipCount = 0;

		// First point is origin
		const firstPoint = { x: 0, y: 0, z: 0 };
		points.push(firstPoint);

		let i = 1;

		while (remainingLength > 0) {
			// Skip if we have too many skips
			if (skipCount > 10) {
				console.log('skipping too many points, stopping');
				break;
			}

			// Choose a valid direction
			let direction;

			if (points[i - 2]) {
				const prevDirection = {
					x: Math.sign(points[i - 2].x - points[i - 1].x),
					y: Math.sign(points[i - 2].y - points[i - 1].y),
					z: Math.sign(points[i - 2].z - points[i - 1].z)
				};

				const filteredDirections = [...directions.filter(
					(d) => !(d.x === prevDirection.x && d.y === prevDirection.y && d.z === prevDirection.z)
				), ...Array(Math.ceil(STRAIGHT_BIAS * STRAIGHT_BIAS * 10)).fill({ x: -prevDirection.x, y: -prevDirection.y, z: -prevDirection.z })];

				direction = filteredDirections[Math.floor(Math.random() * filteredDirections.length)];
			} else {
				direction = directions[Math.floor(Math.random() * directions.length)];
			}

			// Get a valid length
			const length = Math.ceil(Math.random() * (MAX_LENGTH - MIN_LENGTH) + MIN_LENGTH);

			// Generate coordinates based on the direction and length
			const point = {
				x: points[i - 1].x + direction.x * length,
				y: points[i - 1].y + direction.y * length,
				z: points[i - 1].z + direction.z * length
			};

			// Skip if this point was already used
			if (points.some((p) => p.x === point.x && p.y === point.y && p.z === point.z)) {
				// console.log('skipping point', point);
				skipCount++;
				continue;
			}

			// Calculate length since last point
			const lastPoint = points[i - 1];
			const lastLength = Math.hypot(
				point.x - lastPoint.x,
				point.y - lastPoint.y,
				point.z - lastPoint.z
			);

			// Check if the length is too long
			if (lastLength > remainingLength) {
				// console.log('end of length');
				break;
			} else {
				remainingLength -= lastLength;
				// console.log('adding point', point);
				points.push(point);
				i++;
			}
		}

		pipes = points;
	}

	function getCurvePoints() {
		// Create new path
		const curvePath = new CurvePath();

		// Create curves
		for (let i = 0; i < pipes.length; i++) {
			if (i > 1) {
				const start = pipes[i - 2];
				const center = pipes[i - 1];
				const end = pipes[i];

				// Get the offset points
				const [, bendStart] = getOffsetPoints(center, start);
				const bendCenter = center;
				const [, bendEnd] = getOffsetPoints(center, end);

				// Convert to Vector3
				const vertices = [
					new Vector3(bendStart.x, bendStart.y, bendStart.z),
					new Vector3(bendCenter.x, bendCenter.y, bendCenter.z),
					new Vector3(bendEnd.x, bendEnd.y, bendEnd.z)
				];

				// Get spline points
				const bezierCurve = new QuadraticBezierCurve3(...vertices);
				curvePath.add(bezierCurve);
			}

			if (i > 0) {
				// Add pipe
				const offsetPoints = getPipePoints(i);
				const lineCurve = new LineCurve3(...offsetPoints);
				// console.log(lineCurve);
				curvePath.add(lineCurve);
			}
		}

		// const points = curvePath.getPoints(4);
		const length = curvePath.getLength();

		const totalPointCount = Math.floor(length * POINTS_PER_LENGTH);

		const points = curvePath.getSpacedPoints(totalPointCount);

		const progressPoints = points.slice(0, progress * totalPointCount);

		const floatArray = new Float32Array(progressPoints.length * 3);

		progressPoints.forEach((point, index) => {
			floatArray[index * 3] = point.x;
			floatArray[index * 3 + 1] = point.y;
			floatArray[index * 3 + 2] = point.z;
		});

		// Convert to array of coordinates

		return floatArray;
	}

	function getPipePoints(index) {
		// the segment runs from pipes[index-1] → pipes[index]
		const start = pipes[index - 1];
		const end = pipes[index];

		// figure out whether this segment actually has a “bend” at each end
		const hasPrev = pipes[index - 2] !== undefined; // is there a pipe before this one?
		const hasNext = pipes[index + 1] !== undefined; // is there a pipe after this one?

		// compute the offset points for a bend (even if we may not use them)
		const [offsetEnd, offsetStart] = getOffsetPoints(start, end);

		// only use the offset if there really is something to bend against
		const startPoint = hasPrev ? offsetStart : start;
		const endPoint = hasNext ? offsetEnd : end;

		return [
			new Vector3(startPoint.x, startPoint.y, startPoint.z),
			new Vector3(endPoint.x, endPoint.y, endPoint.z)
		];
	}

	function getOffsetPoints(start, end) {
		// Get length of line
		const length = Math.hypot(end.x - start.x, end.y - start.y, end.z - start.z);

		const insetStart = {
			x: end.x - ((end.x - start.x) / length) * BEND_RADIUS,
			y: end.y - ((end.y - start.y) / length) * BEND_RADIUS,
			z: end.z - ((end.z - start.z) / length) * BEND_RADIUS
		};

		const insetEnd = {
			x: start.x - ((start.x - end.x) / length) * BEND_RADIUS,
			y: start.y - ((start.y - end.y) / length) * BEND_RADIUS,
			z: start.z - ((start.z - end.z) / length) * BEND_RADIUS
		};

		return [insetStart, insetEnd];
	}

	function togglePlay() {
		isPlaying = !isPlaying;
	}

	useTask((delta) => {
		// Add elapsed time
		if (!isPlaying) return;
		elapsed += delta;

		// Increment progress at the progress rate

		progress = (progress + (delta * growSpeed) / MAX_LENGTH_TOTAL) % 1;

		// console.log(delta);
	});

	generatePipes();
</script>

<!-- CAMERA -->
<T.PerspectiveCamera
	makeDefault
	position={[20, 20, 20]}
	oncreate={(ref) => {
		ref.lookAt(0, 1, 0);
	}}
>
	<OrbitControls>
		<Gizmo />
	</OrbitControls>
</T.PerspectiveCamera>

<!-- LIGHTING -->
<T.DirectionalLight position={[10, 10, 10]} intensity={10} />
<T.AmbientLight color="white" intensity={1.45} />

<!-- DEBUG VISUALS -->

{#if showDebug}
	<!-- Dashed Line -->
	<T.Points>
		<T.BufferGeometry>
			<T.BufferAttribute
				args={[getCurvePoints(), 3]}
				attach={({ parent, ref }) => {
					parent.setAttribute('position', ref);
				}}
			></T.BufferAttribute>} />
		</T.BufferGeometry>
		<T.PointsMaterial size={0.25} />
	</T.Points>

	<!-- ORBS -->
	<T.Mesh position={(({ x, y, z }) => [x, y, z])(pipes[0])}>
		<T.SphereGeometry />
		<T.MeshStandardMaterial color="orange" />
	</T.Mesh>
	<T.Mesh position={(({ x, y, z }) => [x, y, z])(pipes.at(-1))}>
		<T.SphereGeometry />
		<T.MeshStandardMaterial color="teal" />
	</T.Mesh>
{/if}

<!-- TWEAKS -->

<Pane position="fixed">
	<Folder title="Generate">
		<Checkbox bind:value={isOrtho} label="Orthogonal" />
		<Stepper bind:value={MAX_LENGTH_TOTAL} min={20} max={10000} step={20} label="Length" />
		<Slider bind:value={STRAIGHT_BIAS} min={0} max={1} label="Straight Bias" />
		<Button on:click={() => generatePipes()} label="Points" title="GENERATE" />
	</Folder>

	<Slider bind:value={BEND_RADIUS} min={0} max={10} label="Radius" />

	<Checkbox bind:value={showDebug} label="Debug" />
	<Folder title="Animation">
		<Slider bind:value={growSpeed} min={0} max={100} label="Speed" />
		<Slider bind:value={progress} min={0} max={1} label="Progress" />
		<Button on:click={togglePlay} title={isPlaying ? 'Pause' : 'Play'} />
	</Folder>
</Pane>
