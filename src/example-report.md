---
title: Exploración de datos
---

<div class="hero">
  <h1> Análisis descriptivo </h1>
</div>



<!-- Selección del año -->
```js

const anioSeleccionado = view(Inputs.select(["2021", "2025"], {label: "Elige un año:"}));
anioSeleccionado;
const img2021 = FileAttachment("data/histogramas2021.png").url();
const img2025 = FileAttachment("data/histogramas2025.png").url();

```
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


<!-- FUNCIÓN Histogramas por año -->
```js

function panelDistribucionesPorAnio(data, yearSeleccionado, {
  diccionario = null,
  height = 260,
  binsCount = 16,
  maxCategorias = 10,
  columnasExcluir = [],
  marginTop = 28,
  marginRight = 18,
  marginBottom = 58,
  marginLeft = 48
} = {}) {
  const container = html`<div style="width: 100%;"></div>`;

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

  function render() {
    const width = container.getBoundingClientRect().width;
    if (!width) return;

    container.innerHTML = "";

    const yearNum = +yearSeleccionado;
    const datosAnio = data
      .map(d => ({...d, year: +d.year}))
      .filter(d => d.year === yearNum);

    if (!datosAnio.length) {
      d3.select(container)
        .append("div")
        .style("padding", "12px")
        .text(`No hay datos para el año ${yearSeleccionado}.`);
      return;
    }

    const columnas = Object.keys(datosAnio[0]).filter(c =>
      !columnasExcluir.includes(c) && c !== "year"
    );

    d3.select(container)
      .append("div")
      .style("font-weight", "600")
      .style("margin-bottom", "14px")
      .text(`Distribuciones para ${yearSeleccionado}`);

    const grid = d3.select(container)
      .append("div")
      .style("display", "grid")
      .style("grid-template-columns",
        width < 700 ? "1fr" :
        width < 1100 ? "1fr 1fr" :
        "1fr 1fr 1fr"
      )
      .style("gap", "16px")
      .style("width", "100%");

    columnas.forEach(variableKey => {
      const titulo = diccionario?.[variableKey] || variableKey;
      const valores = datosAnio.map(d => d[variableKey]);
      const tipo = detectarTipoVariable(valores);

      const card = grid.append("div")
        .style("width", "100%")
        .style("border", "1px solid #ddd")
        .style("border-radius", "10px")
        .style("padding", "10px")
        .style("box-sizing", "border-box");

      card.append("div")
        .style("font-size", "14px")
        .style("font-weight", "600")
        .style("margin-bottom", "8px")
        .text(titulo);

      const cardWidth = card.node().getBoundingClientRect().width;
      const svgWidth = Math.max(cardWidth - 2, 260);
      const innerWidth = svgWidth - marginLeft - marginRight;
      const innerHeight = height - marginTop - marginBottom;

      const svg = card.append("svg")
        .attr("width", svgWidth)
        .attr("height", height)
        .attr("viewBox", `0 0 ${svgWidth} ${height}`)
        .style("width", "100%")
        .style("height", "auto")
        .style("display", "block");

      if (tipo === "numerica") {
        const datosNum = valores
          .map(v => +v)
          .filter(v => !isNaN(v));

        if (!datosNum.length) {
          card.append("div")
            .style("padding", "8px")
            .text("Sin datos válidos.");
          return;
        }

        const xDomain = d3.extent(datosNum);
        const bins = d3.bin()
          .domain(xDomain)
          .thresholds(binsCount)(datosNum);

        const yMax = d3.max(bins, d => d.length) || 1;

        const x = d3.scaleLinear()
          .domain(xDomain)
          .nice()
          .range([marginLeft, marginLeft + innerWidth]);

        const y = d3.scaleLinear()
          .domain([0, yMax])
          .nice()
          .range([marginTop + innerHeight, marginTop]);

        svg.append("g")
          .selectAll("rect")
          .data(bins)
          .join("rect")
          .attr("x", d => x(d.x0) + 1)
          .attr("y", d => y(d.length))
          .attr("width", d => Math.max(0, x(d.x1) - x(d.x0) - 1))
          .attr("height", d => y(0) - y(d.length))
          .attr("fill", "currentColor")
          .attr("opacity", 0.75);

        svg.append("g")
          .attr("transform", `translate(0,${marginTop + innerHeight})`)
          .call(d3.axisBottom(x).ticks(5));

        svg.append("g")
          .attr("transform", `translate(${marginLeft},0)`)
          .call(d3.axisLeft(y).ticks(5));
      } else {
        const datosCat = valores
          .filter(valorValido)
          .map(v => String(v).trim());

        if (!datosCat.length) {
          card.append("div")
            .style("padding", "8px")
            .text("Sin datos válidos.");
          return;
        }

        const freq = d3.rollups(
          datosCat,
          v => v.length,
          d => d
        ).sort((a, b) => d3.descending(a[1], b[1]));

        const top = freq.slice(0, maxCategorias);
        const resto = freq.slice(maxCategorias);
        const datosGrafica = top.map(([categoria, valor]) => ({categoria, valor}));

        if (resto.length) {
          datosGrafica.push({
            categoria: "Otros",
            valor: d3.sum(resto, d => d[1])
          });
        }

        const x = d3.scaleBand()
          .domain(datosGrafica.map(d => d.categoria))
          .range([marginLeft, marginLeft + innerWidth])
          .padding(0.15);

        const y = d3.scaleLinear()
          .domain([0, d3.max(datosGrafica, d => d.valor) || 1])
          .nice()
          .range([marginTop + innerHeight, marginTop]);

        svg.append("g")
          .selectAll("rect")
          .data(datosGrafica)
          .join("rect")
          .attr("x", d => x(d.categoria))
          .attr("y", d => y(d.valor))
          .attr("width", x.bandwidth())
          .attr("height", d => y(0) - y(d.valor))
          .attr("fill", "currentColor")
          .attr("opacity", 0.75);

        svg.append("g")
          .attr("transform", `translate(0,${marginTop + innerHeight})`)
          .call(d3.axisBottom(x))
          .call(g => g.selectAll("text")
            .attr("transform", "rotate(-35)")
            .style("text-anchor", "end")
            .style("font-size", "10px"));

        svg.append("g")
          .attr("transform", `translate(${marginLeft},0)`)
          .call(d3.axisLeft(y).ticks(5));
      }
    });
  }

  const resizeObserver = new ResizeObserver(render);
  resizeObserver.observe(container);
  invalidation.then(() => resizeObserver.disconnect());

  render();
  return container;
}

```


```js
panelDistribucionesPorAnio(data, anioSeleccionado, {
  diccionario: data_dicc
})

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

        svg.append("g")
          .selectAll("rect")
          .data(info.bins)
          .join("rect")
          .attr("x", d => x(d.x0) + 1)
          .attr("y", d => y(d.length))
          .attr("width", d => Math.max(0, x(d.x1) - x(d.x0) - 1))
          .attr("height", d => y(0) - y(d.length))
          .attr("fill", "currentColor")
          .attr("opacity", 0.75);

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

        svg.append("g")
          .selectAll("rect")
          .data(info.datosGrafica)
          .join("rect")
          .attr("x", d => x(d.categoria))
          .attr("y", d => y(d.valor))
          .attr("width", x.bandwidth())
          .attr("height", d => y(0) - y(d.valor))
          .attr("fill", "currentColor")
          .attr("opacity", 0.75);

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

  function convertirAMatriz(attributeMap) {
    if (!attributeMap || typeof attributeMap !== "object") return [];

    const atributos = Object.keys(attributeMap);

    return atributos.flatMap(atributo => {
      const valoresColseg = attributeMap[atributo] || {};
      return colsegOrden.map(colseg => ({
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

    if (!variableData) {
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

    const hayNivelYear = tieneNivelYear(variableData);

    years.forEach(year => {
      let datosAnio;

      if (hayNivelYear) {
        datosAnio = extraerDatosPorAnio(variableData, year);
      } else {
        datosAnio = variableData;
      }

      const matriz = convertirAMatriz(datosAnio);

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

      const atributos = Array.from(new Set(matriz.map(d => d.atributo)));
      const cardWidth = card.node().getBoundingClientRect().width;
      const svgWidth = Math.max(cardWidth, 320);
      const innerWidth = svgWidth - marginLeft - marginRight;
      const innerHeight = height - marginTop - marginBottom;

      const x = d3.scaleBand()
        .domain(colsegOrden)
        .range([marginLeft, marginLeft + innerWidth])
        .padding(0.05);

      const y = d3.scaleBand()
        .domain(atributos)
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







