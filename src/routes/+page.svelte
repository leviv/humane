<script lang="ts">
	import P5 from 'p5-svelte';
	import data from '$lib/questions.json';

	let agitation = 1;
	let currentQuestionIndex = 0;
	let selectedOption = '';
	let answerCorrect: boolean | undefined = undefined;

	// This will run when selectedOption changes
	$: if (selectedOption) {
		const selectedOptionIndex = data[currentQuestionIndex].options.indexOf(selectedOption);
		if (selectedOptionIndex >= 0) {
			const answers = data[currentQuestionIndex].answer;
			const currentAnswerCorrect = answers.includes(selectedOptionIndex);

			if (answerCorrect !== currentAnswerCorrect) {
				if (currentAnswerCorrect) {
					agitation = Math.max(1, agitation - 0.4);
				} else {
					agitation = Math.min(2, agitation + 0.4);
				}
				answerCorrect = currentAnswerCorrect;
			}
		}
	}

	// Handler when we click next question
	const onNextQuestion = () => {
		currentQuestionIndex = (currentQuestionIndex + 1) % data.length;
		selectedOption = '';
		answerCorrect = undefined;
	};

	// Handler when we click previous question
	const onPreviousQuestion = () => {
		currentQuestionIndex = (currentQuestionIndex - 1 + data.length) % data.length;
		selectedOption = '';
		answerCorrect = undefined;
	};

	const sketch = (p5: P5) => {
		let capture: any;
		const NUM_BLOB_VERTICES = 20;
		const IMAGE_WIDTH = 400;
		const IMAGE_HEIGHT = 250;
		const CANVAS_PADDING = 50;

		p5.setup = () => {
			p5.createCanvas(IMAGE_WIDTH + CANVAS_PADDING, IMAGE_HEIGHT + CANVAS_PADDING);
			const canvasEl = p5.canvas;
			// I think this is a svelte optiomization ? Not sure if it works
			const ctx = canvasEl.getContext('2d', { willReadFrequently: true });
			p5.drawingContext = ctx;
			capture = p5.createCapture(p5.VIDEO);
			capture.size(IMAGE_WIDTH, IMAGE_HEIGHT);
			capture.hide();
			p5.frameRate(60);
		};

		p5.draw = () => {
			p5.background(255);
			p5.translate(p5.width / 2 - IMAGE_WIDTH / 2, p5.height / 2 - IMAGE_HEIGHT / 2);

			capture.loadPixels();

			for (let col = 0; col < IMAGE_WIDTH; col += 10) {
				for (let row = 0; row < IMAGE_HEIGHT; row += 10) {
					// Pixel data is stored in a flat array, so we calculate the index
					const i = 4 * (row * IMAGE_WIDTH + col);
					const r = capture.pixels[i] * agitation;
					const g = capture.pixels[i + 1];
					const b = capture.pixels[i + 2];
					const a = capture.pixels[i + 3];

					p5.fill(r, g, b, a);
					p5.noStroke();
					drawBlob(col, row, 20);
				}
			}
		};

		// Save the canvas as an image when 's' is pressed
		p5.keyPressed = () => {
			if (p5.key == 's') {
				p5.save('humane.png');
			}
		};

		/**
		 * Function to draw a blob at a given center position. It animates and wiggles around
		 * @param centerX
		 * @param centerY
		 * @param maxHeight
		 */
		function drawBlob(centerX: number, centerY: number, maxHeight: number) {
			p5.push();
			p5.noStroke();
			p5.translate(centerX, centerY);
			p5.beginShape();

			const frameCountMod = p5.frameCount * (agitation * 10 - 9);
			const seed = (centerX * 73856093) ^ (centerY * 19349663); // fast hash
			const baseZ = (frameCountMod + (seed % 5000)) / 50;

			for (let i = 0; i < NUM_BLOB_VERTICES + 3; i++) {
				const pct = i / NUM_BLOB_VERTICES;
				const angle = pct * p5.TWO_PI;
				const nx = p5.map(p5.sin(angle), -1, 1, 0, 2);
				const ny = p5.map(p5.cos(angle), -1, 1, 0, 2);
				const nz = baseZ;

				// Use Perlin noise to create a smooth, organic shape
				const noiseVal = p5.noise(nz, ny, nx);
				let radius = p5.map(noiseVal, 0, 1, maxHeight / 20, maxHeight);
				if (i % 2 === 1) {
					radius *= agitation;
				}

				const x = radius * p5.sin(angle);
				const y = radius * p5.cos(angle);
				p5.curveVertex(x, y);
			}

			p5.endShape();
			p5.pop();
		}
	};
</script>

<!-- Change the icon and the tab title based on the agitation level. -->
<svelte:head>
	{#if agitation > 1.4}
		<link
			rel="icon"
			href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>😈</text></svg>"
		/>
	{:else}
		<link
			rel="icon"
			href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>😇</text></svg>"
		/>
	{/if}
	<title>
		{agitation > 1.4 ? 'In' : ''}Humane
	</title>
</svelte:head>

<div class="main-body">
	<div class="canvas">
		<p>You:</p>
		<P5 {sketch} />
		<p>Press <span class="save">s</span> to save the canvas</p>
	</div>

	<div class="quiz-content">
		<div class="title">
			<h1><span class="in" style="opacity: {agitation - 1 + 0.05};">In</span>Humane</h1>
		</div>

		<div class="question-intro">
			<h3>{data[currentQuestionIndex].question}</h3>
			<img src={data[currentQuestionIndex].image} alt={data[currentQuestionIndex].question} />
		</div>

		<div class="options">
			{#each data[currentQuestionIndex].options as option}
				<input type="radio" id={option} name="option" value={option} bind:group={selectedOption} />
				<label for={option}>{option}</label><br />
			{/each}
		</div>

		<div class="buttons">
			<button
				on:click={onPreviousQuestion}
				class="{currentQuestionIndex === 0 ? 'hidden' : ''} previous">&larr; Previous</button
			>
			<button
				on:click={onNextQuestion}
				class="{currentQuestionIndex === data.length - 1 ? 'hidden' : ''} next">Next &rarr;</button
			>
		</div>

		{#if selectedOption !== '' && data[currentQuestionIndex].explanation}
			<div class="explanation">
				<p>💡: {data[currentQuestionIndex].explanation}</p>
			</div>
		{/if}
	</div>
</div>

<footer>
	<p>Made with 💜 by <a href="https://leviv.cool" target="_blank">Levi</a> in NYC &copy; 2025</p>
</footer>

<style>
	:global(body) {
		padding: 0;
		margin: 5px;
		font-family: 'Overpass', sans-serif;
	}

	.main-body {
		display: flex;
		flex-direction: row;
		gap: 20px;
		margin: 0 auto;
		max-width: 1200px;
	}

	.question-intro {
		border: 1px solid black;
		padding: 10px;
		max-width: 700px;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.options {
		border: 1px solid black;
		margin-top: 10px;
		max-width: 700px;
		width: calc(100% - 20px);
		padding: 10px;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.buttons {
		max-width: 700px;
		width: calc(100% - 20px);
		padding: 10px;
		border: 1px solid black;
		margin-top: 10px;
		position: relative;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.explanation {
		max-width: 700px;
		width: calc(100% - 20px);
		padding: 3px 10px;
		border: 1px solid black;
		margin-top: 10px;
	}

	.next {
		float: right;
	}

	.previous {
		position: relative;
		left: 0;
	}

	h1 {
		text-align: left;
		font-family: 'Sansita', sans-serif;
		text-transform: uppercase;

		span {
			color: #ff6347;
		}
	}

	.canvas {
		margin-top: 82px;
		height: fit-content;
		z-index: -1;
		border: 1px solid black;
		text-align: center;
		padding: 5px 0;

		p {
			font-family: 'Sansita', sans-serif;
			font-size: 24px;
			padding: 0;
			margin: 0;
		}

		.save {
			font-weight: bold;
			color: #ff6347;
		}
	}

	.quiz-content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	img {
		width: 100%;
	}

	button {
		padding: 10px 20px;
		background-color: white;
		border: 1px solid black;
		&:hover {
			background-color: #f0f0f0;
			cursor: pointer;
		}
	}

	button.hidden {
		display: none;
	}

	footer {
		position: fixed;
		right: 25px;
		bottom: 0;
		text-align: right;
		width: 100%;
	}

	@media only screen and (max-width: 900px) {
		.main-body {
			flex-direction: column;
			margin-bottom: 100px;
		}

		.canvas {
			margin-top: 0;
		}
	}
</style>
