<script>
  import datos from './datos.json';
  const fechaInicio = new Date('2024-01-01');
  const fechaFin = new Date('2025-01-01');
  function getAllDates(start, end) {
    const arr = [];
    const dt = new Date(start);
    while (dt <= end) {
      arr.push(new Date(dt));
      dt.setDate(dt.getDate() + 1);
    }
    return arr;
  }
  const dias = getAllDates(fechaInicio, fechaFin);
  const dataMap = new Map();
  const detallesMap = new Map();
  for (const d of datos) {
    let fecha = d.fecha;
    if (!fecha && d.fecha_txt) {
      const [day, month, year] = d.fecha_txt.split('/');
      fecha = `${year}-${month.padStart(2, '0')}-${day.padStart(2, '0')}`;
    }
    if (fecha && d.via && !dataMap.has(fecha)) {
      dataMap.set(fecha, d.via);
      detallesMap.set(fecha, d);
    }
  }
  // Paleta definida por ti: de menor a mayor toneladas
  const escalaColores = ['#8afedf', '#01f3b3', '#01d099', '#01a277'];
  const toneladasList = datos.map(d => Number(d.toneladas)).filter(x => !isNaN(x));
  const minTon = Math.min(...toneladasList);
  const maxTon = Math.max(...toneladasList);

  function getColorPorTons(toneladas) {
    if (isNaN(toneladas)) return '#E0E0E0';
    const percent = (toneladas - minTon) / (maxTon - minTon);
    if (percent <= 0.25) return escalaColores[0];
    else if (percent <= 0.5) return escalaColores[1];
    else if (percent <= 0.75) return escalaColores[2];
    else return escalaColores[3];
  }
  function getRectColor(fecha) {
    const d = detallesMap.get(fecha);
    if (!d || d.toneladas == null) return '#E0E0E0';
    return getColorPorTons(Number(d.toneladas));
  }
  // Formato europeo para toneladas (punto miles, coma decimal)
  function formatTonsEU(n) {
    if (n == null || isNaN(n)) return "";
    return Number(n).toLocaleString('es-ES', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
  }
  function getTooltipContent(d, fecha) {
    if (!d) return 'Sin datos';
    let html = '';
    if (d.fecha_txt) html += `<b>Fecha:</b> ${d.fecha_txt}`;
    if (d.toneladas) html += `<br><b>Toneladas:</b> ${formatTonsEU(d.toneladas)}`;
    return html;
  }
  const anchoRect = 24;
  const altoRect = 24;
  const sep = 30;
  const offx = 18;
  const offy = 18;
  const nFilas = 15;
  const nColumnas = Math.ceil(dias.length / nFilas);
  function toISO(dia) {
    return dia.toISOString().slice(0, 10);
  }
  const viewWidth = nColumnas * sep + offx * 2;
  const viewHeight = nFilas * sep + offy * 2;
  let tooltipVisible = false;
  let tooltipX = 0;
  let tooltipY = 0;
  let tooltipContent = '';
  let tooltipColor = '#000';
  let container;
</script>

<style>
  svg {
    display: block;
    margin: auto;
    height: auto;
    width: 100%;
  }
  .tooltip {
    position: absolute;
    background: white;
    border: 2px solid;
    padding: 10px;
    font-size: 0.95rem;
    pointer-events: none;
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    border-radius: 5px;
    z-index: 10;
    max-width: 240px;
    white-space: nowrap;
    text-align: left;
  }
  .leyenda {
    font-size: 0.8rem;
    font-family: Helvetica, sans-serif;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    gap: 8px;
    margin-left: 6px;
    margin-top: 0;
    margin-bottom: 0.12rem;
    min-height: 15px;
  }
  .leyenda-item {
    display: flex;
    align-items: center;
    gap: 3px;
    margin-right: 10px;
    white-space: nowrap;
  }
  .leyenda-color {
    display: inline-block;
    width: 12px;
    height: 12px;
    border-radius: 2px;
    margin: 0;
    vertical-align: middle;
    border: 1px solid #bbb;
  }
</style>

<!-- Leyenda de escala -->
<div class="leyenda">
  <span class="leyenda-item">
    Datos diarios de 2024.
  </span>
</div>

<div bind:this={container} style="position: relative; background: #fff; max-width: 600px; margin: 0 auto;">
  <svg
    viewBox={`0 0 ${viewWidth} ${viewHeight}`}
    width={viewWidth}
    height={viewHeight}
    preserveAspectRatio="xMidYMid meet"
    style="width: 100%; height: auto; display: block; margin: 0 auto;"
  >
    {#each dias as dia, idx (idx)}
      {#if dia.getDate() === 1}
        <line
          x1={offx + Math.floor(idx / nFilas) * sep}
          y1={offy + (idx % nFilas) * sep - 1}
          x2={offx + Math.floor(idx / nFilas) * sep + anchoRect}
          y2={offy + (idx % nFilas) * sep - 1}
          stroke="#000"
          stroke-width="1.5"
        />
      {/if}
      <rect
        x={offx + Math.floor(idx / nFilas) * sep}
        y={offy + (idx % nFilas) * sep}
        width={anchoRect}
        height={altoRect}
        fill={getRectColor(toISO(dia))}
        stroke="#fff"
        stroke-width="1"
        on:mouseenter={() => {
          const fecha = toISO(dia);
          const d = detallesMap.get(fecha);
          tooltipVisible = true;
          tooltipContent = getTooltipContent(d, fecha);
          tooltipColor = getRectColor(fecha);
        }}
        on:mouseleave={() => {
          tooltipVisible = false;
        }}
        on:mousemove={(e) => {
          const bounds = container.getBoundingClientRect();
          const tooltipWidth = 240;
          const tooltipHeight = 100;
          const padding = 10;

          let relX = e.clientX - bounds.left + padding;
          let relY = e.clientY - bounds.top + padding;

          if (relX + tooltipWidth > bounds.width) {
            relX = bounds.width - tooltipWidth - padding;
          }
          if (relY + tooltipHeight > bounds.height) {
            relY = bounds.height - tooltipHeight - padding;
          }
          if (relX < padding) relX = padding;
          if (relY < padding) relY = padding;

          tooltipX = relX;
          tooltipY = relY;
        }}
      />
    {/each}
  </svg>
  {#if tooltipVisible}
    <div
      class="tooltip"
      style="
        left: {tooltipX}px;
        top: {tooltipY}px;
        border-color: {tooltipColor};
      "
    >{@html tooltipContent}</div>
  {/if}
</div>
