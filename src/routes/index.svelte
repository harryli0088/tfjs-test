<script lang="ts">
  import { onMount } from 'svelte'
  import { afterUpdate } from 'svelte/internal'
  import '@tensorflow/tfjs-backend-cpu'
  import '@tensorflow/tfjs-backend-webgl'
  import * as cocoSsd from '@tensorflow-models/coco-ssd'
  import Fa from 'svelte-fa'
  import { faGithub } from '@fortawesome/free-brands-svg-icons'
  import { faImage } from '@fortawesome/free-solid-svg-icons'

  import beach from '$lib/beach.jpg'
  import dogs from '$lib/dogs.jpg'
  import kittens from '$lib/kittens.jpeg'
  import Blanchor from '$lib/Blanchor.svelte';

  //all the available ObjectDetectionBaseModel options from node_modules/@tensorflow-models/coco-ssd/dist/index.d.ts
  const MODEL_OPTIONS:{display: string, value: cocoSsd.ObjectDetectionBaseModel }[] = [
    { display: "SSD Lite Mobilenet V2", value: 'lite_mobilenet_v2' },
    { display: "SSD Mobilenet V2", value: 'mobilenet_v2' },
    { display: "SSD Mobilenet V1", value: 'mobilenet_v1' },
  ]

  let canvas: HTMLCanvasElement //bound to canvas
  let canvasContainerWidth: number = 600 //bound to the width of the canvas container
  let ctx: CanvasRenderingContext2D //canvas context
  let DPR = 1 //device pixel ratio, to be set onMount
  let files: FileList //bound to the files upload input
  let images:string[] = [ beach, dogs, kittens ] //carousel image optins
  let modelPromise: Promise<any>
  let predictionTime: number = 0 //record performance of model
  let resultHoverIndex: number = -1 //which result is being hovered over
  let results: { //record results of model
    bbox: [number,number,number,number],
    class: string,
    score: number,
  }[] = []
  let state:string = "loading" //loading state
  let targetImage: HTMLImageElement //bound to hidden target image
  let targetImageIndex: number = 0 //which image option is currently selected

  //prep the selected model to be loaded
  const getModelPromise = (base:cocoSsd.ObjectDetectionBaseModel = "lite_mobilenet_v2") => cocoSsd.load({base})

  onMount(async () => {
    DPR = window.devicePixelRatio
    ctx = canvas.getContext('2d')

    modelPromise = getModelPromise()
    await run()
    state = "loaded"
  })

  const onChange = async (e) => {
    state = "loading";
    (await modelPromise).dispose() //dispose of the old model
    modelPromise = getModelPromise(e.target.value)
    await run() //run the new model on the target image
  }

  const changeTargetImage = async (i: number) => {
    targetImageIndex = i //change the target image
    await run() //run the model on the new image
  }

  const run = async () => {
    state = "loading"

    const model = await modelPromise //save models as static JSON?
    const start = Date.now() //record the start time
    results = await model.detect(targetImage) //run the model on the image
    predictionTime = Date.now() - start //record the time elapsed

    state = "loaded"
  }

  afterUpdate(() => {
    //DPR is important for improving the picture quality of the canvas, especially for text
    //based off this fiddle http://jsfiddle.net/65maD/83/ from this stack answer https://stackoverflow.com/a/54027313
    //set the canvas size to the image size scaled by DPR
    canvas.height = targetImage.height * DPR
    canvas.width = targetImage.width * DPR

    ctx.scale(DPR, DPR) //set the new scale via DPR 
    ctx.drawImage(targetImage, 0, 0) //draw the target image
    ctx.font = '10px Arial'

    results.forEach((r,i) => { //draw each result
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
          if(i === files.length - 1) { //if this is the last uploaded image
            changeTargetImage(images.length - 1) //auto analyze the last image
          }
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
  <div>
    <h1>TensorFlow.js Object Detection</h1>

    <div style="font-size:2em">
      <Blanchor href="https://github.com/harryli0088/tfjs-test">
        <Fa class="icon-button" icon={faGithub} style="color:white;"/>
      </Blanchor>
    </div>
  </div>
</header>

<section>
  <div style="position: relative">
    <h3>Select a Model</h3>

    <div>
      <select id='base_model' on:change={onChange}>
        {#each MODEL_OPTIONS as o}
          <option value={o.value}>{o.display}</option>
        {/each}
      </select>
    </div>

    <div>
      <h3>Select an Image to Analyze</h3>
      <div>
        <label id="upload-images-button" class="label-button" for="upload-images">
          Upload Your Own Image
          &nbsp; <Fa icon={faImage}/>
        </label>
        <input type="file" id="upload-images" multiple accept="image/*" bind:files={files} on:change={uploadImages}>
      </div>

      <br/>

      <div id="carousel-container">
        <div id="carousel">
          {#each images as src,i}
            <img
              alt="analyze option"
              class={`image-option ${(targetImageIndex===i ? "focused" : "")}`}
              on:click={() => changeTargetImage(i)}
              {src}
            />
          {/each}
        </div>
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
        style={`width:${canvasContainerWidth}px;`}
      />
    </div>

    <br/>

    <div><b>Prediction Time:</b> {predictionTime}ms</div>

    <br/>

    <div>
      {#if results.length > 0}
        <table>
          <thead>
            <tr>
              <th>Class</th>
              <th>Confidence</th>
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
      {:else}
        <div><b>The model did not detect any objects</b></div>
      {/if}
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

  #carousel-container {
    overflow-x: auto;
    padding: 0.5em;
    background-color: #eee;
  }

  #carousel {
    display: flex;
  }

  .image-option {
    cursor: pointer;
    height: 100px;
    border: 5px solid transparent;
    transition: 0.25s;
  }
  .image-option.focused {
    border-color: gold;
  }
  .image-option:hover {
    transform: scale(1.05);
  }

  #upload-images-button {
    background-color: orange;
    color: white;
  }

  #upload-images {
    display: none;
  }

  #target-image {
    display: none;
  }

  canvas {
    display: block;
  }

  table {
    border-collapse: collapse;
  }

  th, td {
    text-align: left;
    padding-left: 0.5em;
    padding-right: 0.5em;
  }
  th:first-child, td:first-child {
    padding-left: 0;
  }
  th:last-child, td:last-child {
    padding-right: 0;
  }
  td:not(:first-child) {
    text-align: right;
  }

  tbody tr {
    transition: 0.25s;
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
</style>
