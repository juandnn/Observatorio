---
title: Exploración de datos
---


<style>
.observablehq {
  max-width: 200%;
}

.observablehq main {
  max-width: 100%;
}

.observablehq main p,
.observablehq main h1,
.observablehq main li,
.observablehq main blockquote,
.observablehq main table,
.observablehq main pre,
.observablehq main .card,
.observablehq main .hero {
  max-width: 100%;
}
</style>



<div class="hero">
  <h1> ¿Qué está pasando con la percepción de seguridad en Colombia durante los años 2021 y 2025? </h1>
  <p> ¿Qué está pasando con la percepción de seguridad en Colombia durante los años 2021 y 2025? ¿Qué está pasando con la percepción de seguridad en Colombia durante los años 2021 y 2025?  </p>
</div>





<!-- ```js

const mostrarHeatmaps = view(Inputs.select(["Sí", "No"], {label: "Elige si quieres ver los histogramas al mismo tiempo:"}));
mostrarHeatmaps;

``` -->

<!-- Selección del año -->
<!-- ```js

const anioSeleccionado = mostrarHeatmaps === "Sí"
  ? view(Inputs.select(["2021", "2025"], { label: "Elige un año:" }))
  : null;
const img2021 = FileAttachment("data/histogramas2021.png").url();
const img2025 = FileAttachment("data/histogramas2025.png").url();

``` -->

<!-- ```js


const ruta = anioSeleccionado === "2021"
  ? display(html`<img src="${img2021}" style="max-width: 100%;">`)
  : display(html`<img src="${img2025}" style="max-width: 100%;">`);

``` -->

<!-- Carga del dataset -->

```js
const data_dicc = await FileAttachment("data/diccionario_nombres.json").json();
const data = await FileAttachment("data/df_final.csv").csv();
const data_heatmap_relativo = await FileAttachment("/data/heatmap_relativo.json").json();
const data_heatmap_absoluto = await FileAttachment("/data/heatmap_absoluto.json").json();

console.log(data_heatmap_relativo)

```

<!-- Pedir variable -->

```js

const columnas_legibles = Object.keys(data[0]).map(c => ({
  key: c,
  label: data_dicc[c] || c
}));

const variableSeleccionada = view(
  Inputs.select(columnas_legibles, {
    label: "Selecciona variable:",
    format: d => d.label
  })
);

variableSeleccionada;
```

<!-- Pedir si relativo -->

```js

const esRelativo = view(Inputs.select(["Sí", "No"], {label: "Elige si el manejo de variables será relativo:"}));
esRelativo;

```

<!-- FUNCIÓN Histogramas individuales -->

```js
function distribucionPorAnio(data, variableKey, {
  years = [2021, 2025],
  height = 340,
  marginTop = 30,
  marginRight = 20,
  marginBottom = 100,
  marginLeft = 85,
  binsCount = 20,
  maxCategorias = 20,
  diccionario = data_dicc
} = {}) {
  const container = html`<div style="width: 100%;"></div>`;
  const yarrrPalette = [
  "#1f77b4",
  "#ff7f0e",
  "#2ca02c",
  "#d62728",
  "#9467bd",
  "#8c564b",
  "#e377c2",
  "#7f7f7f",
  "#bcbd22",
  "#17becf"
];

  function valorValido(v) {
    return v !== null && v !== undefined && v !== "" && v !== "NA" && v !== "NaN";
  }

  function categorizarEdad(age) {
    if (!valorValido(age)) return null;

    const edad = +age;
    if (isNaN(edad)) return null;
    if (edad < 29) return "Jóvenes";
    if (edad < 45) return "Adultez temprana";
    if (edad < 60) return "Adultez media";
    return "Adultos mayores";
  }

  function normalizarValorVariable(row) {
    if (variableKey === "q2") return categorizarEdad(row[variableKey]);
    return row[variableKey];
  }

  function esNumero(v) {
    return valorValido(v) && !isNaN(+v);
  }

  function detectarTipoVariable(rows) {
    const valores = rows
      .map(d => normalizarValorVariable(d))
      .filter(valorValido);

    if (!valores.length) return "categorica";

    const numericos = valores.filter(esNumero).length;
    const proporcionNumerica = numericos / valores.length;

    return proporcionNumerica > 0.9 ? "numerica" : "categorica";
  }

  function crearTooltip(card) {
    return card.append("div")
      .style("margin-top", "8px")
      .style("min-height", "20px")
      .style("font-size", "13px")
      .style("line-height", "1.3")
      .style("padding", "6px 8px")
      .style("border-radius", "6px")
      .style("background", "rgba(255,255,255,0.08)")
      .style("color", "white")
      .style("visibility", "hidden");
  }

  function activarInteraccionBarras(selection, tooltip, formatearTexto) {
    selection
      .style("cursor", "pointer")
      .on("click", function(event, d) {
        selection.attr("opacity", 0.75);
        d3.select(this).attr("opacity", 1);

        tooltip
          .style("visibility", "visible")
          .text(formatearTexto(d));
      });
  }

  function render() {
    const width = container.getBoundingClientRect().width;
    if (!width) return;

    container.innerHTML = "";

    const wrapper = d3.select(container)
      .append("div")
      .style("display", "grid")
      .style("grid-template-columns", width < 700 ? "1fr" : "1fr 1fr")
      .style("gap", "16px")
      .style("width", "100%");

    const datosBase = data
      .map(d => ({
        ...d,
        year: +d.year
      }))
      .filter(d => !isNaN(d.year));

    const tipo = detectarTipoVariable(datosBase);
    const tituloVariable = diccionario?.[variableKey] || variableKey;

    if (tipo === "numerica") {
      const cardBaseWidth = width < 700 ? width : (width - 16) / 2;
      const innerWidthAprox = Math.max(Math.max(cardBaseWidth, 300) - marginLeft - marginRight, 120);
      const binsAjustados = Math.max(6, Math.min(
        binsCount,
        Math.floor(innerWidthAprox / 32),
        Math.ceil(Math.sqrt(datosBase.length))
      ));

      const datosNumericos = datosBase
        .filter(d => valorValido(normalizarValorVariable(d)))
        .map(d => ({
          year: d.year,
          valorOriginal: normalizarValorVariable(d)
        }))
        .filter(d => esNumero(d.valorOriginal))
        .map(d => ({
          year: d.year,
          valor: +d.valorOriginal
        }));

      if (!datosNumericos.length) {
        d3.select(container)
          .append("div")
          .style("padding", "12px")
          .text(`No hay datos numéricos válidos para "${tituloVariable}".`);
        return;
      }

      const xDomain = d3.extent(datosNumericos, d => d.valor);

      const binGenerator = d3.bin()
        .domain(xDomain)
        .thresholds(binsAjustados);

      const binsPorAnio = years.map(year => {
        const subset = datosNumericos.filter(d => d.year === year).map(d => d.valor);
        return {
          year,
          bins: binGenerator(subset)
        };
      });

      const yMax = d3.max(binsPorAnio.flatMap(d => d.bins), b => b.length) || 1;
      const binKeys = binsPorAnio[0]?.bins.map((b, i) => `${b.x0}-${b.x1}-${i}`) || [];
      const colorInterpolator = d3.interpolateRgbBasis(yarrrPalette);
      const colorScale = d3.scaleOrdinal()
        .domain(binKeys)
        .range(binKeys.map((_, i) => {
          const t = binKeys.length <= 1 ? 0.5 : i / (binKeys.length - 1);
          return colorInterpolator(t);
        }));

      years.forEach(year => {
        const subset = datosNumericos.filter(d => d.year === year);
        const info = binsPorAnio.find(d => d.year === year);

        const card = wrapper.append("div")
          .style("width", "100%");

        card.append("div")
          .style("font-weight", "600")
          .style("margin-bottom", "8px")
          .text(`${tituloVariable} - ${year}`);

        if (!subset.length) {
          card.append("div")
            .style("padding", "12px")
            .text(`No hay datos para ${year}.`);
          return;
        }

        const cardWidth = card.node().getBoundingClientRect().width;
        const svgWidth = Math.max(cardWidth, 300);
        const innerWidth = svgWidth - marginLeft - marginRight;
        const innerHeight = height - marginTop - marginBottom;

        const x = d3.scaleLinear()
          .domain(xDomain)
          .nice()
          .range([marginLeft, marginLeft + innerWidth]);

        const y = d3.scaleLinear()
          .domain([0, yMax])
          .nice()
          .range([marginTop + innerHeight, marginTop]);

        const svg = card.append("svg")
          .attr("width", svgWidth)
          .attr("height", height)
          .attr("viewBox", `0 0 ${svgWidth} ${height}`)
          .style("width", "100%")
          .style("height", "auto")
          .style("display", "block");

        const tooltip = crearTooltip(card);

        const barras = svg.append("g")
          .selectAll("rect")
          .data(info.bins)
          .join("rect")
          .attr("x", d => x(d.x0) + 2)
          .attr("y", d => y(d.length))
          .attr("width", d => Math.max(6, x(d.x1) - x(d.x0) - 4))
          .attr("height", d => y(0) - y(d.length))
          .attr("fill", (d, i) => colorScale(`${d.x0}-${d.x1}-${i}`))
          .attr("opacity", 0.9);

        activarInteraccionBarras(
          barras,
          tooltip,
          d => `Rango: ${d.x0} a ${d.x1} | Frecuencia exacta: ${d.length}`
        );

        svg.append("g")
          .attr("transform", `translate(0,${marginTop + innerHeight})`)
          .call(d3.axisBottom(x).ticks(Math.min(8, binsAjustados)));

        svg.append("g")
          .attr("transform", `translate(${marginLeft},0)`)
          .call(d3.axisLeft(y).ticks(6));

        svg.append("text")
          .attr("x", marginLeft + innerWidth / 2)
          .attr("y", height - 10)
          .attr("text-anchor", "middle")
          .attr("font-size", 12)
          .attr("fill", "white")
          .text(tituloVariable);

        svg.append("text")
          .attr("transform", "rotate(-90)")
          .attr("x", -(marginTop + innerHeight / 2))
          .attr("y", 16)
          .attr("text-anchor", "middle")
          .attr("font-size", 12)
          .attr("fill", "white")
          .text("Frecuencia");
      });

    } else {
      const datosCategoricos = datosBase
        .map(d => ({
          year: d.year,
          valor: normalizarValorVariable(d)
        }))
        .filter(d => valorValido(d.valor))
        .map(d => ({
          year: d.year,
          valor: String(d.valor).trim()
        }));

      if (!datosCategoricos.length) {
        d3.select(container)
          .append("div")
          .style("padding", "12px")
          .text(`No hay datos categóricos válidos para "${tituloVariable}".`);
        return;
      }

      const frecuenciaGlobal = d3.rollups(
        datosCategoricos,
        v => v.length,
        d => d.valor
      )
      .sort((a, b) => d3.descending(a[1], b[1]));

      const categoriasTop = frecuenciaGlobal
        .slice(0, maxCategorias)
        .map(d => d[0]);

      const categoriasFinales = [...categoriasTop];
      const incluirOtros = frecuenciaGlobal.length > maxCategorias;
      if (incluirOtros) categoriasFinales.push("Otros");
      const categoriasOrdenEdad = ["Jóvenes", "Adultez temprana", "Adultez media", "Adultos mayores"];
      const categoriasDominio = variableKey === "q2"
        ? categoriasOrdenEdad.filter(cat => categoriasFinales.includes(cat))
        : categoriasFinales;
      if (incluirOtros && !categoriasDominio.includes("Otros")) categoriasDominio.push("Otros");

      const colorScale = d3.scaleOrdinal()
        .domain(categoriasDominio)
        .range(categoriasDominio.map((_, i) => yarrrPalette[i % yarrrPalette.length]));

      const datosPorAnio = years.map(year => {
        const subset = datosCategoricos.filter(d => d.year === year);

        const conteo = d3.rollup(
          subset,
          v => v.length,
          d => categoriasTop.includes(d.valor) ? d.valor : (incluirOtros ? "Otros" : d.valor)
        );

        const datosGrafica = categoriasDominio.map(cat => ({
          categoria: cat,
          valor: conteo.get(cat) || 0
        }));

        return { year, datosGrafica };
      });

      const yMax = d3.max(datosPorAnio.flatMap(d => d.datosGrafica), d => d.valor) || 1;

      years.forEach(year => {
        const info = datosPorAnio.find(d => d.year === year);

        const card = wrapper.append("div")
          .style("width", "100%");

        card.append("div")
          .style("font-weight", "600")
          .style("margin-bottom", "8px")
          .text(`${tituloVariable} - ${year}`);

        if (!info.datosGrafica.some(d => d.valor > 0)) {
          card.append("div")
            .style("padding", "12px")
            .text(`No hay datos para ${year}.`);
          return;
        }

        const cardWidth = card.node().getBoundingClientRect().width;
        const svgWidth = Math.max(cardWidth, 300);
        const innerWidth = svgWidth - marginLeft - marginRight;
        const innerHeight = height - marginTop - marginBottom;

        const x = d3.scaleBand()
          .domain(categoriasDominio)
          .range([marginLeft, marginLeft + innerWidth])
          .padding(0.15);

        const y = d3.scaleLinear()
          .domain([0, yMax])
          .nice()
          .range([marginTop + innerHeight, marginTop]);

        const svg = card.append("svg")
          .attr("width", svgWidth)
          .attr("height", height)
          .attr("viewBox", `0 0 ${svgWidth} ${height}`)
          .style("width", "100%")
          .style("height", "auto")
          .style("display", "block");

        const tooltip = crearTooltip(card);

        const barras = svg.append("g")
          .selectAll("rect")
          .data(info.datosGrafica)
          .join("rect")
          .attr("x", d => x(d.categoria))
          .attr("y", d => y(d.valor))
          .attr("width", x.bandwidth())
          .attr("height", d => y(0) - y(d.valor))
          .attr("fill", d => colorScale(d.categoria))
          .attr("opacity", 0.75);

        activarInteraccionBarras(
          barras,
          tooltip,
          d => `Categoría: ${d.categoria} | Frecuencia exacta: ${d.valor}`
        );

        svg.append("g")
          .attr("transform", `translate(0,${marginTop + innerHeight})`)
          .call(d3.axisBottom(x))
          .call(g => g.selectAll("text")
            .attr("transform", "rotate(-35)")
            .style("text-anchor", "end"));

        svg.append("g")
          .attr("transform", `translate(${marginLeft},0)`)
          .call(d3.axisLeft(y).ticks(6));

        svg.append("text")
          .attr("x", marginLeft + innerWidth / 2)
          .attr("y", height - 5)
          .attr("text-anchor", "middle")
          .attr("font-size", 12)
          .attr("fill", "white")
          .text(tituloVariable);

        svg.append("text")
          .attr("transform", "rotate(-90)")
          .attr("x", -(marginTop + innerHeight / 2))
          .attr("y", 16)
          .attr("text-anchor", "middle")
          .attr("font-size", 12)
          .attr("fill", "white")
          .text("Frecuencia");
      });
    }
  }

  const resizeObserver = new ResizeObserver(() => render());
  resizeObserver.observe(container);
  invalidation.then(() => resizeObserver.disconnect());

  render();
  return container;
}
```

```js
distribucionPorAnio(data, variableSeleccionada.key)
```

# Heatmap 
<!-- FUNCIÓN Heatmap por variables compartiendo año -->

```js

function heatmapsColsegPorAnio({
  variableKey,
  esRelativo,
  dataHeatmapRelativo,
  dataHeatmapAbsoluto,
  diccionario = null,
  years = [2021, 2025],
  height = 420,
  marginTop = 40,
  marginRight = 20,
  marginBottom = 90,
  marginLeft = 140
} = {}) {
  const container = html`<div style="width: 100%; position: relative;"></div>`;
  const selectedDatumByYear = new Map();
  let lastWidth = null;

  const tituloVariable = diccionario?.[variableKey] || variableKey;
  const colsegOrden = ["Peor", "Igual", "Mejor"];
  const paletasColseg = {
    Peor: ["#fee5d9", "#cb181d"],
    Igual: ["#fff7bc", "#d9a404"],
    Mejor: ["#e5f5e0", "#238b45"]
  };

  function normalizarYearKey(y) {
    return [String(y), `${y}.0`, +y];
  }

  function extraerDatosPorAnio(variableData, year) {
    if (!variableData) return null;

    const posiblesKeys = normalizarYearKey(year);

    for (const key of posiblesKeys) {
      if (variableData[key] != null) return variableData[key];
    }

    return null;
  }

  function tieneNivelYear(variableData) {
    if (!variableData || typeof variableData !== "object") return false;
    const keys = Object.keys(variableData);
    return keys.some(k => ["2021", "2021.0", "2025", "2025.0"].includes(String(k)));
  }

  function valorValido(v) {
    return v !== null && v !== undefined && v !== "" && v !== "NA" && v !== "NaN";
  }

  function construirMapaColsegConColseg(year, modo = "absoluto") {
    if (typeof data === "undefined" || !Array.isArray(data)) return null;

    const filasAnio = data.filter(d => +d.year === +year && valorValido(d.colseg));
    if (!filasAnio.length) return null;

    const conteos = d3.rollup(
      filasAnio,
      v => v.length,
      d => String(d.colseg).trim()
    );

    return Object.fromEntries(
      colsegOrden.map(categoriaFila => [
        categoriaFila,
        Object.fromEntries(
          colsegOrden.map(categoriaColumna => {
            const conteo = categoriaFila === categoriaColumna
              ? (conteos.get(categoriaFila) || 0)
              : 0;
            const totalFila = conteos.get(categoriaFila) || 0;

            return [
              categoriaColumna,
              modo === "relativo" ? (totalFila ? conteo / totalFila : 0) : conteo
            ];
          })
        )
      ])
    );
  }

  function obtenerDatosAnio(variableData, year, hayNivelYear, modo = "absoluto") {
    if (variableKey === "colseg") {
      const matrizColseg = construirMapaColsegConColseg(year, modo);
      if (matrizColseg) return matrizColseg;
    }

    if (hayNivelYear) return extraerDatosPorAnio(variableData, year);
    return variableData;
  }

  function construirOrdenAtributos(datosPorAnio) {
    const orden = [];
    const vistos = new Set();

    datosPorAnio.forEach(({ attributeMap }) => {
      if (!attributeMap || typeof attributeMap !== "object") return;

      Object.keys(attributeMap).forEach(atributo => {
        if (!vistos.has(atributo)) {
          vistos.add(atributo);
          orden.push(atributo);
        }
      });
    });

    return orden;
  }

  function obtenerOrdenColumnas(datosPorAnio) {
    const columnasDetectadas = [];
    const vistas = new Set();

    datosPorAnio.forEach(({ attributeMap }) => {
      if (!attributeMap || typeof attributeMap !== "object") return;

      Object.values(attributeMap).forEach(valoresColseg => {
        if (!valoresColseg || typeof valoresColseg !== "object") return;

        Object.keys(valoresColseg).forEach(columna => {
          if (!vistas.has(columna)) {
            vistas.add(columna);
            columnasDetectadas.push(columna);
          }
        });
      });
    });

    const columnasPrioritarias = colsegOrden.filter(col => vistas.has(col));
    const columnasRestantes = columnasDetectadas.filter(col => !colsegOrden.includes(col));
    return [...columnasPrioritarias, ...columnasRestantes];
  }

  function convertirAMatriz({
    year,
    displayMap,
    absoluteMap,
    relativeMap,
    atributos,
    columnas
  }) {
    if (!displayMap || typeof displayMap !== "object" || !atributos.length || !columnas.length) {
      return [];
    }

    return atributos.flatMap(atributo => {
      const valoresDisplay = displayMap[atributo] || {};
      const valoresAbsolutos = absoluteMap?.[atributo] || {};
      const valoresRelativos = relativeMap?.[atributo] || {};
      const totalAtributo = d3.sum(columnas, colseg => +valoresAbsolutos[colseg] || 0);

      return columnas.map(colseg => {
        const totalAtributoColseg = +valoresAbsolutos[colseg] || 0;
        const valorRelativo = valoresRelativos[colseg] != null
          ? (+valoresRelativos[colseg] || 0)
          : (totalAtributo ? totalAtributoColseg / totalAtributo : 0);

        return {
          year,
          atributo,
          colseg,
          valor: +valoresDisplay[colseg] || 0,
          valorAbsoluto: totalAtributoColseg,
          valorRelativo,
          totalAtributo
        };
      });
    });
  }

  function formatValue(v) {
    return esRelativo === "Sí" ? d3.format(".2f")(v) : d3.format(",")(v);
  }

  function formatAbsolute(v) {
    return d3.format(",")(v);
  }

  function formatRelative(v) {
    return d3.format(".1%")(v);
  }

  function crearEscalaColor(colseg, valorMax) {
    const rango = paletasColseg[colseg] || ["#edf2f7", "#4a5568"];
    return d3.scaleLinear()
      .domain([0, Math.max(valorMax, 1e-9)])
      .range(rango)
      .interpolate(d3.interpolateRgb);
  }

  function colorTextoCelda(fill) {
    const color = d3.color(fill);
    if (!color) return "black";

    const luminancia = (0.299 * color.r + 0.587 * color.g + 0.114 * color.b) / 255;
    return luminancia > 0.62 ? "black" : "white";
  }

  function crearTooltipFlotante(parent) {
    return d3.select(parent)
      .append("div")
      .style("position", "absolute")
      .style("z-index", "10")
      .style("pointer-events", "none")
      .style("opacity", "0")
      .style("visibility", "hidden")
      .style("background", "rgba(20,20,20,0.92)")
      .style("color", "white")
      .style("padding", "8px 10px")
      .style("border-radius", "6px")
      .style("font-size", "12px")
      .style("line-height", "1.35")
      .style("box-shadow", "0 6px 18px rgba(0,0,0,0.25)")
      .style("max-width", "240px");
  }

  function moverTooltip(tooltip, event) {
    const [x, y] = d3.pointer(event, container);
    tooltip
      .style("left", `${x + 14}px`)
      .style("top", `${y + 14}px`);
  }

  function crearTarjetaDetalle(parent) {
    return parent.append("div")
      .style("margin-top", "8px")
      .style("min-height", "20px")
      .style("font-size", "13px")
      .style("line-height", "1.35")
      .style("padding", "8px 10px")
      .style("border-radius", "6px")
      .style("background", "rgba(255,255,255,0.08)")
      .style("color", "white");
  }

  function renderInfoBox(selection) {
    if (!selection) {
      return `
        <div>Haz click en un cuadro para ver el detalle.</div>
      `;
    }

    return `
      <div><strong>Año:</strong> ${selection.year}</div>
      <div><strong>Atributo:</strong> ${selection.atributo}</div>
      <div><strong>Categoría colseg:</strong> ${selection.colseg}</div>
      <div><strong>Valor mostrado en el heatmap:</strong> ${formatValue(selection.valor)}</div>
      <div style="margin-top:6px;"><strong>Detalle</strong></div>
      <div>Total de personas que votaron por ese atributo: ${formatAbsolute(selection.totalAtributo)}</div>
      <div>Total de personas que votaron por ese atributo y esa categoría de colseg: ${formatAbsolute(selection.valorAbsoluto)}</div>
      <div>Total de personas relativas que votaron: ${formatRelative(selection.valorRelativo)}</div>
    `;
  }

  function renderTooltip(selection) {
    return `
      <div><strong>${selection.atributo}</strong></div>
      <div>${selection.colseg}: ${formatValue(selection.valor)}</div>
      <div>Haz click para ver más información.</div>
    `;
  }

  function esSeleccionado(d, selection) {
    if (!selection) return false;

    return (
      d.year === selection.year &&
      d.atributo === selection.atributo &&
      d.colseg === selection.colseg
    );
  }

  function render() {
    const width = container.getBoundingClientRect().width;
    if (!width) return;

    if (lastWidth !== null && width === lastWidth) return;
    lastWidth = width;

    container.innerHTML = "";
    const tooltip = crearTooltipFlotante(container);

    const variableDataAbsoluto = dataHeatmapAbsoluto?.[variableKey];
    const variableDataRelativo = dataHeatmapRelativo?.[variableKey];

    if (!variableDataAbsoluto && !variableDataRelativo && variableKey !== "colseg") {
      d3.select(container)
        .append("div")
        .style("padding", "12px")
        .text(`No se encontró información para la variable "${tituloVariable}".`);
      return;
    }

    const wrapper = d3.select(container)
      .append("div")
      .style("display", "grid")
      .style("grid-template-columns", width < 900 ? "1fr" : "1fr 1fr")
      .style("gap", "16px")
      .style("width", "100%");

    const hayNivelYearAbsoluto = variableKey === "colseg" ? false : tieneNivelYear(variableDataAbsoluto);
    const hayNivelYearRelativo = variableKey === "colseg" ? false : tieneNivelYear(variableDataRelativo);
    const datosPorAnio = years.map(year => ({
      year,
      displayMap: esRelativo === "Sí"
        ? obtenerDatosAnio(variableDataRelativo, year, hayNivelYearRelativo, "relativo")
        : obtenerDatosAnio(variableDataAbsoluto, year, hayNivelYearAbsoluto, "absoluto"),
      absoluteMap: obtenerDatosAnio(variableDataAbsoluto, year, hayNivelYearAbsoluto, "absoluto"),
      relativeMap: obtenerDatosAnio(variableDataRelativo, year, hayNivelYearRelativo, "relativo")
    }));
    const atributosCompartidos = construirOrdenAtributos(
      datosPorAnio.map(d => ({ attributeMap: d.displayMap || d.absoluteMap || d.relativeMap }))
    );
    const columnasCompartidas = obtenerOrdenColumnas(
      datosPorAnio.map(d => ({ attributeMap: d.displayMap || d.absoluteMap || d.relativeMap }))
    );

    if (!atributosCompartidos.length || !columnasCompartidas.length) {
      d3.select(container)
        .append("div")
        .style("padding", "12px")
        .text(`No hay datos para mostrar en "${tituloVariable}".`);
      return;
    }

    const matricesPorAnio = datosPorAnio.map(({ year, displayMap, absoluteMap, relativeMap }) => ({
      year,
      matriz: convertirAMatriz({
        year,
        displayMap,
        absoluteMap,
        relativeMap,
        atributos: atributosCompartidos,
        columnas: columnasCompartidas
      })
    }));

    const cards = [];

    matricesPorAnio.forEach(({ year, matriz }) => {

      const card = wrapper.append("div")
        .style("width", "100%")
        .style("box-sizing", "border-box");

      card.append("div")
        .style("font-weight", "600")
        .style("margin-bottom", "8px")
        .text(`${tituloVariable} - ${year}`);

      if (!matriz.length) {
        card.append("div")
          .style("padding", "12px")
          .text(`No hay datos para ${year}.`);
        return;
      }

      const cardWidth = card.node().getBoundingClientRect().width;
      const svgWidth = Math.max(cardWidth, 320);
      const innerWidth = svgWidth - marginLeft - marginRight;
      const innerHeight = height - marginTop - marginBottom;

      const x = d3.scaleBand()
        .domain(columnasCompartidas)
        .range([marginLeft, marginLeft + innerWidth])
        .padding(0.05);

      const y = d3.scaleBand()
        .domain(atributosCompartidos)
        .range([marginTop, marginTop + innerHeight])
        .padding(0.05);

      const valores = matriz.map(d => d.valor);
      const valorMaxCompartido = esRelativo === "Sí"
        ? 1
        : (d3.max(valores) || 1);

      const escalasColor = new Map(
        columnasCompartidas.map(colseg => [colseg, crearEscalaColor(colseg, valorMaxCompartido)])
      );

      const svg = card.append("svg")
        .attr("width", svgWidth)
        .attr("height", height)
        .attr("viewBox", `0 0 ${svgWidth} ${height}`)
        .style("width", "100%")
        .style("height", "auto")
        .style("display", "block");

      const cells = svg.append("g")
        .selectAll("rect")
        .data(matriz)
        .join("rect")
        .attr("x", d => x(d.colseg))
        .attr("y", d => y(d.atributo))
        .attr("width", x.bandwidth())
        .attr("height", y.bandwidth())
        .attr("rx", 3)
        .attr("ry", 3)
        .attr("fill", d => escalasColor.get(d.colseg)?.(d.valor) || "#cccccc")
        .attr("stroke", d => {
          const selection = selectedDatumByYear.get(year);
          return esSeleccionado(d, selection) ? "white" : "rgba(255,255,255,0.18)";
        })
        .attr("stroke-width", d => {
          const selection = selectedDatumByYear.get(year);
          return esSeleccionado(d, selection) ? 2.5 : 0.8;
        })
        .style("cursor", "pointer");

      svg.append("g")
        .selectAll("text.cell")
        .data(matriz)
        .join("text")
        .attr("class", "cell")
        .attr("x", d => x(d.colseg) + x.bandwidth() / 2)
        .attr("y", d => y(d.atributo) + y.bandwidth() / 2)
        .attr("text-anchor", "middle")
        .attr("dominant-baseline", "middle")
        .attr("font-size", 11)
        .attr("fill", d => colorTextoCelda(escalasColor.get(d.colseg)?.(d.valor)))
        .text(d => {
          if (esRelativo === "Sí") return d3.format(".2f")(d.valor);
          return d3.format(",")(d.valor);
        });

      svg.append("g")
        .attr("transform", `translate(0,${marginTop + innerHeight})`)
        .call(d3.axisBottom(x))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll("line").remove());

      svg.append("g")
        .attr("transform", `translate(${marginLeft},0)`)
        .call(d3.axisLeft(y))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll("line").remove());

      svg.append("text")
        .attr("x", marginLeft + innerWidth / 2)
        .attr("y", height - 12)
        .attr("text-anchor", "middle")
        .attr("font-size", 12)
        .attr("fill", "white")
        .text("Situación de seguridad");

      const defs = svg.append("defs");
      const legendStartX = Math.max(marginLeft, svgWidth - marginRight - 210);
      const legendY = 8;
      const legendBarWidth = 48;
      const legendBarHeight = 8;
      const legendGap = 68;
      const stops = d3.range(0, 1.01, 0.1);

      columnasCompartidas
        .filter(colseg => colsegOrden.includes(colseg))
        .forEach((colseg, index) => {
          const gradientId = `legend-gradient-${variableKey}-${year}-${colseg}-${esRelativo}`;
          const gradient = defs.append("linearGradient")
            .attr("id", gradientId)
            .attr("x1", "0%")
            .attr("x2", "100%")
            .attr("y1", "0%")
            .attr("y2", "0%");

          gradient.selectAll("stop")
            .data(stops)
            .join("stop")
            .attr("offset", d => `${d * 100}%`)
            .attr("stop-color", d => escalasColor.get(colseg)?.(d * valorMaxCompartido));

          const legendX = legendStartX + index * legendGap;

          svg.append("text")
            .attr("x", legendX)
            .attr("y", legendY + 1)
            .attr("font-size", "10px")
            .attr("fill", "white")
            .text(colseg);

          svg.append("rect")
            .attr("x", legendX)
            .attr("y", legendY + 6)
            .attr("width", legendBarWidth)
            .attr("height", legendBarHeight)
            .attr("rx", 3)
            .attr("fill", `url(#${gradientId})`);

          svg.append("text")
            .attr("x", legendX)
            .attr("y", legendY + 26)
            .attr("font-size", "9px")
            .attr("fill", "white")
            .text("0");

          svg.append("text")
            .attr("x", legendX + legendBarWidth)
            .attr("y", legendY + 26)
            .attr("text-anchor", "end")
            .attr("font-size", "9px")
            .attr("fill", "white")
            .text(esRelativo === "Sí"
              ? d3.format(".2f")(valorMaxCompartido)
              : d3.format(",")(valorMaxCompartido));
        });

      const infoBox = crearTarjetaDetalle(card)
        .html(renderInfoBox(selectedDatumByYear.get(year) || null));

      cards.push({ year, cells, infoBox });
    });

    function actualizarSeleccionVisual() {
      cards.forEach(({ year, cells, infoBox }) => {
        const selection = selectedDatumByYear.get(year) || null;

        cells
          .attr("stroke", d => esSeleccionado(d, selection) ? "white" : "rgba(255,255,255,0.18)")
          .attr("stroke-width", d => esSeleccionado(d, selection) ? 2.5 : 0.8)
          .attr("opacity", d => {
            if (!selection) return 1;
            return esSeleccionado(d, selection) ? 1 : 0.55;
          });

        infoBox.html(renderInfoBox(selection));
      });
    }

    cards.forEach(({ year, cells }) => {
      cells
        .on("mouseenter", function(event, d) {
          tooltip
            .style("visibility", "visible")
            .style("opacity", "1")
            .html(renderTooltip(d));

          moverTooltip(tooltip, event);
        })
        .on("mousemove", function(event) {
          moverTooltip(tooltip, event);
        })
        .on("mouseleave", function() {
          tooltip
            .style("opacity", "0")
            .style("visibility", "hidden");
        })
        .on("click", function(event, d) {
          event.preventDefault();
          event.stopPropagation();
          selectedDatumByYear.set(year, d);
          actualizarSeleccionVisual();
        });
    });

    actualizarSeleccionVisual();
  }

  const resizeTarget = container.parentElement || container;
  const resizeObserver = new ResizeObserver(entries => {
    const newWidth = entries[0]?.contentRect?.width;
    if (!newWidth) return;

    if (lastWidth === null || newWidth !== lastWidth) {
      lastWidth = null;
      render();
    }
  });

  resizeObserver.observe(resizeTarget);

  if (typeof invalidation !== "undefined") {
    invalidation.then(() => resizeObserver.disconnect());
  }

  render();
  return container;
}

```


```js

heatmapsColsegPorAnio({
  variableKey: variableSeleccionada.key,
  esRelativo,
  dataHeatmapRelativo: data_heatmap_relativo,
  dataHeatmapAbsoluto: data_heatmap_absoluto,
  diccionario: data_dicc
})

```


<!-- Filtro año -->

```js

const anioSeleccionado = view(Inputs.select(["2021", "2025"], { label: "Elige un año:" }));
anioSeleccionado;
const geojson = await FileAttachment("data/co.json").json();

```

<!-- Controlador mapa  -->


```js
function controlMapaGeograficoInput(data, variableSeleccionada, anioSeleccionado) {
  function valorValido(v) {
    return v !== null && v !== undefined && v !== "" && v !== "NA" && v !== "NaN";
  }

  function esNumero(v) {
    return valorValido(v) && !isNaN(+v);
  }

  function detectarTipoVariable(valores) {
    const validos = valores.filter(valorValido);
    if (!validos.length) return "categorica";
    const nNumericos = validos.filter(esNumero).length;
    return (nNumericos / validos.length) > 0.9 ? "numerica" : "categorica";
  }

  const yearNum = +anioSeleccionado;

  const datosAnio = data
    .map(d => ({ ...d, year: +d.year }))
    .filter(d => d.year === yearNum);

  const variableKey = variableSeleccionada?.key;
  const valores = datosAnio.map(d => d[variableKey]);
  const tipo = detectarTipoVariable(valores);

  if (tipo === "numerica") {
    return Inputs.select(
      ["Promedio", "Mínimo", "Máximo"],
      {
        label: "Agrupar variable numérica por:",
        value: "Promedio"
      }
    );
  }

  const atributos = Array.from(
    new Set(
      valores
        .filter(valorValido)
        .map(v => String(v).trim())
    )
  ).sort(d3.ascending);

  return Inputs.select(
    atributos,
    {
      label: "Selecciona atributo:",
      value: atributos[0] ?? null
    }
  );
}
```

```js

const selectorMapa = view(
  controlMapaGeograficoInput(data, variableSeleccionada, anioSeleccionado)
);
```

<!-- Mostrar Mapa -->

```js
let escalaPorcentualModoMapa = "Auto";

function mapaColombiaRegionesHeatmap({
  data,
  anioSeleccionado,
  variableSeleccionada,
  selectorMapa,
  esRelativo = "Sí",
  diccionario = null,
  width = 850,
  height = 950,
  stroke = "#ffffff",
  strokeWidth = 1,
  mostrarNombres = false,
  colorInterpolator = d3.interpolateBlues,
  colorSinDatos = "#d9d9d9"
} = {}) {
  const container = html`<div style="width:100%;"></div>`;

  const agrupacion = {
    "COAMA": "Amazonía",
    "COANT": "Central",
    "COARA": "Oriental",
    "COATL": "Atlántica",
    "COBOL": "Atlántica",
    "COBOY": "Oriental",
    "COCAL": "Central",
    "COCAQ": "Amazonía",
    "COCAS": "Oriental",
    "COCAU": "Pacífica",
    "COCES": "Atlántica",
    "COCHO": "Pacífica",
    "COCOR": "Atlántica",
    "COCUN": "Central",
    "CODC": "Bogotá",
    "COGUA": "Amazonía",
    "COGUV": "Amazonía",
    "COHUI": "Central",
    "COLAG": "Atlántica",
    "COMAG": "Atlántica",
    "COMET": "Oriental",
    "CONAR": "Pacífica",
    "CONSA": "Oriental",
    "COPUT": "Amazonía",
    "COQUI": "Central",
    "CORIS": "Central",
    "COSAN": "Oriental",
    "COSAP": "Atlántica",
    "COSUC": "Atlántica",
    "COTOL": "Central",
    "COVAC": "Pacífica",
    "COVAU": "Amazonía",
    "COVID": "Oriental"
  };

  let geojsonCache = null;

  function valorValido(v) {
    return v !== null && v !== undefined && v !== "" && v !== "NA" && v !== "NaN";
  }

  function esNumero(v) {
    return valorValido(v) && !isNaN(+v);
  }

  function detectarTipoVariable(valores) {
    const validos = valores.filter(valorValido);
    if (!validos.length) return "categorica";
    const nNumericos = validos.filter(esNumero).length;
    return (nNumericos / validos.length) > 0.9 ? "numerica" : "categorica";
  }

  function normalizarTexto(s) {
    return String(s ?? "")
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .trim()
      .toLowerCase();
  }

  function normalizarRegion(region) {
    const r = normalizarTexto(region);
    if (r === "oriental") return "Oriental";
    if (r === "central") return "Central";
    if (r === "amazonia") return "Amazonía";
    if (r === "atlantica") return "Atlántica";
    if (r === "pacifica") return "Pacífica";
    if (r === "bogota") return "Bogotá";
    return null;
  }

  function formatearValor(v, tipoVariable) {
    if (v === null || v === undefined || isNaN(v)) return "Sin datos";
    return tipoVariable === "categorica"
      ? d3.format(".1%")(v)
      : d3.format(".2f")(v);
  }

  function obtenerDatosBase() {
    const yearNum = +anioSeleccionado;
    const variableKey = variableSeleccionada?.key;
    const variableLabel = variableSeleccionada?.label || diccionario?.[variableKey] || variableKey;

    const datosAnio = data
      .map(d => ({ ...d, year: +d.year }))
      .filter(d => d.year === yearNum);

    const valoresVariable = datosAnio.map(d => d[variableKey]);
    const tipoVariable = detectarTipoVariable(valoresVariable);

    return {
      yearNum,
      variableKey,
      variableLabel,
      datosAnio,
      tipoVariable
    };
  }

  function construirResumenPorRegion(ctx) {
    const { yearNum, variableKey, variableLabel, datosAnio, tipoVariable } = ctx;

    if (tipoVariable === "categorica") {
      const atributoSeleccionado = selectorMapa;

      const datosValidos = datosAnio.filter(d =>
        normalizarRegion(d.estratopri) &&
        valorValido(d[variableKey])
      );

      const totalNacionalAtributo = datosValidos.filter(d =>
        String(d[variableKey]).trim() === String(atributoSeleccionado).trim()
      ).length;

      const regiones = Array.from(new Set(
        datosValidos
          .map(d => normalizarRegion(d.estratopri))
          .filter(Boolean)
      ));

      return Object.fromEntries(
        regiones.map(region => {
          const filasRegion = datosValidos.filter(d => normalizarRegion(d.estratopri) === region);
          const nRegion = filasRegion.length;

          const nAtributoRegion = filasRegion.filter(d =>
            String(d[variableKey]).trim() === String(atributoSeleccionado).trim()
          ).length;

          let valor = null;
          let denominador = null;

          if (esRelativo === "Sí") {
            denominador = nRegion;
            valor = denominador > 0 ? nAtributoRegion / denominador : null;
          } else {
            denominador = totalNacionalAtributo;
            valor = denominador > 0 ? nAtributoRegion / denominador : null;
          }

          return [region, {
            region,
            year: yearNum,
            variableKey,
            variableLabel,
            tipoVariable,
            atributo: atributoSeleccionado,
            valor,
            numerador: nAtributoRegion,
            denominador,
            totalRegion: nRegion,
            totalNacionalAtributo
          }];
        })
      );
    }

    const agregacionSeleccionada = selectorMapa;

    const datosValidos = datosAnio
      .filter(d => normalizarRegion(d.estratopri) && esNumero(d[variableKey]))
      .map(d => ({
        region: normalizarRegion(d.estratopri),
        valor: +d[variableKey]
      }));

    const regiones = Array.from(new Set(datosValidos.map(d => d.region)));

    return Object.fromEntries(
      regiones.map(region => {
        const valores = datosValidos
          .filter(d => d.region === region)
          .map(d => d.valor);

        let valor = null;

        if (valores.length) {
          if (agregacionSeleccionada === "Promedio") valor = d3.mean(valores);
          else if (agregacionSeleccionada === "Mínimo") valor = d3.min(valores);
          else if (agregacionSeleccionada === "Máximo") valor = d3.max(valores);
        }

        return [region, {
          region,
          year: yearNum,
          variableKey,
          variableLabel,
          tipoVariable,
          agregacion: agregacionSeleccionada,
          valor,
          cantidad: valores.length
        }];
      })
    );
  }

  function obtenerValoresMapa(resumenPorRegion) {
    return Object.values(resumenPorRegion)
      .map(d => d.valor)
      .filter(v => v !== null && v !== undefined && !isNaN(v));
  }

  function construirEscalaColor(tipoVariable, valoresMapa) {
    const minValor = d3.min(valoresMapa);
    const maxValor = d3.max(valoresMapa);

    let domainMin = 0;
    let domainMax = 1;
    let color = () => colorSinDatos;

    if (valoresMapa.length === 0 || minValor === undefined || maxValor === undefined) {
      return { color, domainMin, domainMax, minValor, maxValor };
    }

    if (tipoVariable === "categorica" && escalaPorcentualModoMapa === "0–100%") {
      color = d3.scaleSequential(colorInterpolator).domain([0, 1]);
      domainMin = 0;
      domainMax = 1;
      return { color, domainMin, domainMax, minValor, maxValor };
    }

    if (minValor === maxValor) {
      color = () => colorInterpolator(0.65);
      domainMin = minValor;
      domainMax = maxValor;
      return { color, domainMin, domainMax, minValor, maxValor };
    }

    color = d3.scaleSequential(colorInterpolator).domain([minValor, maxValor]);
    domainMin = minValor;
    domainMax = maxValor;

    return { color, domainMin, domainMax, minValor, maxValor };
  }

  function construirTooltipHTML(info, depto, region, ctx) {
    const { yearNum, variableLabel, tipoVariable } = ctx;

    let htmlTooltip = `
      <div><strong>${depto}</strong></div>
      <div><strong>Región:</strong> ${region}</div>
      <div><strong>Año:</strong> ${yearNum}</div>
      <div><strong>Variable:</strong> ${variableLabel}</div>
    `;

    if (!info || info.valor === null || info.valor === undefined || isNaN(info.valor)) {
      htmlTooltip += `<div><strong>Valor:</strong> Sin datos</div>`;
      return htmlTooltip;
    }

    if (tipoVariable === "categorica") {
      htmlTooltip += `
        <div><strong>Atributo:</strong> ${info.atributo}</div>
        <div><strong>Valor:</strong> ${formatearValor(info.valor, tipoVariable)}</div>
        <div><strong>Numerador:</strong> ${d3.format(",")(info.numerador)}</div>
        <div><strong>Denominador:</strong> ${d3.format(",")(info.denominador)}</div>
        <div><strong>Total región:</strong> ${d3.format(",")(info.totalRegion)}</div>
      `;
    } else {
      htmlTooltip += `
        <div><strong>Agrupación:</strong> ${info.agregacion}</div>
        <div><strong>Valor:</strong> ${formatearValor(info.valor, tipoVariable)}</div>
        <div><strong>Observaciones válidas:</strong> ${d3.format(",")(info.cantidad)}</div>
      `;
    }

    return htmlTooltip;
  }

  function dibujarLeyenda(svg, ctx, escalaInfo, valoresMapa) {
    const { variableKey, variableLabel, yearNum, tipoVariable } = ctx;
    const { domainMin, domainMax, minValor, maxValor } = escalaInfo;

    const legendWidth = 220;
    const legendHeight = 12;
    const legendX = 24;
    const legendY = 24;

    svg.append("text")
      .attr("x", legendX)
      .attr("y", legendY - 8)
      .style("font-size", "12px")
      .style("font-weight", "600")
      .style("fill", "white")
      .text(`${variableLabel} - ${selectorMapa}`);

    if (valoresMapa.length > 0) {
      const defs = svg.append("defs");
      const gradientId = `legend-gradient-mapa-${variableKey}-${yearNum}-${Math.random().toString(36).slice(2)}`;

      const gradient = defs.append("linearGradient")
        .attr("id", gradientId)
        .attr("x1", "0%")
        .attr("x2", "100%")
        .attr("y1", "0%")
        .attr("y2", "0%");

      d3.range(0, 1.01, 0.1).forEach(t => {
        gradient.append("stop")
          .attr("offset", `${t * 100}%`)
          .attr("stop-color", colorInterpolator(t));
      });

      svg.append("rect")
        .attr("x", legendX)
        .attr("y", legendY)
        .attr("width", legendWidth)
        .attr("height", legendHeight)
        .attr("fill", domainMin === domainMax ? colorInterpolator(0.65) : `url(#${gradientId})`)
        .attr("stroke", "white")
        .attr("stroke-width", 0.5);

      const legendScale = d3.scaleLinear()
        .domain([domainMin, domainMax])
        .range([legendX, legendX + legendWidth]);

      const tickFormat = tipoVariable === "categorica"
        ? d3.format(".0%")
        : d3.format(".2f");

      const legendAxis = d3.axisBottom(legendScale)
        .ticks(4)
        .tickFormat(tickFormat);

      svg.append("g")
        .attr("transform", `translate(0, ${legendY + legendHeight})`)
        .call(legendAxis)
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll("line").remove())
        .call(g => g.selectAll("text").style("font-size", "10px"));
    }

    svg.append("g")
      .attr("transform", `translate(24, ${legendY + 42})`)
      .call(g => {
        g.append("rect")
          .attr("width", 14)
          .attr("height", 14)
          .attr("fill", colorSinDatos)
          .attr("stroke", "white")
          .attr("stroke-width", 0.5);

        g.append("text")
          .attr("x", 20)
          .attr("y", 11)
          .style("font-size", "11px")
          .style("fill", "white")
          .text("Sin datos");
      });
  }

  function dibujarMapa(svg, geojson, ctx, resumenPorRegion, escalaInfo, tooltip) {
    const { color } = escalaInfo;

    const projection = d3.geoMercator();
    const path = d3.geoPath(projection);

    projection.fitExtent(
      [[20, 20], [width - 20, height - 20]],
      geojson
    );

    svg.append("g")
      .selectAll("path")
      .data(geojson.features)
      .join("path")
      .attr("d", path)
      .attr("fill", d => {
        const region = agrupacion[d.properties.id];
        const info = resumenPorRegion[region];
        if (!info || info.valor === null || info.valor === undefined || isNaN(info.valor)) {
          return colorSinDatos;
        }
        return color(info.valor);
      })
      .attr("stroke", stroke)
      .attr("stroke-width", strokeWidth)
      .on("mouseover", function (event, d) {
        const codigo = d.properties.id;
        const depto = d.properties.name;
        const region = agrupacion[codigo] || "Sin región";
        const info = resumenPorRegion[region];

        d3.select(this).attr("opacity", 0.85);

        tooltip
          .html(construirTooltipHTML(info, depto, region, ctx))
          .style("left", `${event.clientX + 12}px`)
          .style("top", `${event.clientY + 12}px`)
          .style("opacity", 1);
      })
      .on("mousemove", function (event) {
          tooltip
            .style("left", `${event.clientX + 12}px`)
            .style("top", `${event.clientY + 12}px`);
        })
      .on("mouseout", function () {
        d3.select(this).attr("opacity", 1);
        tooltip.style("opacity", 0);
      });

    if (mostrarNombres) {
      svg.append("g")
        .selectAll("text")
        .data(geojson.features)
        .join("text")
        .attr("transform", d => {
          const [x, y] = path.centroid(d);
          return `translate(${x},${y})`;
        })
        .attr("text-anchor", "middle")
        .attr("dominant-baseline", "middle")
        .style("font-size", "8px")
        .style("pointer-events", "none")
        .style("fill", "black")
        .text(d => d.properties.name);
    }
  }

  function render() {
    container.innerHTML = "";

    const ctx = obtenerDatosBase();
    const resumenPorRegion = construirResumenPorRegion(ctx);
    const valoresMapa = obtenerValoresMapa(resumenPorRegion);
    const escalaInfo = construirEscalaColor(ctx.tipoVariable, valoresMapa);

    if (ctx.tipoVariable === "categorica") {
      const controls = d3.select(container)
        .append("div")
        .style("display", "flex")
        .style("justify-content", "flex-start")
        .style("align-items", "center")
        .style("margin-bottom", "10px")
        .style("gap", "8px");

      controls.append("label")
        .style("font-size", "12px")
        .style("font-weight", "600")
        .text("Escala de color:");

      const select = controls.append("select")
        .style("font-family", "inherit")
        .style("font-size", "inherit")
        .style("line-height", "inherit")
        .style("padding", "4px 8px")
        .style("border-radius", "6px")
        .style("background", "var(--theme-background-alt, #222)")
        .style("color", "currentColor")
        .style("border", "1px solid #666");

      select.selectAll("option")
        .data(["Auto", "0–100%"])
        .join("option")
        .attr("value", d => d)
        .property("selected", d => d === escalaPorcentualModoMapa)
        .text(d => d);

      select.on("change", function () {
        escalaPorcentualModoMapa = this.value;
        render();
      });
    }

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .attr("viewBox", [0, 0, width, height]);

    const tooltip = d3.select(container)
      .append("div")
      .style("position", "fixed")
      .style("left", "0px")
      .style("top", "0px")
      .style("background", "white")
      .style("border", "1px solid #ccc")
      .style("border-radius", "6px")
      .style("padding", "8px 10px")
      .style("font", "12px sans-serif")
      .style("line-height", "1.4")
      .style("pointer-events", "none")
      .style("opacity", 0)
      .style("z-index", 9999)
      .style("color", "#111")
      .style("box-shadow", "0 2px 10px rgba(0,0,0,0.15)");

    dibujarMapa(svg, geojsonCache, ctx, resumenPorRegion, escalaInfo, tooltip);
    dibujarLeyenda(svg, ctx, escalaInfo, valoresMapa);

    container.appendChild(svg.node());
  }

  return (async () => {
    geojsonCache = await FileAttachment("data/co.json").json();
    render();
    return container;
  })();
}

```


```js
mapaColombiaRegionesHeatmap({
  data,
  anioSeleccionado,
  variableSeleccionada,
  selectorMapa,
  esRelativo,
  diccionario: data_dicc,
  width: 850,
  height: 950,
  mostrarNombres: false
})

```


<!-- Variable influencia -->

```js
const categoriasInfluencia = view(
  Inputs.checkbox(["Peor", "Igual", "Mejor"], {
    label: "¿Qué categorías quieres mostrar?",
    value: ["Peor", "Igual", "Mejor"]
  })
);

categoriasInfluencia;

```


```js
function topAtributosInfluyentes({
  dataHeatmapRelativo,
  dataHeatmapAbsoluto,
  esRelativo,
  categoriasInfluencia,
  diccionario = null,
  years = [2021, 2025],
  topN = 10,
  height = 420,
  marginTop = 30,
  marginRight = 20,
  marginBottom = 110,
  marginLeft = 70
} = {}) {
  const container = html`<div style="width: 100%;"></div>`;
  const selectedDatumByYear = new Map();
  let lastWidth = null;
  const todasLasCategorias = ["Peor", "Igual", "Mejor"];
  const coloresCategorias = {
    Peor: "#d73027",
    Igual: "#f2c94c",
    Mejor: "#1a9850"
  };

  function normalizarYearKeys(y) {
    return [String(y), `${y}.0`, +y];
  }

  function extraerDatosPorAnio(variableData, year) {
    if (!variableData || typeof variableData !== "object") return null;

    for (const key of normalizarYearKeys(year)) {
      if (variableData[key] != null) return variableData[key];
    }
    return null;
  }

  function tieneNivelYear(variableData) {
    if (!variableData || typeof variableData !== "object") return false;

    return Object.keys(variableData).some(k =>
      ["2021", "2021.0", "2025", "2025.0"].includes(String(k))
    );
  }

  function obtenerCategoriasActivas() {
    const seleccion = Array.isArray(categoriasInfluencia) ? categoriasInfluencia : [];
    return seleccion.length ? seleccion : todasLasCategorias;
  }

  function construirSeries(year, categoriasActivas) {
    const resultados = [];
    const dataHeatmap = esRelativo === "Sí"
      ? dataHeatmapRelativo
      : dataHeatmapAbsoluto;

    for (const variableKey of Object.keys(dataHeatmap || {})) {
      const variableData = dataHeatmap[variableKey];
      const usaYear = tieneNivelYear(variableData);
      const bloque = usaYear ? extraerDatosPorAnio(variableData, year) : variableData;

      if (!bloque) continue;

      for (const atributo of Object.keys(bloque)) {
        const objColseg = bloque[atributo];
        const detalle = {
          Peor: +objColseg?.["Peor"] || 0,
          Igual: +objColseg?.["Igual"] || 0,
          Mejor: +objColseg?.["Mejor"] || 0
        };

        categoriasActivas.forEach(categoria => {
          resultados.push({
            year,
            categoria,
            variableKey,
            variableLabel: diccionario?.[variableKey] || variableKey,
            atributo,
            valor: detalle[categoria] || 0,
            detalle
          });
        });
      }
    }

    return categoriasActivas.map(categoria => {
      const puntos = resultados
        .filter(d => d.categoria === categoria)
        .sort((a, b) => d3.ascending(a.valor, b.valor))
        .slice(-topN)
        .map((d, i) => ({
          ...d,
          orden: i + 1,
          pointKey: `${year}__${categoria}__${d.variableKey}__${d.atributo}__${i}`
        }));

      return { categoria, puntos };
    });
  }

  function formatValue(v) {
    return esRelativo === "Sí" ? d3.format(".3f")(v) : d3.format(",")(v);
  }

  function crearTarjetaDetalle(parent) {
    return parent.append("div")
      .style("margin-top", "8px")
      .style("min-height", "20px")
      .style("font-size", "13px")
      .style("line-height", "1.35")
      .style("padding", "8px 10px")
      .style("border-radius", "6px")
      .style("background", "rgba(255,255,255,0.08)")
      .style("color", "white");
  }

  function renderInfoBox(selection) {
    if (!selection) {
      return `
        <div>Haz click en un punto para ver el detalle.</div>
      `;
    }

    return `
      <div><strong>Año:</strong> ${selection.year}</div>
      <div><strong>Categoría:</strong> ${selection.categoria}</div>
      <div><strong>Variable:</strong> ${selection.variableLabel}</div>
      <div><strong>Atributo:</strong> ${selection.atributo}</div>
      <div><strong>${selection.categoria}:</strong> ${formatValue(selection.valor)}</div>
      <div style="margin-top:6px;"><strong>Detalle</strong></div>
      <div>Peor: ${formatValue(selection.detalle.Peor)}</div>
      <div>Igual: ${formatValue(selection.detalle.Igual)}</div>
      <div>Mejor: ${formatValue(selection.detalle.Mejor)}</div>
    `;
  }

  function esSeleccionado(d, selection) {
    if (!selection) return false;

    return (
      d.year === selection.year &&
      d.categoria === selection.categoria &&
      d.variableKey === selection.variableKey &&
      d.atributo === selection.atributo
    );
  }

  function render() {
    const width = container.getBoundingClientRect().width;
    if (!width) return;

    if (lastWidth !== null && width === lastWidth) return;
    lastWidth = width;

    container.innerHTML = "";
    const categoriasActivas = obtenerCategoriasActivas();

    const wrapper = d3.select(container)
      .append("div")
      .style("display", "grid")
      .style("grid-template-columns", width < 900 ? "1fr" : "1fr 1fr")
      .style("gap", "16px")
      .style("width", "100%")
      .style("margin-bottom", "14px");

    const cards = [];
    const seriesPorAnio = years.map(year => ({
      year,
      series: construirSeries(year, categoriasActivas)
    }));

    const yMaxCompartido = esRelativo === "Sí"
      ? 1
      : d3.max(
        seriesPorAnio.flatMap(({ series }) => series.flatMap(({ puntos }) => puntos)),
        d => d.valor
      ) || 1;

    const maxPuntos = d3.max(
      seriesPorAnio.flatMap(({ series }) => series.map(({ puntos }) => puntos.length))
    ) || topN;

    seriesPorAnio.forEach(({ year, series }) => {

      const card = wrapper.append("div")
        .style("width", "100%");

      card.append("div")
        .style("font-weight", "600")
        .style("margin-bottom", "8px")
        .text(`Top ${topN} por categoría - ${year}`);

      if (!series.some(({ puntos }) => puntos.length)) {
        card.append("div")
          .style("padding", "12px")
          .text(`No hay datos para ${year}.`);
        return;
      }

      const cardWidth =
        card.node().getBoundingClientRect().width ||
        (width < 900 ? width : width / 2);

      const svgWidth = Math.max(cardWidth, 320);
      const innerWidth = svgWidth - marginLeft - marginRight;
      const innerHeight = height - marginTop - marginBottom;

      const x = d3.scalePoint()
        .domain(d3.range(1, maxPuntos + 1))
        .range([marginLeft, marginLeft + innerWidth])
        .padding(0.5);

      const y = d3.scaleLinear()
        .domain([0, yMaxCompartido])
        .nice()
        .range([marginTop + innerHeight, marginTop]);

      const svg = card.append("svg")
        .attr("width", svgWidth)
        .attr("height", height)
        .attr("viewBox", `0 0 ${svgWidth} ${height}`)
        .style("width", "100%")
        .style("height", "auto")
        .style("display", "block");

      const line = d3.line()
        .x(d => x(d.orden))
        .y(d => y(d.valor));

      const lines = svg.append("g")
        .selectAll("path")
        .data(series.filter(({ puntos }) => puntos.length))
        .join("path")
        .attr("fill", "none")
        .attr("stroke", d => coloresCategorias[d.categoria])
        .attr("stroke-width", 2.5)
        .attr("opacity", d => {
          const selection = selectedDatumByYear.get(year);
          if (!selection) return 0.9;
          return selection.categoria === d.categoria ? 1 : 0.35;
        })
        .attr("d", d => line(d.puntos));

      const points = svg.append("g")
        .selectAll("circle")
        .data(series.flatMap(({ puntos }) => puntos))
        .join("circle")
        .attr("cx", d => x(d.orden))
        .attr("cy", d => y(d.valor))
        .attr("r", d => {
          const selection = selectedDatumByYear.get(year);
          return esSeleccionado(d, selection) ? 6.5 : 4.5;
        })
        .attr("fill", d => coloresCategorias[d.categoria])
        .attr("stroke", "white")
        .attr("stroke-width", d => {
          const selection = selectedDatumByYear.get(year);
          return esSeleccionado(d, selection) ? 2 : 1;
        })
        .attr("opacity", d => {
          const selection = selectedDatumByYear.get(year);
          if (!selection) return 0.95;
          return esSeleccionado(d, selection) ? 1 : 0.45;
        })
        .style("cursor", "pointer")
        .style("pointer-events", "all");

      svg.append("g")
        .attr("transform", `translate(0,${marginTop + innerHeight})`)
        .call(
          d3.axisBottom(x)
            .tickFormat(d => d)
        )
        .call(g => g.selectAll("text")
          .style("font-size", "10px"));

      svg.append("g")
        .attr("transform", `translate(${marginLeft},0)`)
        .call(
          d3.axisLeft(y)
            .ticks(6)
            .tickFormat(esRelativo === "Sí" ? d3.format(".2f") : d3.format(","))
        );

      svg.append("text")
        .attr("transform", "rotate(-90)")
        .attr("x", -(marginTop + innerHeight / 2))
        .attr("y", 16)
        .attr("text-anchor", "middle")
        .attr("font-size", 12)
        .attr("fill", "white")
        .text("Influencia");

      svg.append("text")
        .attr("x", marginLeft + innerWidth / 2)
        .attr("y", height - 12)
        .attr("text-anchor", "middle")
        .attr("font-size", 12)
        .attr("fill", "white")
        .text("Posición en el orden ascendente");

      const legend = svg.append("g")
        .attr("transform", `translate(${marginLeft},${marginTop - 10})`);

      categoriasActivas.forEach((categoria, index) => {
        const legendItem = legend.append("g")
          .attr("transform", `translate(${index * 82},0)`);

        legendItem.append("line")
          .attr("x1", 0)
          .attr("x2", 18)
          .attr("y1", 0)
          .attr("y2", 0)
          .attr("stroke", coloresCategorias[categoria])
          .attr("stroke-width", 2.5);

        legendItem.append("circle")
          .attr("cx", 9)
          .attr("cy", 0)
          .attr("r", 4)
          .attr("fill", coloresCategorias[categoria])
          .attr("stroke", "white")
          .attr("stroke-width", 1);

        legendItem.append("text")
          .attr("x", 24)
          .attr("y", 4)
          .attr("font-size", 11)
          .attr("fill", "white")
          .text(categoria);
      });

      const infoBox = crearTarjetaDetalle(card)
        .html(renderInfoBox(selectedDatumByYear.get(year) || null));

      cards.push({ year, lines, points, infoBox });
    });

    function actualizarSeleccionVisual() {
      cards.forEach(({ year, lines, points, infoBox }) => {
        const selection = selectedDatumByYear.get(year) || null;
        infoBox.html(renderInfoBox(selection));

        lines.attr("opacity", d => {
          if (!selection) return 0.9;
          return selection.categoria === d.categoria ? 1 : 0.35;
        });

        points
          .attr("r", d => esSeleccionado(d, selection) ? 6.5 : 4.5)
          .attr("stroke-width", d => esSeleccionado(d, selection) ? 2 : 1)
          .attr("opacity", d => {
            if (!selection) return 0.95;
            return esSeleccionado(d, selection) ? 1 : 0.45;
          });
      });
    }

    cards.forEach(({ year, points }) => {
      points.on("click", function(event, d) {
        event.preventDefault();
        event.stopPropagation();
        selectedDatumByYear.set(year, d);
        actualizarSeleccionVisual();
      });
    });
  }

  const resizeTarget = container.parentElement || container;

  const resizeObserver = new ResizeObserver(entries => {
    const newWidth = entries[0]?.contentRect?.width;
    if (!newWidth) return;

    if (lastWidth === null || newWidth !== lastWidth) {
      lastWidth = null;
      render();
    }
  });

  resizeObserver.observe(resizeTarget);

  if (typeof invalidation !== "undefined") {
    invalidation.then(() => resizeObserver.disconnect());
  }

  render();
  return container;
}

```
```js
topAtributosInfluyentes({
  dataHeatmapRelativo: data_heatmap_relativo,
  dataHeatmapAbsoluto: data_heatmap_absoluto,
  esRelativo,
  categoriasInfluencia,
  diccionario: data_dicc
})

```
