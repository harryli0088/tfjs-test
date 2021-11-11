<script lang="ts">
  import { onMount } from 'svelte'
  import { afterUpdate } from 'svelte/internal'
  import '@tensorflow/tfjs-backend-cpu'
  import '@tensorflow/tfjs-backend-webgl'
  import * as cocoSsd from '@tensorflow-models/coco-ssd'
  import { CLASSES } from '@tensorflow-models/coco-ssd/dist/classes'
  import Fa from 'svelte-fa'
  import { faGithub } from '@fortawesome/free-brands-svg-icons'
  import { faImage } from '@fortawesome/free-solid-svg-icons'

  import beach from '$lib/beach.jpg'
  import dogs from '$lib/dogs.jpg'
  import food from '$lib/food.jpeg'
  import kittens from '$lib/kittens.jpeg'
  import Blanchor from '$lib/Blanchor.svelte'

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
  let images:string[] = [ beach, dogs, kittens, food ] //carousel image optins
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

  const changeModel = async (base:cocoSsd.ObjectDetectionBaseModel, shouldRun:boolean=true) => {
    state = "loading";
    (await modelPromise).dispose() //dispose of the old model
    modelPromise = getModelPromise(base)
    shouldRun && await run() //run the new model on the target image
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
    //BUG: when you enter mobile mode in Chrome, the target image has 0 dimensions after the first click
    canvas.height = targetImage.height * DPR
    canvas.width = targetImage.width * DPR
    ctx.scale(DPR, DPR) //set the new scale via DPR 

    //loosely scale the font size and line thickness with image size
    const diag = Math.hypot(targetImage.height, targetImage.width)
    const thicknessScale = Math.ceil(diag / 1000)
    const fontSize = thicknessScale * 10

    ctx.drawImage(targetImage, 0, 0) //draw the target image
    ctx.font = `${fontSize}px Arial` //set the font

    results.forEach((r,i) => { //draw each result
      const isHovering = resultHoverIndex === i

      ctx.beginPath()
      ctx.rect(...r.bbox)
      ctx.lineWidth = isHovering ? 3*thicknessScale : thicknessScale
      ctx.strokeStyle = 'green'
      ctx.fillStyle = 'green'
      ctx.stroke()
      ctx.fillText(
        `${r.score.toFixed(3)} ${r.class}`,
        r.bbox[0],
        r.bbox[1] > fontSize ? r.bbox[1] - 5 : fontSize + 5
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

  const setRefrigerator = async () => {
    await changeModel("lite_mobilenet_v2", false)
    await changeTargetImage(3)
  }

  const sortedClasses = Object.values(CLASSES).map(c => c.displayName).sort()
  const CLASS_SLICES = [0,20,40,60]
</script>

<svelte:head>
	<title>TensorFlow.js - In-Browser Object Detection with SSD Mobilenet</title>
</svelte:head>

<header>
  <div>
    <h1>TensorFlow.js</h1>

    <div><i>In-Browser Object Detection with SSD Mobilenet</i></div>

    <br/>

    <div style="font-size:2em">
      <Blanchor href="https://github.com/harryli0088/tfjs-test">
        <Fa class="icon-button" icon={faGithub} style="color:white;"/>
      </Blanchor>
    </div>
  </div>
</header>

<section>
  <div style="position: relative">
    <div>
      <label for="base-model" style="display:inline-block;"><h3>Select a Model:</h3></label>

      <select id='base-model' on:change={e => changeModel(e.target.value)}>
        {#each MODEL_OPTIONS as o}
          <option value={o.value}>{o.display}</option>
        {/each}
      </select>
    </div>

    <div>
      <h3>Select an Image to Analyze:</h3>

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

      <div style="margin-top: 0.5em;text-align:right;">
        <label id="upload-images-button" class="label-button" for="upload-images">
          Upload Your Own Image
          &nbsp; <Fa icon={faImage}/>
        </label>
        <input type="file" id="upload-images" multiple accept="image/*" bind:files={files} on:change={uploadImages}>
      </div>
    </div>

    <h3>Analyzed Image</h3>

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

    <div id="table-container" style="overflow-x:auto;">
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

<section><hr/></section>

<section>
  <h2>Background</h2>

  <p>I made this website based off the <Blanchor href="https://github.com/tensorflow/tfjs-models/tree/master/coco-ssd">Tensorflow.js COCO-SSD demo</Blanchor>. The model is ported into Tensorflow.js and runs <i>completely</i> in your browser with no backend server component, pretty cool!</p>

  <h3>Why does the model [suck in some way]?</h3>
  <p>As with all machine learning models, the model is only as good as the data you give it. This model was trained on the COCO dataset and will not perform as well on images that look different. Hilariously, SSD Lite Mobilenet V2 thinks the food image is a <a href="#carousel" on:click={() => setRefrigerator()}>refrigerator</a>.</p>

  <h3>COCO-SSD</h3>
  <p>COCO-SSD is an object detection model trained on the <Blanchor href="https://cocodataset.org/#home">Common Objects in Context</Blanchor> (aka COCO) dataset. SSD stands for Single Shot MultiBox Detection
  which generates default boxes over different aspect ratios and scales, adjusts boxes during prediction time, and can combine predictions from multiple feature maps to handle various object sizes. SSD encapsulates all computation in a single Feed-Forward Convolutional Neural Network which eliminates proposal generation and resampling, making it easier to train and operationalize (<Blanchor href="https://arxiv.org/abs/1512.02325">full paper</Blanchor>).</p>


  <p>Currently the models support these {sortedClasses.length} classes:</p>

  <div id="classes-list">
    {#each CLASS_SLICES as slice}
      <div>
        <ul>
          {#each sortedClasses.slice(slice,slice+20) as c}
            <li>{c}</li>
          {/each}
        </ul>
      </div>
    {/each}
  </div>

  <h3>Tensorflow? JavaScript? Is this for real??</h3>
  <p>I could hardly believe it myself, which is why I made this test website. I guess there's no running away from JavaScript.</p>
  <p>Quoted from <Blanchor href="https://www.tensorflow.org/js/guide/platform_environment">the docs</Blanchor>: "TensorFlow.js executes operations on the GPU by running WebGL shader programs. These shaders are assembled and compiled lazily when the user asks to execute an operation".</p>
  <p>Say what you will about JavaScript; it's powerful enough to run client-side ML! (Some people even claim that you could do all the training in JS...but I haven't gotten there yet)</p>

  <h3>Advantages of client-side Machine Learning</h3>
  <p>Running ML models in a browser or even in a React Native mobile app can greatly simplify the application architecture. Instead of building a beefy server that can handle many prediction requests, you can simply cache the model in a Content Delivery Network and make the user's device do all the work for you!</p>

  <p>Of course, one of the downsides of client-side ML is that your model is published for everyone to see, not good if you have some secret sauce. Also, extremely large models could eat up network bandwidth and a user's processing power, and not all client devices have GPU hardware.</p>
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

  #table-container {
    padding: 0.5em;
    background-color: #eee;
  }

  table {
    border-collapse: collapse;
    width: 100%;
  }

  th, td {
    text-align: left;
    padding-left: 0.5em;
    padding-right: 0.5em;
    white-space: nowrap;
  }
  th:first-child, td:first-child {
    padding-left: 0;
  }
  th:last-child, td:last-child {
    padding-right: 0;
  }
  th:not(:first-child), td:not(:first-child) {
    text-align: right;
  }

  tbody tr {
    transition: 0.25s;
  }
  tbody tr:hover {
    background-color: #ddd;
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

  #classes-list {
    display: flex;
    flex-wrap: wrap;
    max-height: 450px;
    overflow-y: auto;
    background-color: #eee;
  }
  #classes-list > div {
    width: 100%;
  }
  @media only screen and (min-width: 300px) {
    #classes-list > div {
      width: 50%;
    }
  }
  @media only screen and (min-width: 600px) {
    #classes-list > div {
      width: 25%;
    }
  }
</style>
