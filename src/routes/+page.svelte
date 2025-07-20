<script lang="ts">
	import P5 from 'p5-svelte';
	import data from '$lib/questions.json';

	let agitation = 1;
	let currentQuestionIndex = 0;
	let selectedOption = '';
	let answerCorrect: boolean | undefined = undefined;

	$: if (selectedOption) {
		const selectedOptionIndex = data[currentQuestionIndex].options.indexOf(selectedOption);
		if (selectedOptionIndex >= 0) {
			const answers = data[currentQuestionIndex].answer;
			const currentAnswerCorrect = answers.includes(selectedOptionIndex);

			if (answerCorrect !== currentAnswerCorrect) {
				if (currentAnswerCorrect) {
					agitation = Math.max(1, agitation - 0.1);
				} else {
					agitation = Math.min(2, agitation + 0.1);
				}
				answerCorrect = currentAnswerCorrect;
			}
		}
	}

	const onNextQuestion = () => {
		currentQuestionIndex = (currentQuestionIndex + 1) % data.length;
		selectedOption = '';
		answerCorrect = undefined;
	};

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
			const ctx = canvasEl.getContext('2d', { willReadFrequently: true });
			p5.drawingContext = ctx;
			capture = p5.createCapture(p5.VIDEO);
			capture.size(IMAGE_WIDTH, IMAGE_HEIGHT);
			capture.hide();
		};

		p5.draw = () => {
			p5.background(255);
			p5.translate(p5.width / 2 - IMAGE_WIDTH / 2, p5.height / 2 - IMAGE_HEIGHT / 2);

			capture.loadPixels();

			for (let col = 0; col < IMAGE_WIDTH; col += 10) {
				for (let row = 0; row < IMAGE_HEIGHT; row += 10) {
					const i = 4 * (row * IMAGE_WIDTH + col);
					const r = capture.pixels[i] * agitation;
					const g = capture.pixels[i + 1];
					const b = capture.pixels[i + 2];
					const a = capture.pixels[i + 3];

					p5.fill(r, g, b, a);
					p5.noStroke();

					// Draw fewer blobs if performance is still a concern
					if ((col + row) % 20 === 0) {
						drawBlob(col, row, 20);
					}
				}
			}
		};

		function drawBlob(centerX: number, centerY: number, maxHeight: number) {
			p5.push();
			p5.noStroke();
			p5.translate(centerX, centerY);
			p5.beginShape();

			const frameCountMod = p5.frameCount;
			const seed = (centerX * 73856093) ^ (centerY * 19349663); // fast hash
			const baseZ = (frameCountMod + (seed % 5000)) / 50;

			for (let i = 0; i < NUM_BLOB_VERTICES + 3; i++) {
				const pct = i / NUM_BLOB_VERTICES;
				const angle = pct * p5.TWO_PI;
				const nx = p5.map(p5.sin(angle), -1, 1, 0, 2);
				const ny = p5.map(p5.cos(angle), -1, 1, 0, 2);
				const nz = baseZ;

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

<div class="title">
	<h1><span class="in" style="opacity: {agitation - 1 + 0.05};">In</span>Humane</h1>
</div>

<div class="canvas">
	<p>You:</p>
	<P5 {sketch} />
</div>

<div class="quiz-content">
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

<footer>
	<p>Made with 💜 by <a href="https://leviv.cool" target="_blank">Levi</a> in NYC &copy; 2025</p>
</footer>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: 'Overpass', sans-serif;
	}

	.question-intro {
		border: 1px solid black;
		padding: 10px;
		width: 700px;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.options {
		border: 1px solid black;
		margin-top: 10px;
		width: 700px;
		padding: 10px;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.buttons {
		width: 700px;
		padding: 10px;
		border: 1px solid black;
		margin-top: 10px;
		position: relative;
		background-color: rgba(255, 255, 255, 0.7);
	}

	.explanation {
		width: 700px;
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
		text-align: center;
		font-family: 'Sansita', sans-serif;
		text-transform: uppercase;

		span {
			color: #ff6347;
		}
	}

	.canvas {
		position: fixed;
		left: 10px;
		top: 10px;
		z-index: -1;
		border: 1px solid black;
		text-align: center;

		p {
			font-family: 'Sansita', sans-serif;
			font-size: 24px;
			padding: 8px 0 0 0;
			margin: 0;
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
</style>
