<script lang="ts">
  import { Editor, Handle } from '@annotorious/annotorious/src';
  import { getMaskDimensions } from '@annotorious/annotorious';
  import type { Circle, Shape, Transform } from '@annotorious/annotorious';
  
  export let shape: Circle;
  export let computedStyle: string | undefined;
  export let transform: Transform;
  export let viewportScale: number = 1;
  export let svgEl: SVGSVGElement;

  $: geom = shape.geometry;

  const editor = (circle: Shape, handle: string, delta: [number, number]) => {
    const initialBounds = circle.geometry.bounds;

    let [x0, y0] = [initialBounds.minX, initialBounds.minY];
    let [x1, y1] = [initialBounds.maxX, initialBounds.maxY];

    const [dx, dy] = delta;

    if (handle === 'SHAPE') {
      x0 += dx;
      x1 += dx;
      y0 += dy;
      y1 += dy;
    } else {
      switch (handle) {
        case 'TOP': {
          y0 += dy;
          break;
        }

        case 'BOTTOM': {
          y1 += dy;
          break;
        }

        case 'LEFT': {
          x0 += dx;
          break;
        }

        case 'RIGHT': {
          x1 += dx;
          break;
        }
      }
    }

    let w = Math.abs(x1 - x0);
    let h = Math.abs(y1 - y0);

    let cx = (x0 + x1) / 2;
    let cy = (y0 + y1) / 2;


    if (handle === 'TOP' || handle === 'BOTTOM') {
      w = h; 
      cx = (initialBounds.minX + initialBounds.maxX) / 2; 
    } else if (handle === 'LEFT' || handle === 'RIGHT') {
      h = w; 
      cy = (initialBounds.minY + initialBounds.maxY) / 2; 
    }

    const r = w / 2;
    const x = cx - r;
    const y = cy - r;

    return {
      ...circle,
      geometry: {
        ...circle.geometry,
        cx, cy, r,
        bounds: {
          minX: x,
          minY: y,
          maxX: x + w,
          maxY: y + h
        }
      }
    };
  }

  $: mask = getMaskDimensions(geom.bounds, 2 / viewportScale);
  const maskId = `circle-mask-${Math.random().toString(36).substring(2, 12)}`;
</script>

<Editor
  shape={shape}
  transform={transform}
  editor={editor}
  svgEl={svgEl}
  on:grab
  on:change 
  on:release
  let:grab={grab}>
  <defs>
    <mask id={maskId} class="a9s-circle-editor-mask">
      <rect x={mask.x} y={mask.y} width={mask.w} height={mask.h} />
      <circle cx={geom.cx} cy={geom.cy} r={geom.r} />
    </mask>
  </defs>

  <circle 
    class="a9s-outer"
    mask={`url(#${maskId})`}
    on:pointerdown={grab('SHAPE')}
    cx={geom.cx} cy={geom.cy} r={geom.r} />

  <circle 
    class="a9s-inner a9s-shape-handle"
    style={computedStyle}
    on:pointerdown={grab('SHAPE')}
    cx={geom.cx} cy={geom.cy} r={geom.r} />

  <Handle 
    class="a9s-corner-top"
    on:pointerdown={grab('TOP')}
    x={geom.cx} y={geom.cy - geom.r} 
    scale={viewportScale} />

  <Handle 
    class="a9s-corner-handle-right"
    on:pointerdown={grab('RIGHT')}
    x={geom.cx + geom.r} y={geom.cy} 
    scale={viewportScale} />
  
  <Handle 
    class="a9s-corner-handle-bottom"
    on:pointerdown={grab('BOTTOM')}
    x={geom.cx} y={geom.cy + geom.r} 
    scale={viewportScale} />
    
  <Handle 
    class="a9s-corner-handle-left"
    on:pointerdown={grab('LEFT')}
    x={geom.cx - geom.r} y={geom.cy} 
    scale={viewportScale} />
</Editor>

<style>
  mask.a9s-circle-editor-mask > rect {
    fill: #fff;
  }

  mask.a9s-circle-editor-mask > circle {
    fill: #000;
  }
</style>