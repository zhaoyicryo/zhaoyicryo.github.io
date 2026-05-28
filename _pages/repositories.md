---
layout: page
permalink: /repositories/
title: Data
description: All data and code related to my publications can be accessed through the Data and Code Availability statements within the respective papers, or by contacting me directly.
nav: true
nav_order: 4
---

### **Permafrost hydro-thermal dataset of Tibetan Plateau (PHD-TP)**

This dataset provides key hydrological and thermal variables of the active layer and near-surface permafrost across the Tibetan Plateau at 10 km spatial resolution and daily temporal resolution from 1979 to 2018. Use the selectors below to switch variable, layer, and year, then click download for the currently displayed GeoTIFF.

<div style="display:flex; flex-wrap:wrap; gap:10px; align-items:flex-end; margin: 0.8rem 0 0.9rem 0;">
  <label style="display:flex; flex-direction:column; font-size:0.92rem; color:#333;">
    Variable
    <select id="qtp-var-select" style="min-width:220px; padding:6px 8px;">
      <option value="st">Soil temperature</option>
      <option value="liq">Unfrozen water content</option>
      <option value="sme">Soil moisture equivalent</option>
      <option value="ice">Ice content</option>
    </select>
  </label>
  <label style="display:flex; flex-direction:column; font-size:0.92rem; color:#333;">
    Layer
    <select id="qtp-layer-select" style="min-width:120px; padding:6px 8px;">
      <option value="002">002</option>
      <option value="007">007</option>
      <option value="013">013</option>
      <option value="023">023</option>
      <option value="039">039</option>
      <option value="066">066</option>
      <option value="111">111</option>
      <option value="184">184</option>
      <option value="275">275</option>
      <option value="Mean">Mean</option>
    </select>
  </label>
  <label style="display:flex; flex-direction:column; font-size:0.92rem; color:#333;">
    Year
    <select id="qtp-year-select" style="min-width:120px; padding:6px 8px;"></select>
  </label>
</div>

<div style="position:relative;">
  <div id="qtp-soil-temp-map" style="height: 560px; width: 100%; border-radius: 8px; border: 1px solid #ddd;"></div>
  <a id="qtp-download-link" href="#" download style="position:absolute; top:12px; right:12px; z-index:1000; display:inline-flex; align-items:center; justify-content:center; height:34px; padding:0 12px; border:1px solid #666; border-radius:5px; text-decoration:none; color:#111; background:rgba(255,255,255,0.95); box-sizing:border-box;">
    Download
  </a>
</div>
<p id="qtp-soil-temp-status" style="margin-top: 0.75rem; color: #555;">Loading raster...</p>
<style>
  #qtp-soil-temp-map .raster-overlay-crisp {
    image-rendering: pixelated;
    image-rendering: crisp-edges;
    -ms-interpolation-mode: nearest-neighbor;
  }
</style>

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://unpkg.com/geotiff@2.1.3/dist-browser/geotiff.js"></script>
<script>
  window.addEventListener("load", function () {
    const mapElement = document.getElementById("qtp-soil-temp-map");
    const statusElement = document.getElementById("qtp-soil-temp-status");
    const varSelect = document.getElementById("qtp-var-select");
    const layerSelect = document.getElementById("qtp-layer-select");
    const yearSelect = document.getElementById("qtp-year-select");
    const downloadLink = document.getElementById("qtp-download-link");

    if (!mapElement || !statusElement || !varSelect || !layerSelect || !yearSelect || !downloadLink) return;
    if (typeof L === "undefined") {
      statusElement.textContent = "Map library failed to load (Leaflet not available).";
      return;
    }
    if (typeof GeoTIFF === "undefined") {
      statusElement.textContent = "GeoTIFF library failed to load.";
      return;
    }

    const map = L.map(mapElement).setView([33.5, 91.0], 4);
    let rasterLayer = null;
    let legendControl = null;
    let loadingId = 0;

    L.tileLayer("https://{s}.basemaps.cartocdn.com/light_nolabels/{z}/{x}/{y}{r}.png", {
      maxZoom: 18,
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> &copy; CARTO',
    }).addTo(map);

    for (let y = 1979; y <= 2018; y++) {
      const opt = document.createElement("option");
      opt.value = String(y);
      opt.textContent = String(y);
      yearSelect.appendChild(opt);
    }
    yearSelect.value = "2018";

    const buildTifPath = function (variableKey, layerKey, year) {
      if (variableKey === "st") {
        if (layerKey === "Mean") {
          return "{{ '/assets/data/PHD-TP/Soil temperature/Mean/Tif_ST_Mean_' | relative_url }}" + year + ".tif";
        }
        return "{{ '/assets/data/PHD-TP/Soil temperature/' | relative_url }}" + layerKey + "/Tif_ST_Layer" + layerKey + "_" + year + ".tif";
      }
      if (variableKey === "liq") {
        if (layerKey === "Mean") {
          return "{{ '/assets/data/PHD-TP/Unfrozen water content/Mean/Tif_Liq_Mean_' | relative_url }}" + year + ".tif";
        }
        return "{{ '/assets/data/PHD-TP/Unfrozen water content/' | relative_url }}" + layerKey + "/Tif_Liq_Layer" + layerKey + "_" + year + ".tif";
      }
      if (variableKey === "sme") {
        if (layerKey === "Mean") {
          return "{{ '/assets/data/PHD-TP/Soil moisture equivalent/Mean/Tif_SME_Mean_' | relative_url }}" + year + ".tif";
        }
        return "{{ '/assets/data/PHD-TP/Soil moisture equivalent/' | relative_url }}" + layerKey + "/Tif_SME_Layer" + layerKey + "_" + year + ".tif";
      }
      if (layerKey === "Mean") {
        return "{{ '/assets/data/PHD-TP/Ice content/Mean/Tif_Ice_Mean_' | relative_url }}" + year + ".tif";
      }
      return "{{ '/assets/data/PHD-TP/Ice content/' | relative_url }}" + layerKey + "/Tif_Ice_Layer" + layerKey + "_" + year + ".tif";
    };

    const prettyVariableName = function (variableKey) {
      if (variableKey === "st") return "Soil Temperature";
      if (variableKey === "liq") return "Unfrozen Water Content";
      if (variableKey === "sme") return "Soil Moisture Equivalent";
      return "Ice Content";
    };

    const withTimeout = function (promise, ms, stage) {
      return Promise.race([
        promise,
        new Promise(function (_, reject) {
          setTimeout(function () {
            reject(new Error(stage + " timed out."));
          }, ms);
        }),
      ]);
    };

    const loadCurrentRaster = async function () {
      loadingId += 1;
      const currentId = loadingId;
      const variableKey = varSelect.value;
      const layerKey = layerSelect.value;
      const year = yearSelect.value;
      const tifUrl = buildTifPath(variableKey, layerKey, year);

      downloadLink.href = tifUrl;
      downloadLink.setAttribute("download", tifUrl.split("/").pop());

      try {
        statusElement.textContent = "Fetching raster file...";
        const response = await withTimeout(fetch(tifUrl), 15000, "Fetch");
        if (!response.ok) throw new Error("Failed to fetch GeoTIFF (" + response.status + ").");
        if (currentId !== loadingId) return;

        statusElement.textContent = "Parsing GeoTIFF...";
        const arrayBuffer = await withTimeout(response.arrayBuffer(), 15000, "Read ArrayBuffer");
        const tiff = await withTimeout(GeoTIFF.fromArrayBuffer(arrayBuffer), 30000, "Open GeoTIFF");
        const image = await withTimeout(tiff.getImage(), 30000, "Read first image");
        if (currentId !== loadingId) return;

        const bbox = image.getBoundingBox();
        const width = image.getWidth();
        const height = image.getHeight();
        const raster = await withTimeout(
          image.readRasters({ samples: [0], interleave: true }),
          180000,
          "Read raster data"
        );
        if (currentId !== loadingId) return;

        const nodataRaw = image.getGDALNoData();
        const nodataValue = nodataRaw === null || nodataRaw === undefined ? null : Number(nodataRaw);

        const isNoData = function (v) {
          if (!Number.isFinite(v)) return true;
          if (v <= -1e20 || v >= 1e20) return true;
          if (nodataValue !== null && Number.isFinite(nodataValue) && Math.abs(v - nodataValue) < 1e-8) return true;
          return false;
        };

        let rawMinValue = Infinity;
        let rawMaxValue = -Infinity;
        for (let i = 0; i < raster.length; i++) {
          const v = raster[i];
          if (isNoData(v)) continue;
          if (v < rawMinValue) rawMinValue = v;
          if (v > rawMaxValue) rawMaxValue = v;
        }
        const looksLikeKelvin = rawMinValue > 150 && rawMaxValue > 150;
        const toDisplay = function (v) {
          return looksLikeKelvin ? v - 273.15 : v;
        };

        let minValue = Infinity;
        let maxValue = -Infinity;
        for (let i = 0; i < raster.length; i++) {
          const v = raster[i];
          if (isNoData(v)) continue;
          const dv = toDisplay(v);
          if (dv < minValue) minValue = dv;
          if (dv > maxValue) maxValue = dv;
        }
        if (!Number.isFinite(minValue) || !Number.isFinite(maxValue)) {
          throw new Error("No valid raster values found.");
        }

        const interpolate = function (a, b, t) {
          return Math.round(a + (b - a) * t);
        };
        const colorFor = function (v) {
          if (variableKey === "liq" || variableKey === "sme") {
            const lo = 0.0;
            const hi = Math.max(maxValue, lo + 1e-6);
            const t = (v - lo) / (hi - lo);
            const clamped = Math.max(0, Math.min(1, t));
            // Tan -> dark green
            return [
              interpolate(210, 0, clamped),
              interpolate(180, 100, clamped),
              interpolate(120, 0, clamped),
            ];
          }
          if (variableKey === "ice") {
            const lo = Math.min(minValue, maxValue);
            const hi = Math.max(maxValue, lo + 1e-6);
            const t = (v - lo) / (hi - lo);
            const clamped = Math.max(0, Math.min(1, t));
            // Light blue -> deep purple
            return [
              interpolate(173, 75, clamped),
              interpolate(216, 0, clamped),
              interpolate(230, 130, clamped),
            ];
          }
          const centerValue = 0.0;
          const halfSpan = Math.max(Math.abs(maxValue - centerValue), Math.abs(minValue - centerValue), 0.000001);
          if (v <= centerValue) {
            const t = (v - (centerValue - halfSpan)) / halfSpan;
            const clamped = Math.max(0, Math.min(1, t));
            return [
              interpolate(49, 255, clamped),
              interpolate(130, 255, clamped),
              interpolate(189, 255, clamped),
            ];
          }
          const t = (v - centerValue) / halfSpan;
          const clamped = Math.max(0, Math.min(1, t));
          return [
            interpolate(255, 215, clamped),
            interpolate(255, 48, clamped),
            interpolate(255, 39, clamped),
          ];
        };

        const minLon = bbox[0];
        const minLat = bbox[1];
        const maxLon = bbox[2];
        const maxLat = bbox[3];
        const dataBounds = L.latLngBounds([minLat, minLon], [maxLat, maxLon]);

        const RasterGridLayer = L.GridLayer.extend({
          createTile: function (coords) {
            const tile = document.createElement("canvas");
            const tileSize = this.getTileSize();
            tile.width = tileSize.x;
            tile.height = tileSize.y;
            tile.className = "raster-overlay-crisp";
            const ctx = tile.getContext("2d");
            const img = ctx.createImageData(tileSize.x, tileSize.y);
            const px = img.data;

            const nw = map.unproject(L.point(coords.x * tileSize.x, coords.y * tileSize.y), coords.z);
            const se = map.unproject(L.point((coords.x + 1) * tileSize.x, (coords.y + 1) * tileSize.y), coords.z);
            const lonLeft = nw.lng;
            const lonRight = se.lng;
            const latTop = nw.lat;
            const latBottom = se.lat;

            for (let y = 0; y < tileSize.y; y++) {
              const tY = y / (tileSize.y - 1 || 1);
              const lat = latTop + (latBottom - latTop) * tY;
              if (lat < minLat || lat > maxLat) continue;
              const row = Math.floor(((maxLat - lat) / (maxLat - minLat)) * (height - 1));
              if (row < 0 || row >= height) continue;

              for (let x = 0; x < tileSize.x; x++) {
                const tX = x / (tileSize.x - 1 || 1);
                const lon = lonLeft + (lonRight - lonLeft) * tX;
                if (lon < minLon || lon > maxLon) continue;
                const col = Math.floor(((lon - minLon) / (maxLon - minLon)) * (width - 1));
                if (col < 0 || col >= width) continue;

                const srcIdx = row * width + col;
                const value = raster[srcIdx];
                const dstIdx = (y * tileSize.x + x) * 4;
                if (isNoData(value)) {
                  px[dstIdx + 3] = 0;
                  continue;
                }
                const dv = toDisplay(value);
                const c = colorFor(dv);
                px[dstIdx] = c[0];
                px[dstIdx + 1] = c[1];
                px[dstIdx + 2] = c[2];
                px[dstIdx + 3] = 255;
              }
            }

            ctx.putImageData(img, 0, 0);
            return tile;
          },
        });

        if (rasterLayer) map.removeLayer(rasterLayer);
        rasterLayer = new RasterGridLayer({
          tileSize: 256,
          noWrap: true,
          opacity: 1.0,
          updateWhenZooming: false,
          updateWhenIdle: true,
          bounds: dataBounds,
        });
        rasterLayer.addTo(map);
        map.fitBounds(dataBounds);

        if (legendControl) map.removeControl(legendControl);
        legendControl = L.control({ position: "bottomright" });
        legendControl.onAdd = function () {
          const div = L.DomUtil.create("div", "info legend");
          div.style.background = "rgba(255,255,255,0.92)";
          div.style.padding = "10px 12px";
          div.style.border = "1px solid #bbb";
          div.style.borderRadius = "6px";
          div.style.boxShadow = "0 1px 4px rgba(0,0,0,0.2)";
          div.style.lineHeight = "1.2";
          div.style.minWidth = "190px";
          if (variableKey === "liq" || variableKey === "sme") {
            div.innerHTML =
              '<div style="font-weight:600; margin-bottom:6px;">' +
              prettyVariableName(variableKey) +
              " (raw)</div>" +
              '<div style="height:12px; border:1px solid #999; border-radius:3px; background:linear-gradient(to right, rgb(210,180,120), rgb(0,100,0));"></div>' +
              '<div style="display:flex; justify-content:space-between; font-size:12px; margin-top:4px;">' +
              "<span>0</span>" +
              "<span>" + maxValue.toFixed(1) + "</span>" +
              "</div>";
          } else if (variableKey === "ice") {
            div.innerHTML =
              '<div style="font-weight:600; margin-bottom:6px;">' +
              prettyVariableName(variableKey) +
              " (raw)</div>" +
              '<div style="height:12px; border:1px solid #999; border-radius:3px; background:linear-gradient(to right, rgb(173,216,230), rgb(75,0,130));"></div>' +
              '<div style="display:flex; justify-content:space-between; font-size:12px; margin-top:4px;">' +
              "<span>" + minValue.toFixed(1) + "</span>" +
              "<span>" + maxValue.toFixed(1) + "</span>" +
              "</div>";
          } else {
            div.innerHTML =
              '<div style="font-weight:600; margin-bottom:6px;">' +
              prettyVariableName(variableKey) +
              " (" +
              (looksLikeKelvin ? "deg C" : "raw") +
              ")</div>" +
              '<div style="height:12px; border:1px solid #999; border-radius:3px; background:linear-gradient(to right, rgb(49,130,189), rgb(255,255,255), rgb(215,48,39));"></div>' +
              '<div style="display:flex; justify-content:space-between; font-size:12px; margin-top:4px;">' +
              "<span>" + minValue.toFixed(1) + "</span>" +
              "<span>0</span>" +
              "<span>" + maxValue.toFixed(1) + "</span>" +
              "</div>";
          }
          return div;
        };
        legendControl.addTo(map);

        statusElement.textContent =
          "Loaded: " + prettyVariableName(variableKey) + ", layer " + layerKey + ", year " + year +
          ". Range: " + minValue.toFixed(2) + " to " + maxValue.toFixed(2) +
          (looksLikeKelvin ? " deg C." : " raw unit.");
      } catch (error) {
        if (currentId !== loadingId) return;
        statusElement.textContent = "Failed to load raster: " + error.message;
      }
    };

    varSelect.addEventListener("change", loadCurrentRaster);
    layerSelect.addEventListener("change", loadCurrentRaster);
    yearSelect.addEventListener("change", loadCurrentRaster);
    loadCurrentRaster();
  });
</script>

#### References

1. Zhao, Y., Nan, Z., Ji, H., Chen, Y., Ou, M., & Li, D. (2025). *A hybrid modeling approach for improved simulation of thermal-hydrological dynamics in active layer on the Qinghai-Tibet Plateau*. **Water Resources Research, 61**(12), e2025WR040288. [https://doi.org/10.1029/2025WR040288](https://doi.org/10.1029/2025WR040288)
2. Zhao, Y., Nan, Z., Cao, Z., Ji, H., & Hu, J. (2023). *Evaluation of parameterization schemes for matric potential in frozen soil in land surface models: A modeling perspective*. **Water Resources Research, 59**(6), e2023WR034644. [https://doi.org/10.1029/2023WR034644](https://doi.org/10.1029/2023WR034644)
