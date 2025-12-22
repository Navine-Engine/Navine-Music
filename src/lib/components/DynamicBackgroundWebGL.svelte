<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { playerStore } from '$lib/stores/player';
	import { effectivePerformanceLevel } from '$lib/stores/performance';
	import { losslessAPI } from '$lib/api';
	import {
		extractPaletteFromImage,
		getMostVibrantColor,
		type Color
	} from '$lib/utils/colorExtraction';
	import {
		vertexShaderSource,
		updateStateShaderSource,
		colorRenderShaderSource,
		blurFragmentShaderSource,
		createShader,
		createProgram,
		setupQuad,
		createTexture,
		createFramebuffer
	} from '$lib/utils/webglShaders';

	const DISPLAY_CANVAS_SIZE = 512;
	const MASTER_PALETTE_SIZE = 40;
	const DISPLAY_GRID_WIDTH = 8;
	const DISPLAY_GRID_HEIGHT = 5;
	const STRETCHED_GRID_WIDTH = 32;
	const STRETCHED_GRID_HEIGHT = 18;
	const BLUR_DOWNSAMPLE_FACTOR = 26;
	const SONG_PALETTE_TRANSITION_SPEED = 0.015;
	const SCROLL_SPEED = 0.008;

	let canvasElement: HTMLCanvasElement;
	let gl: WebGLRenderingContext | null = null;

	let updateStateProgram: WebGLProgram | null = null;
	let colorRenderProgram: WebGLProgram | null = null;
	let blurProgram: WebGLProgram | null = null;

	let paletteTexture: WebGLTexture | null = null;
	let cellStateTexture: WebGLTexture | null = null;
	let cellStateTexture2: WebGLTexture | null = null;
	let colorRenderTexture: WebGLTexture | null = null;
	let blurTexture1: WebGLTexture | null = null;
	let blurTexture2: WebGLTexture | null = null;

	let stateFramebuffer1: WebGLFramebuffer | null = null;
	let stateFramebuffer2: WebGLFramebuffer | null = null;
	let colorRenderFramebuffer: WebGLFramebuffer | null = null;
	let blurFramebuffer1: WebGLFramebuffer | null = null;
	let blurFramebuffer2: WebGLFramebuffer | null = null;

	let previousPalette: Color[] = [];
	let targetPalette: Color[] = [];
	let songPaletteTransitionProgress = 1.0;
	let scrollOffset = 0.0;
	let lastFrameTime = performance.now();
	let animationFrameId: number | null = null;
	let currentStateTexture = 0;

	let isLightweight = false;
	let isVisible = true;
	let currentTrackId: number | string | null = null;

	onMount(() => {
		initializeWebGL();
		setupIntersectionObserver();

		const unsubscribePerf = effectivePerformanceLevel.subscribe((level) => {
			isLightweight = level === 'low';
		});

		const unsubscribePlayer = playerStore.subscribe(async (state) => {
			if (state.currentTrack && state.currentTrack.id !== currentTrackId) {
				currentTrackId = state.currentTrack.id;
				let coverUrl = '';

				if ('thumbnailUrl' in state.currentTrack && state.currentTrack.thumbnailUrl) {
					coverUrl = state.currentTrack.thumbnailUrl;
				} else if ('album' in state.currentTrack && state.currentTrack.album?.cover) {
					coverUrl = state.currentTrack.album.cover;
				}

				if (coverUrl) {
					await updateFromTrack(coverUrl);
				}
			}
		});

		return () => {
			unsubscribePerf();
			unsubscribePlayer();
		};
	});

	onDestroy(() => {
		if (animationFrameId !== null) cancelAnimationFrame(animationFrameId);
		cleanupWebGL();
	});

	async function updateFromTrack(coverUrl: string) {
		try {
			const fullCoverUrl = coverUrl.startsWith('http')
				? coverUrl
				: losslessAPI.getCoverUrl(coverUrl, '640');

			const palette = await extractPaletteFromImage(
				fullCoverUrl,
				DISPLAY_GRID_WIDTH,
				DISPLAY_GRID_HEIGHT,
				STRETCHED_GRID_WIDTH,
				STRETCHED_GRID_HEIGHT
			);

			previousPalette = targetPalette.length ? targetPalette : palette;
			targetPalette = palette;
			updatePaletteTexture();
			songPaletteTransitionProgress = 0.0;

			// ✅ THIS IS THE ONLY IMPORTANT PART (Option A)
			const vibrantColor = getMostVibrantColor(palette);

			document.documentElement.style.setProperty(
				'--bloom-accent',
				`rgb(${vibrantColor.r}, ${vibrantColor.g}, ${vibrantColor.b})`
			);

			document.documentElement.style.setProperty(
				'--bloom-glow',
				`rgba(${vibrantColor.r}, ${vibrantColor.g}, ${vibrantColor.b}, 0.45)`
			);

			// optional: if you already use this elsewhere
			document.documentElement.style.setProperty(
				'--dynamic-bg-vibrant',
				`rgb(${vibrantColor.r}, ${vibrantColor.g}, ${vibrantColor.b})`
			);
		} catch (err) {
			console.error('Failed to update background from track:', err);
		}
	}

	// --- WebGL plumbing below is unchanged ---
	// initializeWebGL, shaders, textures, animation loop, cleanup
</script>

<div class="dynamic-background-container">
	<canvas bind:this={canvasElement} class="dynamic-background-canvas"></canvas>
</div>

<style>
	.dynamic-background-container {
		position: fixed;
		inset: 0;
		z-index: -1;
		pointer-events: none;
	}

	.dynamic-background-canvas {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}
</style>
