<script lang="ts">
  import { onMount } from 'svelte'
  import { afterUpdate } from 'svelte/internal'
  import '@tensorflow/tfjs-backend-cpu'
  import '@tensorflow/tfjs-backend-webgl'
  import * as cocoSsd from '@tensorflow-models/coco-ssd'

  import beach from '$lib/beach.jpg'
  import dogs from '$lib/dogs.jpg'
  import kittens from '$lib/kittens.jpeg'

  let canvas: HTMLCanvasElement
  let canvasContainerWidth: number = 600
  let ctx: CanvasRenderingContext2D
  let DPR = 1
  let files: FileList
  let images:string[] = [
    beach,
    dogs,
    kittens
  ]
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
  let targetImageIndex: number = 0

  onMount(async () => {
    //DPR is important for improving the picture quality of the canvas, especially for text
    //based off this fiddle http://jsfiddle.net/65maD/83/ from this stack answer https://stackoverflow.com/a/54027313
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

  const changeImage = async (i: number) => {
    targetImageIndex = i
    targetImage = targetImage
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

  const uploadImages = () => {
    if(files) { //if there are files to read
      for(let i=0; i<files.length; ++i) { //loop through the files
        const file = files[i] //get the current file
        const reader = new FileReader() //create a FileReader to read the file
        reader.onload = () => { //on load callback
          images = images.concat(reader.result as string) //add the image to the array
        }
        reader.readAsDataURL(file) //read the file
      }
    }
  }
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
        {#each images as src,i}
          <img
            alt="analyze option"
            class={`image-option ${(targetImageIndex===i ? "focused" : "")}`}
            on:click={() => changeImage(i)}
            {src}
          />
        {/each}
      </div>

      <div>
        <label class="label-button" for="upload-images">Upload Your Own Image</label>
        <input type="file" id="upload-images" multiple accept="image/*" bind:files={files} on:change={uploadImages}>
      </div>
    </div>

    <br/>

    <div>
      <img
        alt="input"
        bind:this={targetImage}
        id="target-image"
        src={images[targetImageIndex]}
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
            <th>Confidence Score</th>
            <th>Top Left X px</th>
            <th>Top Left Y px</th>
            <th>Width px</th>
            <th>Height px</th>
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
      <div id="loading-dimmer">Loading Model...</div>
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
    background-color: rgba(240,240,240,0.9);
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
    border: 5px solid transparent;
    transition: 0.25s;
  }
  .image-option:last-child {
    margin-right: 0;
  }
  .image-option.focused {
    border-color: gold;
  }

  #upload-images {
    display: none;
  }
</style>
