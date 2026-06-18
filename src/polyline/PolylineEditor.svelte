<script lang="ts">
  import { createEventDispatcher, onMount, tick } from 'svelte';
  import { Editor, Handle, MidpointHandle } from '@annotorious/annotorious/src';
  import type { Polyline, PolylineGeometry, PolylinePoint, Shape, Transform } from '@annotorious/annotorious';
  import { getPathMidpoint } from '../path/pathUtils';
  import { 
    approximateAsPolygon, 
    boundsFromPoints, 
    computeSVGPath, 
    getMaskDimensions, 
    isTouch 
  } from '@annotorious/annotorious';
  
  const dispatch = createEventDispatcher<{ change: Polyline }>();
  
  /** Time difference (milliseconds) required for registering a click/tap **/
  const CLICK_THRESHOLD = 250;

  /** Minimum distance (px) to shape required for midpoints to show */
  const MIN_HOVER_DISTANCE = 1000;

  /** Minimum distance (px) between corners required for midpoints to show **/
  const MIN_CORNER_DISTANCE = 12;

  /** Needed for the <mask> element **/
  const MIDPOINT_SIZE = 4.5;

  /** Props */
  export let shape: Polyline;
  export let computedStyle: string | undefined;
  export let transform: Transform;
  export let viewportScale: number = 1;
  export let svgEl: SVGSVGElement;

  /** Drawing tool layer **/
  let visibleMidpoint: number | undefined;
  let isHandleHovered = false;
  let lastHandleClick: number | null = null;
  let selectedCorners: number[] = [];

  $: geom = shape.geometry;
  $: isPolyline = (shape.geometry as PolylineGeometry & { isPolyline?: boolean }).isPolyline === true;

  $: midpoints = isTouch ? [] : geom.points.reduce<{ point: [number, number], visible: boolean }[]>((all, thisCorner, idx) => {
    const nextCorner = idx === geom.points.length - 1 
      // Last point
      ? (geom.closed ? geom.points[0] : undefined)
      : geom.points[idx + 1];

    if (!nextCorner)
      return all;
    
    const [x, y] = getPathMidpoint(thisCorner, nextCorner);

    const dist = Math.sqrt( 
      Math.pow(nextCorner.point[0] - x, 2) + Math.pow(nextCorner.point[1] - y, 2));

    // Don't show if the distance between the corners is too small
    const visible = dist > MIN_CORNER_DISTANCE / viewportScale;

    return [...all, { point: [x, y], visible }];
  }, []);

  /** Handle hover state **/
  const onEnterHandle = () => isHandleHovered = true;
  const onLeaveHandle = () => isHandleHovered = false;

  /** Determine visible midpoint, if any **/
  const onPointerMove = (evt: PointerEvent) => {
    if (selectedCorners.length > 0 || !midpoints.some(m => m.visible)) {
      visibleMidpoint = undefined;
      return;
    }

    const [px, py] = transform.elementToImage(evt.offsetX, evt.offsetY);

    const getDistSq = (pt: number[]) =>
      Math.pow(pt[0] - px, 2) + Math.pow(pt[1] - py, 2);
    
    const closestCorner = geom.points.reduce((closest, corner) =>
      getDistSq(corner.point) < getDistSq(closest.point) ? corner : closest);

    const closestVisibleMidpoint = midpoints
      .filter(m => m.visible)
      .reduce((closest, midpoint) =>
        getDistSq(midpoint.point) < getDistSq(closest.point) ? midpoint : closest);

    // Show midpoint if the mouse is at least within THRESHOLD distance
    // of the midpoint or the closest corner. (Basically a poor man's shape buffering).
    const threshold = Math.pow(MIN_HOVER_DISTANCE / viewportScale, 2);

    const shouldShow = 
      getDistSq(closestCorner.point) < threshold ||
      getDistSq(closestVisibleMidpoint.point) < threshold;

    if (shouldShow)
      visibleMidpoint = midpoints.indexOf(closestVisibleMidpoint);
    else
      visibleMidpoint = undefined;
  }

  /** 
   * SVG element keeps losing focus when interacting with 
   * shapes–this function refocuses.
   */
  const reclaimFocus = () => {
    if (document.activeElement !== svgEl)
      svgEl.focus();
  }

  /**
   * De-selects all corners and reclaims focus.
   */
  const onShapePointerUp = () => {
    selectedCorners = [];
    reclaimFocus();
  }

  /**
   * Updates state, waiting for potential click.
   */
  const onHandlePointerDown = (evt: PointerEvent) => {
    isHandleHovered = true;

    evt.preventDefault();
    evt.stopPropagation();

    lastHandleClick = performance.now();
  }

  /** Selection handling logic **/
  const onHandlePointerUp = (idx: number) => (evt: PointerEvent) => {
    if (!lastHandleClick) return;

    // Drag, not click
    if (performance.now() - lastHandleClick > CLICK_THRESHOLD) return;

    const isSelected = selectedCorners.includes(idx);

    if (selectedCorners.length > 0 && (evt.metaKey || evt.ctrlKey || evt.shiftKey)) {
      if (isSelected && selectedCorners.length > 1) 
        selectedCorners = selectedCorners.filter(i => i !== idx);
      else if (isSelected)
        selectedCorners = [];
      else
        selectedCorners = [...selectedCorners, idx];
    } else {
      if (!isSelected || selectedCorners.length > 1)
        selectedCorners = [idx];
    }
  }

  const editor = (polyline: Shape, handle: string, delta: [number, number]) => {
    const geom = (polyline.geometry) as PolylineGeometry;

    const [dx, dy] = delta;

    let points: PolylinePoint[];

    if (handle === 'SHAPE') {
      points = geom.points.map(pt => ({
        ...pt,
        point: [pt.point[0] + dx, pt.point[1] + dy],
        inHandle: pt.inHandle ? [pt.inHandle[0] + dx, pt.inHandle[1] + dy] : undefined,
        outHandle: pt.outHandle ? [pt.outHandle[0] + dx, pt.outHandle[1] + dy] : undefined
      }));
    } else if (handle.startsWith('CORNER-')) {
      const idx = parseInt(handle.split('-')[1]);
      
      points = geom.points.map((pt, i) => i === idx ? {
        ...pt,
        point: [pt.point[0] + dx, pt.point[1] + dy],
        inHandle: pt.inHandle ? [pt.inHandle[0] + dx, pt.inHandle[1] + dy] : undefined,
        outHandle: pt.outHandle ? [pt.outHandle[0] + dx, pt.outHandle[1] + dy] : undefined
      } : pt);
    } else if (handle.startsWith('IN-') || handle.startsWith('OUT-')) {
      if (isPolyline)
        return polyline as Polyline;

      const [handleType, idxStr] = handle.split('-');
      const idx = parseInt(idxStr);
      
      points = geom.points.map((pt, i) => {
        if (i === idx && pt.type === 'CURVE') {
          const locked = pt.locked;

          const newPt = { ...pt, locked };
          
          if (handleType === 'IN' && pt.inHandle) {
            newPt.inHandle = [pt.inHandle[0] + dx, pt.inHandle[1] + dy];
            
            // Maintain symmetry for smooth curves
            if (pt.outHandle && locked) {
              const inDx = newPt.inHandle[0] - pt.point[0];
              const inDy = newPt.inHandle[1] - pt.point[1];
              const inLength = Math.sqrt(inDx ** 2 + inDy ** 2);

              const outLength = Math.sqrt(
                (pt.outHandle[0] - pt.point[0]) ** 2 + 
                (pt.outHandle[1] - pt.point[1]) ** 2
              );
              
              if (inLength > 0) {
                newPt.outHandle = [
                  pt.point[0] - (inDx / inLength) * outLength,
                  pt.point[1] - (inDy / inLength) * outLength
                ];
              }
            }
          } else if (handleType === 'OUT' && pt.outHandle) {
            newPt.outHandle = [pt.outHandle[0] + dx, pt.outHandle[1] + dy];
            
            // Maintain symmetry for smooth curves
            if (pt.inHandle && locked) {
              const outDx = newPt.outHandle[0] - pt.point[0];
              const outDy = newPt.outHandle[1] - pt.point[1];
              const outLength = Math.sqrt(outDx ** 2 + outDy ** 2);

              const inLength = Math.sqrt(
                (pt.inHandle[0] - pt.point[0]) ** 2 + 
                (pt.inHandle[1] - pt.point[1]) ** 2
              );
              
              if (outLength > 0) {
                newPt.inHandle = [
                  pt.point[0] - (outDx / outLength) * inLength,
                  pt.point[1] - (outDy / outLength) * inLength
                ];
              }
            }
          }
          
          return newPt;
        }

        return pt;
      });
    } else {   
      // Should never happen
      points = geom.points;
    }

    const bounds = boundsFromPoints(approximateAsPolygon(points, geom.closed));

    return {
      ...polyline,
      geometry: {
        bounds,
        points,
        closed: geom.closed,
        isPolyline: isPolyline
      }
    } as Polyline;
  }

const onAddPoint = (midpointIdx: number) => async (evt: PointerEvent) => {
    evt.stopPropagation();

    const points = [
      ...geom.points.slice(0, midpointIdx + 1),
      { type: 'CORNER', point: midpoints[midpointIdx].point },
      ...geom.points.slice(midpointIdx + 1)
    ] as PolylinePoint[];

    const bounds = boundsFromPoints(approximateAsPolygon(points, geom.closed));

    dispatch('change', {
      ...shape,
      geometry: { points, bounds, closed: geom.closed, isPolyline: isPolyline }
    });

    await tick();

    const newHandle = [...document.querySelectorAll(`.a9s-handle`)][midpointIdx + 1];
    if (newHandle?.firstChild) {
      const newEvent = new PointerEvent('pointerdown', {
        bubbles: true,
        cancelable: true,
        clientX: evt.clientX,
        clientY: evt.clientY,
        pointerId: evt.pointerId,
        pointerType: evt.pointerType,
        isPrimary: evt.isPrimary,
        buttons: evt.buttons
      });

      newHandle.firstChild.dispatchEvent(newEvent);
    }
  }

  const onDeleteSelected = () => {
    // Open path needs 2 points min, closed path needs 3
    const minLen = geom.closed ? 3 : 2;
    if (geom.points.length - selectedCorners.length < minLen) return;

    const points = geom.points.filter((_, i) => !selectedCorners.includes(i));
    const bounds = boundsFromPoints(approximateAsPolygon(points, geom.closed));

    dispatch('change', {
      ...shape,
      geometry: { 
        closed: shape.geometry.closed,
        bounds,
        points,
        isPolyline: isPolyline
      }
    });

    selectedCorners = [];
  }

  onMount(() => {
    const onKeyDown = (evt: KeyboardEvent) => {
      if (evt.key === 'Delete' || evt.key === 'Backspace')
        onDeleteSelected();
    }
  
    const onKeyUp = () => {
      // no-op
    }

    svgEl.addEventListener('pointermove', onPointerMove);

    window.addEventListener('keydown', onKeyDown);
    window.addEventListener('keyup', onKeyUp);

    return () => {
      svgEl.removeEventListener('pointermove', onPointerMove);

      window.removeEventListener('keydown', onKeyDown);
      window.removeEventListener('keyup', onKeyUp);
    }
  });

  $: d = computeSVGPath(geom);

  $: mask = getMaskDimensions(geom.bounds, MIDPOINT_SIZE / viewportScale);

  const maskId = `polygon-mask-${Math.random().toString(36).substring(2, 12)}`;
</script>

<Editor
  shape={shape}
  transform={transform}
  editor={editor}
  svgEl={svgEl}
  on:change 
  on:grab
  on:release
  let:grab={grab}>

  <defs>
    {#if (visibleMidpoint !== undefined && !isHandleHovered)}
      {@const { point } = midpoints[visibleMidpoint]}
      <mask id={maskId}  class="a9s-polygon-editor-mask">
        <rect x={mask.x} y={mask.y} width={mask.w} height={mask.h} /> 
        <circle cx={point[0]} cy={point[1]} r={MIDPOINT_SIZE / viewportScale} />
      </mask>
    {/if}
  </defs>

  <g mask={`url(#${maskId})`}>
    <rect x={mask.x} y={mask.y} width={mask.w} height={mask.h} class="mask-buffer" /> 
    <path
      class={`a9s-outer polyline ${shape.geometry.closed ? 'closed' : 'open'}`}
      on:pointerup={onShapePointerUp}
      on:pointerdown={grab('SHAPE')}
      d={d} />

    <path
      class={`a9s-inner polyline a9s-shape-handle ${shape.geometry.closed ? 'closed' : 'open'}`}
      style={computedStyle}
      on:pointerup={onShapePointerUp}
      on:pointerdown={grab('SHAPE')}
      d={d} />
  </g>

  {#each geom.points as pt, idx}
    <Handle 
      class="a9s-corner-handle"
      x={pt.point[0]}
      y={pt.point[1]}
      scale={viewportScale}
      selected={selectedCorners.includes(idx)}
      on:pointerenter={onEnterHandle}
      on:pointerleave={onLeaveHandle}
      on:pointerdown={onHandlePointerDown}
      on:pointerdown={grab(`CORNER-${idx}`)}
      on:pointerup={onHandlePointerUp(idx)} />
  {/each}

  {#if (visibleMidpoint !== undefined && !isHandleHovered)}
    {@const { point } = midpoints[visibleMidpoint]}
    <MidpointHandle 
      x={point[0]}
      y={point[1]}
      scale={viewportScale} 
      on:pointerdown={onAddPoint(visibleMidpoint)} />
  {/if}
</Editor>

<style>
  :global(.a9s-annotationlayer .a9s-annotation) path.polyline.open {
    fill: transparent !important;
  }

  mask.a9s-polygon-editor-mask > rect {
    fill: #fff;
  }

  mask.a9s-polygon-editor-mask > circle {
    fill: #000;
  }

  .mask-buffer {
    fill: none;
  }
</style>