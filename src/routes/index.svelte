<script lang="ts">
  import '@tensorflow/tfjs-backend-cpu'
  import '@tensorflow/tfjs-backend-webgl'

  import * as cocoSsd from '@tensorflow-models/coco-ssd'

  import imageURL from '$lib/image1.jpg'
  import image2URL from '$lib/image2.jpg'
  import { onMount } from 'svelte'
  import { afterUpdate } from 'svelte/internal';

  //DPR is important for improving the picture quality of the canvas, especially for text
  //based off this fiddle http://jsfiddle.net/65maD/83/ from this stack answer https://stackoverflow.com/a/54027313

  let canvas: HTMLCanvasElement
  let canvasContainerWidth: number = 600
  let ctx: CanvasRenderingContext2D
  let DPR = 1
  let images:string[] = [
    imageURL,
    image2URL
  ]
  let imageSrc: string = imageURL
  let modelPromise: Promise<any>
  let predictionTime: number = 0
  let resultHoverIndex: number = -1
  let results: {
    bbox: [number,number,number,number],
    class: string,
    score: number,
  }[] = []
  let state:string = "loading"
  let targetImage: HTMLImageElement

  onMount(async () => {
    DPR = window.devicePixelRatio
    
    ctx = canvas.getContext('2d')
    ctx.scale(DPR, DPR)

    modelPromise = cocoSsd.load()
    await run()
    state = "loaded"
  })

  const onChange = async (event) => {
    state = "loading"
    const model = await modelPromise
    model.dispose()
    modelPromise = cocoSsd.load(
      {base: event.srcElement.options[event.srcElement.selectedIndex].value}
    )
    await run()
  }

  const changeImage = async (src: string) => {
    imageSrc = src
    await run()
  }

  const run = async () => {
    state = "loading"
    const model = await modelPromise //save models as static JSON?
    const start = Date.now()
    results = await model.detect(targetImage)
    predictionTime = Date.now() - start
    console.timeEnd('prediction time')

    state = "loaded"
  }

  $: height = targetImage?.height || 600
  $: width = targetImage?.width || 400

  afterUpdate(() => {
    ctx.drawImage(targetImage, 0, 0)
    ctx.font = '10px Arial'

    results.forEach((r,i) => {
      const isHovering = resultHoverIndex === i

      ctx.beginPath()
      ctx.rect(...r.bbox)
      ctx.lineWidth = isHovering ? 3 : 1
      ctx.strokeStyle = 'green'
      ctx.fillStyle = 'green'
      ctx.stroke()
      ctx.fillText(
        `${r.score.toFixed(3)} ${r.class}`,
        r.bbox[0],
        r.bbox[1] > 10 ? r.bbox[1] - 5 : 10
      )
    })
  })
</script>

<svelte:head>
	<title>Home</title>
</svelte:head>

<header>
  <h1>TensorFlow.js Object Detection</h1>
</header>

<section>
  <div style="position: relative">
    <h3>Select a Model</h3>
    <div>
      <select id='base_model' on:change={onChange}>
        <option value="lite_mobilenet_v2">SSD Lite Mobilenet V2</option>
        <option value="mobilenet_v1">SSD Mobilenet v1</option>
        <option value="mobilenet_v2">SSD Mobilenet v2</option>
      </select>
    </div>


    <div>
      <h3>Select an Image to Analyze</h3>
      <div id="carousel">
        {#each images as src}
          <img class="image-option" {src} alt="analyze option" on:click={() => changeImage(src)}/>
        {/each}
      </div>
    </div>

    <br/>

    <div>
      <img
        alt="input"
        bind:this={targetImage}
        id="target-image"
        src={imageSrc}
      />
    </div>

    <div bind:clientWidth={canvasContainerWidth}>
      <canvas
        bind:this={canvas}
        id="canvas"
        height={height}
        width={width}
      />
    </div>

    <br/>

    <div><b>Prediction Time:</b> {predictionTime}ms</div>

    <br/>

    <div>
      <table>
        <thead>
          <tr>
            <th>Class</th>
            <th>Score</th>
            <th>Top Left X</th>
            <th>Top Left Y</th>
            <th>Width</th>
            <th>Height</th>
          </tr>
        </thead>

        <tbody on:mouseout={() => resultHoverIndex = -1} on:blur={() => resultHoverIndex = -1}>
          {#each results as r,i}
            <tr on:mouseenter={() => resultHoverIndex = i}>
              <td>{r.class}</td>
              <td>{r.score.toFixed(3)}</td>
              <td>{Math.round(r.bbox[0])}</td>
              <td>{Math.round(r.bbox[1])}</td>
              <td>{Math.round(r.bbox[2])}</td>
              <td>{Math.round(r.bbox[3])}</td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>

    {#if state === "loading"}
      <div id="loading-dimmer">Loading...</div>
    {/if}
  </div>
</section>

<style>
  header {
		display: flex;
		align-items: center;
		justify-content: center;
		background-color: #17202A;
		color: white;
    text-align: center;
    padding-top: 5em;
    padding-bottom: 5em;
  }

  #target-image {
    display: none;
  }

  canvas {
    display: block;
    width: 100%;
  }

  table {
    border-collapse: collapse;
  }

  th, td {
    text-align: left;
    padding-left: 1em;
    padding-right: 1em;
  }
  th:first-child, td:first-child {
    padding-left: 0;
  }
  th:last-child, td:last-child {
    padding-right: 0;
  }

  tbody tr {
    transition: 0.5s;
  }
  tbody tr:hover {
    background-color: #eee;
  }

  #loading-dimmer {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(230,230,230,0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    font-weight: bold;
  }

  #carousel {
    display: flex;
  }

  .image-option {
    cursor: pointer;
    height: 100px;
    margin-right: 10px;
  }
  .image-option:last-child {
    margin-right: 0;
  }
</style>
