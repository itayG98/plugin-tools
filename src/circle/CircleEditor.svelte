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
    const isCircle = (circle.geometry as Circle['geometry'] & { isCircle?: boolean }).isCircle === true;
    const initialCx = circle.geometry.cx;
    const initialCy = circle.geometry.cy;
    const initialR = circle.geometry.r;

    const [dx, dy] = delta;

    if (handle === 'SHAPE') {
      const x = initialBounds.minX + dx;
      const y = initialBounds.minY + dy;

      return {
        ...circle,
        geometry: {
          ...circle.geometry,
          cx: initialCx + dx,
          cy: initialCy + dy,
          r: initialR,
          isCircle,
          bounds: {
            minX: x,
            minY: y,
            maxX: x + 2 * initialR,
            maxY: y + 2 * initialR
          }
        }
      };
    }

    let r = initialR;

    switch (handle) {
      case 'TOP':
        r = Math.max(0, Math.abs(initialCy - (initialBounds.minY + dy)));
        break;

      case 'BOTTOM':
        r = Math.max(0, Math.abs((initialBounds.maxY + dy) - initialCy));
        break;

      case 'LEFT':
        r = Math.max(0, Math.abs(initialCx - (initialBounds.minX + dx)));
        break;

      case 'RIGHT':
        r = Math.max(0, Math.abs((initialBounds.maxX + dx) - initialCx));
        break;
    }

    const x = initialCx - r;
    const y = initialCy - r;

    return {
      ...circle,
      geometry: {
        ...circle.geometry,
        cx: initialCx,
        cy: initialCy,
        r,
        isCircle,
        bounds: {
          minX: x,
          minY: y,
          maxX: x + 2 * r,
          maxY: y + 2 * r
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