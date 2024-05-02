<!DOCTYPE html>
<html>
  <head>
    <link
      rel="stylesheet"
      href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css"
      integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T"
      crossorigin="anonymous"
    />
    <link
      rel="stylesheet"
      href="https://openlayers.org/en/v6.1.1/css/ol.css"
    />
    <link
      href="https://fonts.googleapis.com/css?family=Roboto&display=swap"
      rel="stylesheet"
    />

    <link
        rel="stylesheet"
        href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.11.2/css/all.css"
		integrity="sha256-46qynGAkLSFpVbEBog43gvNhfrOj+BmwXdxFgVK/Kvc="
		crossorigin="anonymous" />

    <style>
      #map {
        background: #202020;
        font-size: 16px;
        font-family: "Roboto", sans-serif;
      }

      body,
      html,
      #map {
        margin: 0px;
        width: 100%;
        height: 100%;
      }

      .layer-select {
        position: absolute;
        top: 0.5em;
        right: 0.5em;
        width: 150px;
      }
    </style>
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  </head>

  <body margin="0" padding="0">
    <script src="https://openlayers.org/en/v6.1.1/build/ol.js"></script>
    <script
      src="https://code.jquery.com/jquery-3.3.1.slim.min.js"
      integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo"
      crossorigin="anonymous"
    ></script>
    <script
      src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"
      integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1"
      crossorigin="anonymous"
    ></script>
    <script
      src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"
      integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM"
      crossorigin="anonymous"
    ></script>

    <script src="playersData.js"></script>

    <div id="map"></div>
    <script>
      var layers = {
        dim0: {
          name: "Overworld",
          attribution:
            '<a href="https://github.com/mjungnickel18/papyruscs">PapyrusCS</a>',
          minNativeZoom: 15,
          minZoom: 15,
          maxNativeZoom: 20,
          maxZoom: 22,
          noWrap: true,
          tileSize: 512,
          folder: "dim0",
          fileExtension: "png"
        }
      };

      var config = {
        factor: 1
      };

      layers = {"dim0":{"name":"Overworld","attribution":"Generated by <a href=\"https://github.com/mjungnickel18/papyruscs\">PapyrusCS</a>","minNativeZoom":13,"maxNativeZoom":20,"noWrap":true,"tileSize":512,"folder":"dim0","fileExtension":"png"}}; 
config = {"factor":65536.0,"globalMinZoom":13,"globalMaxZoom":20,"tileSize":512,"blocksPerTile":32};

      // ======

      // For the extent sizes specified below, OpenLayers tries to fit
      // the whole extents given in a space [0, 0] -> [2^8, 2^8] visually
      // on screen at zoom level 0.
      //
      // This constant represents that internal tile size that OpenLayers
      // will aim for, so that we can calculate zoom levels correctly later.
      const openLayersInternalTileSize = Math.pow(2, 8); // 256

      // The actual size of the tiles that we're using in Papyrus.
      const tileSize = config.tileSize;

      // The maximum positive extents of the whole map. The units for this
      // value are "number of tiles at the most zoomed out level generated
      // by Papyrus". That is, if Papyrus generates a minimum zoom level
      // of 15, then the units for this extent size are the number of (Papyrus)
      // zoom level 15 tiles to display at (OpenLayers) zoom level 0.
      //
      // The minimum zoom level that Papyrus is using is in config.globalMinZoom.
      const maximumExtentSize = 10000;

      // The minimum negative extents of the whole map. Uses the same units
      // as maximumExtentSize.
      const minimumExtentSize = -10000;

      // Computes the resolutions array to use for OpenLayers. For each OpenLayers
      // zoom level (0 through 42), this computes the ratio such that a single tile
      // at OpenLayers zoom level 0 is a single tile at the minimum Papyrus zoom level.
      const papyrusMinimumZoomScale = Math.pow(2, config.globalMinZoom);
      const convertedFromTilesToPixelsUsingTileSize = papyrusMinimumZoomScale / tileSize;
      const resolutions = new Array(43);
      for (let z = 0; z < 43; ++z) {
        resolutions[z] = convertedFromTilesToPixelsUsingTileSize / Math.pow(2, z);
      }

      // When we calculate resolutions above, we've effectively saying that zoom level 0 is
      // zoom level N, where N is the lowest zoom level. This zoom level N also becomes
      // the range of [0, 0] -> [1, 1] in the coordinate system. We need to be able to translate
      // coordinates in that "zoomed out" coordinate system, back down to the maximum zoom level
      // where each tile represents 32x32 (depending on tilesize) Minecraft tiles.
      const zoomRatioForMaximumZoom = 1 / Math.pow(2, config.globalMaxZoom - config.globalMinZoom);
      const minecraftTilesAtMostZoomedInLevel = config.blocksPerTile;

      // Use a projection where pixels have a 1:1 ratio with the screen at zoom level 0.
      const projection = new ol.proj.Projection({
        code: "ZOOMIFY",
        units: "pixels",
        extent: [
          0,
          0,
          openLayersInternalTileSize / tileSize,
          openLayersInternalTileSize / tileSize
        ]
      });

      // Construct the tile grid using the desired maximum and minimum extents listed above,
      // set the origin to [0, 0] (the center of the map), and the calculated resolutions array.
      const tilegrid = new ol.tilegrid.TileGrid({
        extent: [
          minimumExtentSize,
          minimumExtentSize,
          maximumExtentSize,
          maximumExtentSize
        ],
        origin: [0, 0],
        resolutions: resolutions,
        tileSize: [tileSize, tileSize]
      });

      let map;
      let locationElement;

      const tileLayers = Object.keys(layers)
        .sort()
        .map(function(layerKey, idx) {
          const layer = layers[layerKey];
          const tileLayer = new ol.layer.Tile({
            source: new ol.source.XYZ({
              tileUrlFunction: function(tileCoord, pixelRatio, projection) {
                const z = tileCoord[0];
                const x = tileCoord[1];
                const y = tileCoord[2];
                return (
                  "./" +
                  layer.folder +
                  "/" +
                  z +
                  "/" +
                  x +
                  "/" +
                  y +
                  "." +
                  layer.fileExtension
                );
              },
              projection: projection,
              tileGrid: tilegrid,
              attributions: layer.attribution
            }),
            visible: idx == 0
          });
          tileLayer.metaLayerKey = layerKey;
          return tileLayer;
        });

      const initialLayer = layers[Object.keys(layers).sort()[0]];

      if (Object.keys(layers).sort()[0] == "dim0_stronghold") {
        document.getElementById("map").style.background = "#fff";
      } else {
        document.getElementById("map").style.background = "#202020";
      }
                
      const view = new ol.View({
        projection: projection,
        center: [0, 0],
        zoom: 0,
        minZoom: 0,
        maxZoom: config.globalMaxZoom - config.globalMinZoom
      });

      const PapyrusControls = (function(Control) {
        function PapyrusControls(opt_options) {
          const options = opt_options || {};

          const element = document.createElement("div");
          element.className = "layer-select ol-unselectable";

          const card = document.createElement("div");
          card.className = "card";
          element.appendChild(card);

          const cardBody = document.createElement("div");
          cardBody.className = "card-body p-3 px-3";
          card.appendChild(cardBody);

          const form = document.createElement("form");
          cardBody.appendChild(form);

          let currentSelectedLayer = Object.keys(layers).sort()[0];
          let rememberedCenters = {};
          let rememberedZoom = {};

          Object.keys(layers)
            .sort()
            .forEach(function(layerKey, idx) {
              const layer = layers[layerKey];

              const radioContainer = document.createElement("div");
              radioContainer.className = "custom-control custom-radio";

              const radioInput = document.createElement("input");
              radioInput.type = "radio";
              radioInput.id = layerKey;
              radioInput.name = "layers";
              radioInput.className = "custom-control-input";
              radioInput.checked = idx == 0;
              radioInput.value = layerKey;
              radioContainer.appendChild(radioInput);

              const radioLabel = document.createElement("label");
              radioLabel.htmlFor = layerKey;
              radioLabel.className = "custom-control-label";
              radioLabel.innerText = layer.name;
              radioContainer.appendChild(radioLabel);

              const selectLayer = function(e) {
                if (layerKey == "dim0_stronghold") {
                  document.getElementById("map").style.background = "#fff";
                } else {
                  document.getElementById("map").style.background = "#202020";
                }
        
                if (currentSelectedLayer != layerKey) {
                  const runtimeLayers = map.getLayers();
                  runtimeLayers.forEach(function(runtimeLayer) {
                    runtimeLayer.setVisible(
                      runtimeLayer.metaLayerKey == layerKey
                    );
                  });
                  
                  const oldFocusGroup = currentSelectedLayer.substr(0, 4);
                  const newFocusGroup = layerKey.substr(0, 4);

                  rememberedCenters[oldFocusGroup] = view.getCenter();
                  if (rememberedCenters[newFocusGroup] === undefined) {
                    // refocus the map to 0, 0
                    view.setCenter([0, 0]);
                  } else {
                    // set back to where we were
                    view.setCenter(rememberedCenters[newFocusGroup]);
                  }

                  rememberedZoom[oldFocusGroup] = view.getZoom();
                  view.setMinZoom(layer.minNativeZoom - config.globalMinZoom);
                  view.setMaxZoom(layer.maxNativeZoom - config.globalMinZoom);
                  if (rememberedZoom[newFocusGroup] === undefined) {
                    // rezoom the map to minimum zoom
                    view.setZoom(layer.minNativeZoom - config.globalMinZoom);
                  } else {
                    // set back to where we were
                    view.setZoom(rememberedZoom[newFocusGroup]);
                  }

                  const radios = $("input[name='layers']");
                  radios.each(function(idx, elem) {
                    if (elem.value == layerKey) {
                      elem.checked = true;
                    } else {
                      elem.checked = false;
                    }
                  });

                  currentSelectedLayer = layerKey;
                }
              };

              radioInput.addEventListener(
                "click",
                selectLayer.bind(this),
                false
              );

              form.appendChild(radioContainer);
            });

          const hr = document.createElement("hr");
          form.appendChild(hr);

          locationElement = document.createElement("div");
          locationElement.innerText = "X: 0, Z: 0";
          form.appendChild(locationElement);

          Control.call(this, {
            element: element,
            target: options.target
          });
        }

        if (Control) PapyrusControls.__proto__ = Control;
        PapyrusControls.prototype = Object.create(Control && Control.prototype);
        PapyrusControls.prototype.constructor = PapyrusControls;

        return PapyrusControls;
      })(ol.control.Control);

      map = new ol.Map({
        target: "map",
        layers: tileLayers,
        view: view,
        controls: [
          new ol.control.Zoom(),
          new ol.control.Attribution(),
          new PapyrusControls()
        ]
      });

      map.on("pointermove", function(event) {
        var x = Math.floor(
          (event.coordinate[0] / zoomRatioForMaximumZoom) *
            minecraftTilesAtMostZoomedInLevel
        );
        var z = Math.floor(
          (-event.coordinate[1] / zoomRatioForMaximumZoom) *
            minecraftTilesAtMostZoomedInLevel
        );

        locationElement.innerText = "X: " + x + " Z: " + z;
      });

      if (typeof(playersData) !== "undefined") {
        var playerFeatures = [];

        for (var playerIndex in playersData.players)
        {
            var player = playersData.players[playerIndex];

            if (!player.visible) {
                continue;
            }

            if (player.dimensionId !== 0) {
                // TODO: Currently I'm only adding player markers who are in the Overworld
                // We'll want to show players depending on which dimension is being viewed
                // Maybe add a separate layer for players in each dimension
                continue;
            }

            var style = new ol.style.Style({
                text: new ol.style.Text({
                    text: player.name + "\n\uf041", // map-marker
                    font: "900 18px 'Font Awesome 5 Free'",
                    textBaseline: "bottom",
                    fill: new ol.style.Fill({color: player.color}),
                    stroke: new ol.style.Stroke({color: "white", width: 2})
                })
            });

            var playerFeature = new ol.Feature({
                geometry: new ol.geom.Point([
                    (player.position[0] * zoomRatioForMaximumZoom) / minecraftTilesAtMostZoomedInLevel,
                    (-player.position[2] * zoomRatioForMaximumZoom) / minecraftTilesAtMostZoomedInLevel
                ])
            });

            playerFeature.setStyle(style);

            playerFeatures.push(playerFeature);
        }

        var vectorSource = new ol.source.Vector({
            features: playerFeatures
        });

        var vectorLayer = new ol.layer.Vector({
            source: vectorSource
        });

        map.addLayer(vectorLayer);
      }
    </script>
  </body>
</html>
