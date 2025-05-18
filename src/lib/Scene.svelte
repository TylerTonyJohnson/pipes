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
		LineCurve3,
		TubeGeometry,
		DoubleSide
	} from 'three';

	// interactivity();
	const PI = Math.PI;
	const HALF_PI = PI / 2;
	const directions = [
		{ x: 1, y: 0, z: 0 },
		{ x: 0, y: 1, z: 0 },
		{ x: 0, y: 0, z: 1 },
		{ x: -1, y: 0, z: 0 },
		{ x: 0, y: -1, z: 0 },
		{ x: 0, y: 0, z: -1 }
	];

	// --- CONFIG ---

	const PIPE_RADIUS = 1;
	const PIPE_SEGMENTS = 8;
	const BEND_SEGMENTS = 4;

	let BEND_RADIUS = $state(1);
	let MIN_LENGTH = $state(4);
	let MAX_LENGTH = $state(4);

	let MAX_ANGLE_DEVIATION = $state((PI * 3) / 4);

	let MAX_LENGTH_TOTAL = $state(1600);
	let STRAIGHT_BIAS = $state(1);
	// const BEND_ARC = PI / 2;
	// const BEND_RESOLUTION = 8;
	let POINTS_PER_LENGTH = $state(2);

	let SPHERE_CHANCE = $state(0);

	let growSpeed = $state(200); // units per second

	//  --- GENERATION ---

	let isOrtho = $state(true);

	// --- RUNTIME ---

	let showPipes = $state(true);
	let showBends = $state(false);
	let showDebug = $state(false);

	let isPlaying = $state(true);

	let elapsed = $state(1);
	let progress = $state(0.5);
	// $inspect('progress', progress);

	// --- DATA ---
	let pipes = $state();
	let spheres = $state();
	// $inspect('pipes', pipes);

	let curvePath = $derived(getCurvePath(pipes));
	// $inspect('path', curvePath);

	let curvePoints = $derived(getCurvePoints(curvePath));
	// $inspect('curve', curvePoints);

	let progressPoints = $derived(getProgressPoints(curvePoints, progress));
	// $inspect('progress', progressPoints);

	let progressCurve = $derived(getProgressCurve(curvePath, progress));
	// $inspect('progressCurve', progressCurve);

	let isReady = $derived(
		!!pipes && !!curvePath && !!curvePoints && !!progressPoints && !!progressCurve
	);
	// $inspect('ready', isReady);

	let color = $state(getRandomColor());

	// --- FUNCTIONS ---

	function generatePipes() {
		// Generate an array of points at right angles, random length
		const newPipes = [];
		const newSpheres = [];
		let remainingLength = MAX_LENGTH_TOTAL;
		let skipCount = 0;

		// First point is origin
		const firstPoint = { x: 0, y: 0, z: 0 };
		newPipes.push(firstPoint);

		let i = 1;

		while (remainingLength > 0) {
			// Skip if we have too many skips
			if (skipCount > 10) {
				console.log('skipping too many points, stopping');
				break;
			}

			// Choose a valid direction
			let direction, prevDirection;

			let isStraight;

			if (newPipes[i - 2]) {
				prevDirection = {
					x: Math.sign(newPipes[i - 2].x - newPipes[i - 1].x),
					y: Math.sign(newPipes[i - 2].y - newPipes[i - 1].y),
					z: Math.sign(newPipes[i - 2].z - newPipes[i - 1].z)
				};

				const filteredDirections = [
					...directions.filter(
						(d) => !(d.x === prevDirection.x && d.y === prevDirection.y && d.z === prevDirection.z)
					),
					...Array(Math.ceil(STRAIGHT_BIAS * STRAIGHT_BIAS * 10)).fill({
						x: -prevDirection.x,
						y: -prevDirection.y,
						z: -prevDirection.z
					})
				];

				direction = filteredDirections[Math.floor(Math.random() * filteredDirections.length)];
			} else {
				direction = directions[Math.floor(Math.random() * directions.length)];
			}

			// Check if the direction is straight
			if (
				prevDirection?.x === -direction.x &&
				prevDirection?.y === -direction.y &&
				prevDirection?.z === -direction.z
			) {
				isStraight = true;
			}

			// Get a valid length
			const length = Math.ceil(Math.random() * (MAX_LENGTH - MIN_LENGTH) + MIN_LENGTH);

			// Generate coordinates based on the direction and length
			const point = {
				x: newPipes[i - 1].x + direction.x * length,
				y: newPipes[i - 1].y + direction.y * length,
				z: newPipes[i - 1].z + direction.z * length
			};

			// Skip if this point was already used
			if (newPipes.some((p) => p.x === point.x && p.y === point.y && p.z === point.z)) {
				// console.log('skipping point', point);
				skipCount++;
				continue;
			}

			// Calculate length since last point
			const lastPoint = newPipes[i - 1];
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
				newPipes.push(point);

				// Check if we should add a sphere
				if (!isStraight && Math.random() < SPHERE_CHANCE) {
					newSpheres.push(point);
				}
				i++;
			}
		}

		progress = 0;
		color = getRandomColor();

		pipes = newPipes;
		spheres = newSpheres;
		// curvePath = getCurvePath(pipes);
		// [curvePoints, progressPoints] = getCurvePoints(curvePath);
	}

	function getCurvePath(pipes) {
		if (!pipes) return null;

		// Create new path
		const newCurvePath = new CurvePath();

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
				newCurvePath.add(bezierCurve);
			}

			if (i > 0) {
				// Add pipe
				const offsetPoints = getPipePoints(i);
				const lineCurve = new LineCurve3(...offsetPoints);
				// console.log(lineCurve);
				newCurvePath.add(lineCurve);
			}
		}

		return newCurvePath;
	}

	function getCurvePoints(curvePath) {
		if (!curvePath) return [];

		// const points = curvePath.getPoints(4);
		const length = curvePath.getLength();

		const totalPointCount = Math.floor(length * POINTS_PER_LENGTH);

		const points = curvePath.getSpacedPoints(totalPointCount);

		// Format total points for buffer geometry

		const pointsArray = new Float32Array(points.length * 3);

		points.forEach((point, index) => {
			pointsArray[index * 3] = point.x;
			pointsArray[index * 3 + 1] = point.y;
			pointsArray[index * 3 + 2] = point.z;
		});

		return pointsArray;
	}

	function getProgressPoints(curvePoints, progress) {
		if (!curvePoints) return [];

		const progressPoints = curvePoints.slice(
			0,
			Math.floor(Math.floor(curvePoints.length * progress) / 3) * 3
		);

		return progressPoints;
	}

	function getProgressCurve(curvePath, progress) {
		if (!curvePath) return null;

		const progressCurve = curvePath.clone();

		// Get the index of the cumulative length that is closest to the progress
		const totalLength = curvePath.getLength();
		const segmentLengths = curvePath.getLengths();
		// console.table('segmentLengths', segmentLengths);

		const targetLength = totalLength * progress;

		// console.log('targetLength', targetLength);

		progressCurve.curves = progressCurve.curves.filter((curve, index) => {
			const isAccepted = segmentLengths[index + 1] < targetLength;
			// console.table(index, segmentLengths[index], targetLength, isAccepted);

			return isAccepted;
		});

		// console.log('segmentLengths', segmentLengths);
		// console.log('progressCurve', progressCurve.curves);

		return progressCurve;
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

	function getRandomColor() {
		const letters = '0123456789ABCDEF';
		let color = '#';
		for (let i = 0; i < 6; i++) {
			color += letters[Math.floor(Math.random() * 16)];
		}
		return color;
	}

	function togglePlay() {
		isPlaying = !isPlaying;
	}

	useTask((delta) => {
		// Add elapsed time
		if (!isPlaying) return;

		elapsed += delta;

		// Increment progress at the progress rate

		const newProgress = progress + (delta * growSpeed) / curvePath.getLength();
		if (newProgress >= 1) {
			generatePipes();
			progress = 0;
		} else {
			progress = newProgress;
		}

		// progress = (progress + (delta * growSpeed) / MAX_LENGTH_TOTAL) % 1;

		// console.log(delta);
	});

	generatePipes();
</script>

<!-- CAMERA -->
<T.PerspectiveCamera
	makeDefault
	position={[20, 10, 20]}
	fov={90}
	oncreate={(ref) => {
		ref.lookAt(0, 1, 0);
	}}
>
	<!-- <OrbitControls>
		<Gizmo />
	</OrbitControls> -->
</T.PerspectiveCamera>

<!-- LIGHTING -->
<T.DirectionalLight position={[10, 10, 10]} intensity={10} />
<T.AmbientLight color="white" intensity={1.45} />

{#if isReady}
	<!-- PIPE -->
	{#if showPipes && progressCurve.curves.length > 0}
		<T.Mesh>
			<T.TubeGeometry
				args={[progressCurve, progressPoints.length, PIPE_RADIUS, PIPE_SEGMENTS, false]}
			/>
			<T.MeshStandardMaterial {color} side={DoubleSide} />
		</T.Mesh>

		{#each spheres as sphere}
			<T.Mesh position={(({ x, y, z }) => [x, y, z])(sphere)}>
				<T.SphereGeometry args={[1.75, 8, 8]} />
				<T.MeshStandardMaterial color="orange" />
			</T.Mesh>
		{/each}
	{/if}

	<!-- DEBUG VISUALS -->

	{#if showDebug} 
		<!-- Total Path -->
		<T.Points>
			<T.BufferGeometry>
				<T.BufferAttribute
					args={[curvePoints, 3]}
					attach={({ parent, ref }) => {
						parent.setAttribute('position', ref);
					}}
				></T.BufferAttribute>} />
			</T.BufferGeometry>
			<T.PointsMaterial size={0.25} color="white" />
		</T.Points>
		<!-- Progress Path -->
		<T.Points>
			<T.BufferGeometry>
				<T.BufferAttribute
					args={[progressPoints, 3]}
					attach={({ parent, ref }) => {
						parent.setAttribute('position', ref);
					}}
				></T.BufferAttribute>} />
			</T.BufferGeometry>
			<T.PointsMaterial size={0.25} color="red" />
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
{/if}

<!-- TWEAKS -->

<!-- <Pane position="fixed">
	<Folder title="Generate">
		<Checkbox bind:value={isOrtho} label="Orthogonal" />
		<Stepper bind:value={MAX_LENGTH_TOTAL} min={20} max={10000} step={20} label="Length" />
		<Slider bind:value={STRAIGHT_BIAS} min={0} max={1} label="Straight Bias" />
		<Button on:click={() => generatePipes()} label="Points" title="GENERATE" />
	</Folder>

	<Folder title="Config">
		<Checkbox bind:value={showPipes} label="Show Pipes" />
		<Slider bind:value={BEND_RADIUS} min={0} max={10} label="Radius" />
		<Slider bind:value={SPHERE_CHANCE} min={0} max={1} label="Orbs Chance" />
	</Folder>

	<Checkbox bind:value={showDebug} label="Debug" />
	<Folder title="Animation">
		<Slider bind:value={growSpeed} min={0} max={100} label="Speed" />
		<Slider bind:value={progress} min={0} max={1} label="Progress" />
		<Button on:click={togglePlay} title={isPlaying ? 'Pause' : 'Play'} />
	</Folder>
</Pane> -->
