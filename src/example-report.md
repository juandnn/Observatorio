---
title: Exploración de datos
---

<div class="hero">
  <h1> Análisis exploratorio </h1>
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

  function valorValido(v) {
    return v !== null && v !== undefined && v !== "" && v !== "NA" && v !== "NaN";
  }

  function esNumero(v) {
    return valorValido(v) && !isNaN(+v);
  }

  function detectarTipoVariable(rows) {
    const valores = rows
      .map(d => d[variableKey])
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
      const datosNumericos = datosBase
        .map(d => ({
          year: d.year,
          valor: +d[variableKey]
        }))
        .filter(d => !isNaN(d.valor));

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
        .thresholds(binsCount);

      const binsPorAnio = years.map(year => {
        const subset = datosNumericos.filter(d => d.year === year).map(d => d.valor);
        return {
          year,
          bins: binGenerator(subset)
        };
      });

      const yMax = d3.max(binsPorAnio.flatMap(d => d.bins), b => b.length) || 1;

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
          .attr("x", d => x(d.x0) + 1)
          .attr("y", d => y(d.length))
          .attr("width", d => Math.max(0, x(d.x1) - x(d.x0) - 1))
          .attr("height", d => y(0) - y(d.length))
          .attr("fill", "currentColor")
          .attr("opacity", 0.75);

        activarInteraccionBarras(
          barras,
          tooltip,
          d => `Rango: ${d.x0} a ${d.x1} | Frecuencia exacta: ${d.length}`
        );

        svg.append("g")
          .attr("transform", `translate(0,${marginTop + innerHeight})`)
          .call(d3.axisBottom(x).ticks(Math.min(8, binsCount)));

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
          valor: d[variableKey]
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

      const datosPorAnio = years.map(year => {
        const subset = datosCategoricos.filter(d => d.year === year);

        const conteo = d3.rollup(
          subset,
          v => v.length,
          d => categoriasTop.includes(d.valor) ? d.valor : (incluirOtros ? "Otros" : d.valor)
        );

        const datosGrafica = categoriasFinales.map(cat => ({
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
          .domain(categoriasFinales)
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
          .attr("fill", "currentColor")
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
  const container = html`<div style="width: 100%;"></div>`;

  const dataHeatmap = esRelativo === "Sí"
    ? dataHeatmapRelativo
    : dataHeatmapAbsoluto;

  const tituloVariable = diccionario?.[variableKey] || variableKey;
  const colsegOrden = ["Peor", "Igual", "Mejor"];

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

  function construirMapaColsegConColseg(year) {
    if (typeof data === "undefined" || !Array.isArray(data)) return null;

    const filasAnio = data.filter(d => +d.year === +year && valorValido(d.colseg));
    if (!filasAnio.length) return null;

    const total = filasAnio.length;
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

            return [
              categoriaColumna,
              esRelativo === "Sí" ? (total ? conteo / total : 0) : conteo
            ];
          })
        )
      ])
    );
  }

  function obtenerDatosAnio(variableData, year, hayNivelYear) {
    if (variableKey === "colseg") {
      const matrizColseg = construirMapaColsegConColseg(year);
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

  function convertirAMatriz(attributeMap, atributos, columnas) {
    if (!attributeMap || typeof attributeMap !== "object" || !atributos.length || !columnas.length) {
      return [];
    }

    return atributos.flatMap(atributo => {
      const valoresColseg = attributeMap[atributo] || {};
      return columnas.map(colseg => ({
        atributo,
        colseg,
        valor: +valoresColseg[colseg] || 0
      }));
    });
  }

  function render() {
    const width = container.getBoundingClientRect().width;
    if (!width) return;

    container.innerHTML = "";

    const variableData = dataHeatmap?.[variableKey];

    if (!variableData && variableKey !== "colseg") {
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

    const hayNivelYear = variableKey === "colseg" ? false : tieneNivelYear(variableData);
    const datosPorAnio = years.map(year => ({
      year,
      attributeMap: obtenerDatosAnio(variableData, year, hayNivelYear)
    }));
    const atributosCompartidos = construirOrdenAtributos(datosPorAnio);
    const columnasCompartidas = obtenerOrdenColumnas(datosPorAnio);

    if (!atributosCompartidos.length || !columnasCompartidas.length) {
      d3.select(container)
        .append("div")
        .style("padding", "12px")
        .text(`No hay datos para mostrar en "${tituloVariable}".`);
      return;
    }

    const matricesPorAnio = datosPorAnio.map(({ year, attributeMap }) => ({
      year,
      matriz: convertirAMatriz(attributeMap, atributosCompartidos, columnasCompartidas)
    }));

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
      const valorMax = d3.max(valores) || 1;

      const color = esRelativo === "Sí"
        ? d3.scaleSequential(d3.interpolateBlues).domain([0, valorMax])
        : d3.scaleSequential(d3.interpolateBlues).domain([0, valorMax]);

      const svg = card.append("svg")
        .attr("width", svgWidth)
        .attr("height", height)
        .attr("viewBox", `0 0 ${svgWidth} ${height}`)
        .style("width", "100%")
        .style("height", "auto")
        .style("display", "block");

      svg.append("g")
        .selectAll("rect")
        .data(matriz)
        .join("rect")
        .attr("x", d => x(d.colseg))
        .attr("y", d => y(d.atributo))
        .attr("width", x.bandwidth())
        .attr("height", y.bandwidth())
        .attr("rx", 3)
        .attr("ry", 3)
        .attr("fill", d => color(d.valor));

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
        .attr("fill", d => d.valor > valorMax * 0.55 ? "white" : "black")
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

      const legendWidth = Math.min(180, innerWidth * 0.5);
      const legendHeight = 10;
      const legendX = marginLeft + innerWidth - legendWidth;
      const legendY = 12;

      const defs = svg.append("defs");
      const gradientId = `legend-gradient-${variableKey}-${year}-${esRelativo}`;

      const gradient = defs.append("linearGradient")
        .attr("id", gradientId)
        .attr("x1", "0%")
        .attr("x2", "100%")
        .attr("y1", "0%")
        .attr("y2", "0%");

      const stops = d3.range(0, 1.01, 0.1);
      gradient.selectAll("stop")
        .data(stops)
        .join("stop")
        .attr("offset", d => `${d * 100}%`)
        .attr("stop-color", d => color(d * valorMax));

      svg.append("rect")
        .attr("x", legendX)
        .attr("y", legendY)
        .attr("width", legendWidth)
        .attr("height", legendHeight)
        .attr("fill", `url(#${gradientId})`);

      const legendScale = d3.scaleLinear()
        .domain([0, valorMax])
        .range([legendX, legendX + legendWidth]);

      const legendAxis = d3.axisBottom(legendScale)
        .ticks(4)
        .tickFormat(esRelativo === "Sí" ? d3.format(".2f") : d3.format(","));

      svg.append("g")
        .attr("transform", `translate(0,${legendY + legendHeight})`)
        .call(legendAxis)
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll("line").remove())
        .call(g => g.selectAll("text").style("font-size", "10px"));
    });
  }

  const resizeObserver = new ResizeObserver(() => render());
  resizeObserver.observe(container);
  invalidation.then(() => resizeObserver.disconnect());

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
  let escalaPorcentualModo = "Auto";

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

    if (tipoVariable === "categorica" && escalaPorcentualModo === "0–100%") {
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
      .text(`${variableLabel} — ${selectorMapa}`);

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
        .style("font-size", "12px")
        .style("padding", "4px 8px")
        .style("border-radius", "6px")
        .style("background", "var(--theme-background-alt, #222)")
        .style("color", "currentColor")
        .style("border", "1px solid #666");

      select.selectAll("option")
        .data(["Auto", "0–100%"])
        .join("option")
        .attr("value", d => d)
        .property("selected", d => d === escalaPorcentualModo)
        .text(d => d);

      select.on("change", function () {
        escalaPorcentualModo = this.value;
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
const tipoInfluencia = view(
  Inputs.select(["Peor", "Igual", "Mejor"], {
    label: "¿Qué define influencia?"
  })
);

tipoInfluencia;

```


```js
function topAtributosInfluyentes({
  dataHeatmapRelativo,
  dataHeatmapAbsoluto,
  esRelativo,
  tipoInfluencia,
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

  const dataHeatmap = esRelativo === "Sí"
    ? dataHeatmapRelativo
    : dataHeatmapAbsoluto;

  const selectedDatumByYear = new Map();
  let lastWidth = null;

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

  function calcularInfluencia(objColseg) {
    if (!objColseg) return 0;

    const peor = +objColseg["Peor"] || 0;
    const igual = +objColseg["Igual"] || 0;
    const mejor = +objColseg["Mejor"] || 0;

    if (tipoInfluencia === "Peor") return peor;
    if (tipoInfluencia === "Igual") return igual;
    return mejor;
  }

  function construirRanking(year) {
    const resultados = [];

    for (const variableKey of Object.keys(dataHeatmap || {})) {
      const variableData = dataHeatmap[variableKey];
      const usaYear = tieneNivelYear(variableData);
      const bloque = usaYear ? extraerDatosPorAnio(variableData, year) : variableData;

      if (!bloque) continue;

      for (const atributo of Object.keys(bloque)) {
        const objColseg = bloque[atributo];
        const valor = calcularInfluencia(objColseg);

        resultados.push({
          year,
          variableKey,
          variableLabel: diccionario?.[variableKey] || variableKey,
          atributo,
          valor,
          detalle: {
            Peor: +objColseg?.["Peor"] || 0,
            Igual: +objColseg?.["Igual"] || 0,
            Mejor: +objColseg?.["Mejor"] || 0
          }
        });
      }
    }

    return resultados
      .sort((a, b) => d3.descending(a.valor, b.valor))
      .slice(0, topN)
      .map((d, i) => ({
        ...d,
        xKey: `${d.variableKey}__${d.atributo}__${i}__${year}`
      }));
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
        <div>Haz click en una barra para ver el detalle.</div>
      `;
    }

    return `
      <div><strong>Año:</strong> ${selection.year}</div>
      <div><strong>Variable:</strong> ${selection.variableLabel}</div>
      <div><strong>Atributo:</strong> ${selection.atributo}</div>
      <div><strong>${tipoInfluencia}:</strong> ${formatValue(selection.valor)}</div>
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

    const wrapper = d3.select(container)
      .append("div")
      .style("display", "grid")
      .style("grid-template-columns", width < 900 ? "1fr" : "1fr 1fr")
      .style("gap", "16px")
      .style("width", "100%")
      .style("margin-bottom", "14px");

    const cards = [];
    const rankingPorAnio = years.map(year => ({
      year,
      ranking: construirRanking(year)
    }));

    const yMaxCompartido = esRelativo === "Sí"
      ? 1
      : d3.max(rankingPorAnio.flatMap(({ ranking }) => ranking), d => d.valor) || 1;

    rankingPorAnio.forEach(({ year, ranking }) => {

      const card = wrapper.append("div")
        .style("width", "100%");

      card.append("div")
        .style("font-weight", "600")
        .style("margin-bottom", "8px")
        .text(`Top ${topN} (${tipoInfluencia}) — ${year}`);

      if (!ranking.length) {
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

      const x = d3.scaleBand()
        .domain(ranking.map(d => d.xKey))
        .range([marginLeft, marginLeft + innerWidth])
        .padding(0.15);

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

      const bars = svg.append("g")
        .selectAll("rect")
        .data(ranking)
        .join("rect")
        .attr("x", d => x(d.xKey))
        .attr("y", d => y(d.valor))
        .attr("width", x.bandwidth())
        .attr("height", d => Math.max(0, y(0) - y(d.valor)))
        .attr("fill", "currentColor")
        .attr("opacity", d => {
          const selection = selectedDatumByYear.get(year);
          if (!selection) return 0.8;
          return esSeleccionado(d, selection) ? 1 : 0.45;
        })
        .style("cursor", "pointer")
        .style("pointer-events", "all");

      svg.append("g")
        .attr("transform", `translate(0,${marginTop + innerHeight})`)
        .call(
          d3.axisBottom(x)
            .tickFormat(key => {
              const datum = ranking.find(d => d.xKey === key);
              return datum ? datum.variableLabel : key;
            })
        )
        .call(g => g.selectAll("text")
          .attr("transform", "rotate(-35)")
          .style("text-anchor", "end")
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
        .text(tipoInfluencia);

      const infoBox = crearTarjetaDetalle(card)
        .html(renderInfoBox(selectedDatumByYear.get(year) || null));

      cards.push({ year, bars, infoBox });
    });

    function actualizarSeleccionVisual() {
      cards.forEach(({ year, bars, infoBox }) => {
        const selection = selectedDatumByYear.get(year) || null;
        infoBox.html(renderInfoBox(selection));
        bars.attr("opacity", d => {
          if (!selection) return 0.8;
          return esSeleccionado(d, selection) ? 1 : 0.45;
        });
      });
    }

    cards.forEach(({ year, bars }) => {
      bars.on("click", function(event, d) {
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
  tipoInfluencia,
  diccionario: data_dicc
})

```
