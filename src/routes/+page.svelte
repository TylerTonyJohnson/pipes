<script>
	import { Canvas } from '@threlte/core';
	import Scene from '$lib/Scene.svelte';
</script>

<div class="frame">
	<Canvas>
		<Scene />
	</Canvas>
</div>

<style>
	:global(html, body) {
		box-sizing: border-box;
		width: 100%;
		height: 100%;
		margin: 0;
		background-color: #000;
	}

	.frame {
		width: 100%;
		height: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
	}
</style>
