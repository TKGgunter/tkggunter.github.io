+++
title = "Testing a post with WASM"
date = 2021-02-24
description = "Would you like to see some wasm rendering?"
+++

Can you see this?
This is an example from [here](http://cliffle.com/blog/bare-metal-wasm/).
And, yes I'm using the same theme as Cliffle.

<script type="module">

  import init, { render } from '../paint.js';
  async function run(div_id) {

    let wasm = await init('../paint_bg.wasm');
    const width = 600;
    const height = 600;

    const canvas = document.getElementById("canvas");
    const div = document.getElementById(div_id);

    canvas.width = width;
    canvas.height = height;

    div.appendChild(canvas);

    const buffer_address = wasm.BUFFER.value;
    const ctx = canvas.getContext("2d");

    const render_loop = () => {
      render();

      const image = new ImageData(
            new Uint8ClampedArray(
                wasm.memory.buffer,
                buffer_address,
                4 * width * height,
            ),
            width,
      );
      ctx.putImageData(image, 0, 0);
      requestAnimationFrame(render_loop);
    };
    render_loop();


  }
  run("post");
</script>

<canvas id="canvas"></canvas>

