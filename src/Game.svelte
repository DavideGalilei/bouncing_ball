<script lang="ts">
  import {
    Engine,
    Render,
    World,
    Bodies,
    Runner,
    Composite,
    Body,
    Events,
  } from "matter-js";
  import { onMount } from "svelte";

  export let playNote: (ticks: number) => Promise<void>;
  let canvas: HTMLCanvasElement;

  let innerWidth: number,
    outerWidth: number,
    innerHeight: number,
    outerHeight: number;

  let bouncingBall: Body;
  let velocityMagnitude = 5; // Desired constant speed

  function containerPolygon(
    x: number,
    y: number,
    sides: number,
    radius: number,
    options: {
      extraLength?: number;
      width?: number;
      initialRotation?: number;
      bodyOptions: Matter.IBodyDefinition;
    }
  ): Body {
    const width = options.width || 20;
    delete options.width;
    const extraLength = options.extraLength || 1.15;
    delete options.extraLength;
    const initialRotation = options.initialRotation || 0;
    delete options.initialRotation;

    const theta = (2 * Math.PI) / sides;
    const sideLength = ((2 * radius * theta) / 2) * extraLength;

    const parts = [];
    for (let i = 0; i < sides; i++) {
      const body = Bodies.rectangle(0, 0, sideLength, width, options.bodyOptions);
      Body.rotate(body, i * theta);
      Body.translate(body, {
        x: radius * Math.sin(i * theta),
        y: -radius * Math.cos(i * theta),
      });
      parts.push(body);
    }
    const ret = Body.create({
      parts: parts,
      ...options.bodyOptions,
    });
    Body.setParts(ret, parts);
    if (initialRotation) {
      Body.rotate(ret, initialRotation);
    }
    Body.translate(ret, { x: x, y: y });

    return ret;
  }

  onMount(async () => {
    const engine = Engine.create();
    const render = Render.create({
      canvas: canvas,
      engine: engine,
      options: {
        wireframes: false,
      },
    });

    const ring = containerPolygon(innerWidth / 2, innerHeight / 2, 40, 150, {
      extraLength: 1.15,
      width: 10,
      bodyOptions: {
        isStatic: true,
        render: {
          fillStyle: "#f92ab1",
        },
      },
    });

    bouncingBall = Bodies.rectangle(innerWidth / 2, innerHeight / 2, 40, 40, {
      restitution: 1,
      frictionAir: 0,
      render: {
        fillStyle: "#00ff00",
      },
    });

    // Set initial velocity
    Body.setVelocity(bouncingBall, { x: velocityMagnitude, y: 0 });

    engine.gravity.scale = 0;

    const debounce = (fn: () => void, ms: number) => {
      let timeout: number;
      return () => {
        clearTimeout(timeout);
        timeout = setTimeout(fn, ms);
      };
    };

    const playNoteDebounced = debounce(() => playNote(500), 300);

    Events.on(engine, "collisionStart", (event) => {
      const pairs = event.pairs;
      for (let pair of pairs) {
        if (pair.bodyA === bouncingBall || pair.bodyB === bouncingBall) {
          playNoteDebounced();
        }
      }
    });

    // Constantly adjust ball speed to maintain constant velocity
    Events.on(engine, "beforeUpdate", () => {
      const velocity = bouncingBall.velocity;
      const speed = Math.sqrt(velocity.x * velocity.x + velocity.y * velocity.y);
      const scale = velocityMagnitude / speed;
      Body.setVelocity(bouncingBall, { x: velocity.x * scale, y: velocity.y * scale });
    });

    render.canvas.style.backgroundColor = "#ff0000";

    Composite.add(engine.world, [ring, bouncingBall]);
    Render.run(render);

    const runner = Runner.create();
    Runner.run(runner, engine);
  });

  function randomBallForce() {
    const angle = Math.random() * 2 * Math.PI;
    const force = { x: Math.cos(angle) * 0.1, y: Math.sin(angle) * 0.1 };
    Body.setVelocity(bouncingBall, force);
  }
</script>

<svelte:window
  bind:innerWidth
  bind:outerWidth
  bind:innerHeight
  bind:outerHeight
  on:click={randomBallForce}
/>
<canvas width={innerWidth} height={innerHeight} bind:this={canvas}> </canvas>

<style>
  canvas {
    width: 100%;
    height: 100%;
  }
</style>
