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

	// =====================
	// Constants
	// =====================
	const DISPLAY_CANVAS_SIZE = 512;
	const MASTER_PALETTE_SIZE = 40;
	const DISPLAY_GRID_WIDTH = 8;
	const DISPLAY_GRID_HEIGHT = 5;
	const STRETCHED_GRID_WIDTH = 32;
	const STRETCHED_GRID_HEIGHT = 18;
	const BLUR_DOWNSAMPLE_FACTOR = 26;
	const SONG_PALETTE_TRANSITION_SPEED = 0.015;
	const SCROLL_SPEED = 0.008;

	// =====================
	// WebGL State
	// =====================
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

	// =====================
	// Lifecycle
	// =====================
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
		if (animationFrameId !== null) {
			cancelAnimationFrame(animationFrameId);
		}
		cleanupWebGL();
	});

	// =====================
	// Track → Palette → UI
	// =====================
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

			// =====================
			// ✅ OPTION A: UI COLOR = RAW ALBUM COLOR
			// =====================
			const vibrant = getMostVibrantColor(palette);

			document.documentElement.style.setProperty(
				'--bloom-accent',
				`rgb(${vibrant.r}, ${vibrant.g}, ${vibrant.b})`
			);

			document.documentElement.style.setProperty(
				'--bloom-glow',
				`rgba(${vibrant.r}, ${vibrant.g}, ${vibrant.b}, 0.45)`
			);

			document.documentElement.style.setProperty(
				'--dynamic-bg-vibrant',
				`rgb(${vibrant.r}, ${vibrant.g}, ${vibrant.b})`
			);
		} catch (error) {
			console.error('Failed to update background from track:', error);
		}
	}

	// =====================
	// WebGL Setup / Render
	// =====================
	function initializeWebGL() {
		if (!canvasElement) return;

		canvasElement.width = DISPLAY_CANVAS_SIZE;
		canvasElement.height = DISPLAY_CANVAS_SIZE;

		gl = canvasElement.getContext('webgl', {
			alpha: false,
			antialias: false,
			depth: false
		});

		if (!gl) return;

		setupShaders();
		setupTextures();
		setupFramebuffers();
		initializeCellStates();
		startAnimation();
	}

	function setupShaders() {
		if (!gl) return;
		const vs = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
		updateStateProgram = createProgram(gl, vs!, createShader(gl, gl.FRAGMENT_SHADER, updateStateShaderSource)!);
		colorRenderProgram = createProgram(gl, vs!, createShader(gl, gl.FRAGMENT_SHADER, colorRenderShaderSource)!);
		blurProgram = createProgram(gl, vs!, createShader(gl, gl.FRAGMENT_SHADER, blurFragmentShaderSource)!);
		setupQuad(gl);
	}

	function setupTextures() {
		if (!gl) return;
		const blurW = Math.round(DISPLAY_CANVAS_SIZE / BLUR_DOWNSAMPLE_FACTOR);
		const blurH = blurW;

		paletteTexture = createTexture(gl, DISPLAY_GRID_WIDTH, DISPLAY_GRID_HEIGHT * 2);
		cellStateTexture = createTexture(gl, DISPLAY_GRID_WIDTH, DISPLAY_GRID_HEIGHT);
		cellStateTexture2 = createTexture(gl, DISPLAY_GRID_WIDTH, DISPLAY_GRID_HEIGHT);
		colorRenderTexture = createTexture(gl, STRETCHED_GRID_WIDTH, STRETCHED_GRID_HEIGHT);
		blurTexture1 = createTexture(gl, blurW, blurH);
		blurTexture2 = createTexture(gl, blurW, blurH);
	}

	function setupFramebuffers() {
		if (!gl) return;
		stateFramebuffer1 = createFramebuffer(gl, cellStateTexture);
		stateFramebuffer2 = createFramebuffer(gl, cellStateTexture2);
		colorRenderFramebuffer = createFramebuffer(gl, colorRenderTexture);
		blurFramebuffer1 = createFramebuffer(gl, blurTexture1);
		blurFramebuffer2 = createFramebuffer(gl, blurTexture2);
	}

	function initializeCellStates() {
		if (!gl || !cellStateTexture) return;
		const data = new Uint8Array(DISPLAY_GRID_WIDTH * DISPLAY_GRID_HEIGHT * 4);
		for (let i = 0; i < MASTER_PALETTE_SIZE; i++) {
			data[i * 4] = Math.random() * 255;
			data[i * 4 + 1] = Math.random() * 255;
			data[i * 4 + 2] = Math.random() * 255;
			data[i * 4 + 3] = Math.random() * 255;
		}
		gl.bindTexture(gl.TEXTURE_2D, cellStateTexture);
		gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, DISPLAY_GRID_WIDTH, DISPLAY_GRID_HEIGHT, 0, gl.RGBA, gl.UNSIGNED_BYTE, data);
	}

	function updatePaletteTexture() {
		if (!gl || !paletteTexture) return;
		const data = new Uint8Array(DISPLAY_GRID_WIDTH * DISPLAY_GRID_HEIGHT * 2 * 4);
		previousPalette.forEach((c, i) => {
			data.set([c.r, c.g, c.b, 255], i * 4);
		});
		targetPalette.forEach((c, i) => {
			data.set([c.r, c.g, c.b, 255], (MASTER_PALETTE_SIZE + i) * 4);
		});
		gl.bindTexture(gl.TEXTURE_2D, paletteTexture);
		gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, DISPLAY_GRID_WIDTH, DISPLAY_GRID_HEIGHT * 2, 0, gl.RGBA, gl.UNSIGNED_BYTE, data);
	}

	function startAnimation() {
		if (!animationFrameId) animationFrameId = requestAnimationFrame(animate);
	}

	function animate(time: number) {
		animationFrameId = requestAnimationFrame(animate);
	}
	function setupIntersectionObserver() {}
	function cleanupWebGL() {}
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
