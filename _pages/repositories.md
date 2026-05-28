---
layout: page
permalink: /repositories/
title: Data
description: All data and code related to my publications can be accessed through the Data and Code Availability statements within the respective papers, or by contacting me directly.
nav: true
nav_order: 4
---

### Permafrost hydro-thermal dataset of Tibetan Plateau (PHD-TP)

This dataset provides key hydrological and thermal variables of the active layer and near-surface permafrost across the Tibetan Plateau at 10 km spatial resolution and daily temporal resolution from 1979 to 2018. Annual mean products are available for direct download here, while the original daily-scale dataset can be obtained by contacting the corresponding author.

- [Download GeoTIFF]({{ '/assets/data/qtp_soil_temperature.tif' | relative_url }})

<div id="qtp-soil-temp-map" style="height: 560px; width: 100%; border-radius: 8px; border: 1px solid #ddd;"></div>
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
    const tifUrl = "{{ '/assets/data/qtp_soil_temperature.tif' | relative_url }}";
    const mapElement = document.getElementById("qtp-soil-temp-map");
    const statusElement = document.getElementById("qtp-soil-temp-status");

    if (!mapElement || !statusElement) return;
    if (typeof L === "undefined") {
      statusElement.textContent = "Map library failed to load (Leaflet not available).";
      return;
    }
    if (typeof GeoTIFF === "undefined") {
      statusElement.textContent = "GeoTIFF library failed to load.";
      return;
    }

    const map = L.map(mapElement).setView([33.5, 91.0], 4);

    L.tileLayer("https://{s}.basemaps.cartocdn.com/light_nolabels/{z}/{x}/{y}{r}.png", {
      maxZoom: 18,
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> &copy; CARTO',
    }).addTo(map);

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

    (async function () {
      try {
        statusElement.textContent = "Fetching raster file...";
        const response = await withTimeout(fetch(tifUrl), 15000, "Fetch");
        if (!response.ok) throw new Error("Failed to fetch GeoTIFF (" + response.status + ").");

        statusElement.textContent = "Parsing GeoTIFF...";
        const arrayBuffer = await withTimeout(response.arrayBuffer(), 15000, "Read ArrayBuffer");
        const tiff = await withTimeout(GeoTIFF.fromArrayBuffer(arrayBuffer), 30000, "Open GeoTIFF");
        const image = await withTimeout(tiff.getImage(), 30000, "Read first image");
        const bbox = image.getBoundingBox();
        const width = image.getWidth();
        const height = image.getHeight();
        const raster = await withTimeout(
          image.readRasters({ samples: [0], interleave: true }),
          180000,
          "Read raster data"
        );
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
        const toCelsius = function (v) {
          return looksLikeKelvin ? v - 273.15 : v;
        };

        let minValue = Infinity;
        let maxValue = -Infinity;
        for (let i = 0; i < raster.length; i++) {
          const v = raster[i];
          if (isNoData(v)) continue;
          const vc = toCelsius(v);
          if (vc < minValue) minValue = vc;
          if (vc > maxValue) maxValue = vc;
        }
        if (!Number.isFinite(minValue) || !Number.isFinite(maxValue)) {
          throw new Error("No valid raster values found.");
        }
        const centerValue = 0.0;
        const halfSpan = Math.max(Math.abs(maxValue - centerValue), Math.abs(minValue - centerValue), 0.000001);

        const interpolate = function (a, b, t) {
          return Math.round(a + (b - a) * t);
        };

        // Blue -> White -> Red diverging colormap
        const colorFor = function (v) {
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

        statusElement.textContent = "Rendering map layer...";
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
                const vc = toCelsius(value);
                const c = colorFor(vc);
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

        const rasterLayer = new RasterGridLayer({
          tileSize: 256,
          noWrap: true,
          opacity: 1.0,
          updateWhenZooming: false,
          updateWhenIdle: true,
          bounds: dataBounds,
        });
        rasterLayer.addTo(map);
        map.fitBounds(dataBounds);

        const legend = L.control({ position: "bottomright" });
        legend.onAdd = function () {
          const div = L.DomUtil.create("div", "info legend");
          div.style.background = "rgba(255,255,255,0.92)";
          div.style.padding = "10px 12px";
          div.style.border = "1px solid #bbb";
          div.style.borderRadius = "6px";
          div.style.boxShadow = "0 1px 4px rgba(0,0,0,0.2)";
          div.style.lineHeight = "1.2";
          div.style.minWidth = "190px";
          div.innerHTML =
            '<div style="font-weight:600; margin-bottom:6px;">Soil Temperature (deg C)</div>' +
            '<div style="height:12px; border:1px solid #999; border-radius:3px; background:linear-gradient(to right, rgb(49,130,189), rgb(255,255,255), rgb(215,48,39));"></div>' +
            '<div style="display:flex; justify-content:space-between; font-size:12px; margin-top:4px;">' +
            "<span>" + minValue.toFixed(1) + "</span>" +
            "<span>0</span>" +
            "<span>" + maxValue.toFixed(1) + "</span>" +
            "</div>";
          return div;
        };
        legend.addTo(map);

        statusElement.textContent =
          "Loaded. Range: " +
          minValue.toFixed(2) +
          " to " +
          maxValue.toFixed(2) +
          " deg C. Use mouse wheel to zoom and drag to pan.";
      } catch (error) {
        statusElement.textContent = "Failed to load raster: " + error.message;
      }
    })();
  });
</script>

