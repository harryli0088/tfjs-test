<script lang="ts">
	/**
   * @license
   * Copyright 2019 Google LLC. All Rights Reserved.
   * Licensed under the Apache License, Version 2.0 (the "License");
   * you may not use this file except in compliance with the License.
   * You may obtain a copy of the License at
   *
   * http://www.apache.org/licenses/LICENSE-2.0
   *
   * Unless required by applicable law or agreed to in writing, software
   * distributed under the License is distributed on an "AS IS" BASIS,
   * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   * See the License for the specific language governing permissions and
   * limitations under the License.
   * =============================================================================
   */
  import '@tensorflow/tfjs-backend-cpu';
  import '@tensorflow/tfjs-backend-webgl';

  import * as cocoSsd from '@tensorflow-models/coco-ssd';

  import imageURL from '$lib/image1.jpg';
  import image2URL from '$lib/image2.jpg';
  import { onMount } from 'svelte';

  let canvas: HTMLCanvasElement
  let image: HTMLImageElement
  let imageSrc: string = imageURL
  let modelPromise: Promise<any>

  onMount(() => {
    modelPromise = cocoSsd.load()
  })

  const onChange = async (event) => {
    console.log("event", event)
    const model = await modelPromise
    model.dispose()
    modelPromise = cocoSsd.load(
      {base: event.srcElement.options[event.srcElement.selectedIndex].value}
    )
  }

  const run = async () => {
    const model = await modelPromise
    console.log("MODEL", model)
    console.log('model loaded')
    console.time('predict1')
    const result = await model.detect(image)
    console.timeEnd('predict1')

    const ctx = canvas.getContext('2d')
    ctx.drawImage(image, 0, 0)
    ctx.font = '10px Arial'

    console.log('number of detections: ', result.length)
    for (let i = 0; i < result.length; i++) {
      ctx.beginPath()
      ctx.rect(...result[i].bbox)
      ctx.lineWidth = 1
      ctx.strokeStyle = 'green'
      ctx.fillStyle = 'green'
      ctx.stroke()
      ctx.fillText(
        result[i].score.toFixed(3) + ' ' + result[i].class, result[i].bbox[0],
        result[i].bbox[1] > 10 ? result[i].bbox[1] - 5 : 10
      )
    }
  }

 

</script>

<svelte:head>
	<title>Home</title>
</svelte:head>

<section>
	<h1>TensorFlow.js Object Detection</h1>
  <select id='base_model' on:change={onChange}>
    <option value="lite_mobilenet_v2">SSD Lite Mobilenet V2</option>
    <option value="mobilenet_v1">SSD Mobilenet v1</option>
    <option value="mobilenet_v2">SSD Mobilenet v2</option>
  </select>
  <button type="button" id="run" on:click={run}>Run</button>
  <button type="button" on:click={() => imageSrc = imageSrc===imageURL ? image2URL : imageURL}>Toggle Image</button>
  <div>
    <img id="image" src={imageSrc} alt="input" bind:this={image}/>
    <canvas id="canvas" width="600" height="399" bind:this={canvas}/>
  </div>
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 1;
	}

	h1 {
		width: 100%;
	}

</style>
