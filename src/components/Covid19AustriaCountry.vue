<template>
  <div>
    <div id="covid19-country-map" />
    <div class="data-date">
      <p>letzte Datenaktualisierung:</p>
      <span id="timeOfDate">- - -</span>
    </div>
  </div>
</template>

<script>
import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import VectorLayer from 'ol/layer/Vector';
import VectorSource from 'ol/source/Vector';
import XYZ from 'ol/source/XYZ';
import ZoomToExtent from 'ol/control/ZoomToExtent';
import { defaults } from 'ol/control';
import { transformExtent } from 'ol/proj';
import { GeoJSON } from 'ol/format';
import { Fill, Stroke, Style, Text } from 'ol/style';
import { defaults as defaultInteractions } from 'ol/interaction';


export default {
  name: 'Covid19CountryMap',
  mounted () {
    const me = this;

    // Create Basemap Tile Layer
    var basemap2 = new TileLayer({
      source: new XYZ({
        maxZoom: 19,
        attributions: [
          'Hintergrundkarte:  <a href="http://www.basemap.at">basemap.at</a> &copy; <a href="http://creativecommons.org/licenses/by/3.0/at/">CC BY 3.0 AT</a>',
        ],
        crossOrigin: "anonymous",
        url:
          "//maps{1-4}.wien.gv.at/basemap/bmapgelaende/grau/google3857/{z}/{y}/{x}.jpeg",
      }),
    });

    // Main Map
    me.countryMap = new Map({
      target: "covid19-country-map",
      layers: [basemap2],
       view: new View({
        extent: transformExtent(
          [7, 44, 20, 52],
          "EPSG:4326",
          "EPSG:3857"
        ),
      }),
      controls: defaults({
        attribution: true,
        zoom: true,
        rotate: false,
      }),
      interactions: new defaultInteractions({
        doubleClickZoom: true,
        dragAndDrop: false,
        keyboardPan: false,
        dragPan: true,
        pointer: false,
        select: false,
        altShiftDragRotate: false,
        pinchRotate: false,
        mouseWheelZoom: false,
        pinchZoom: true,
        onFocusOnly: false
      }),
    });

    function Quartile_25(data) {
      return Quartile(data, 0.25);
    }

    function Quartile_50(data) {
      return Quartile(data, 0.5);
    }

    function Quartile_75(data) {
      return Quartile(data, 0.75);
    }

    function Quartile(data, q) {
      data = data.filter((x) => x); // filter all NaN and other JavaScript Falsy values
      data = data.sort((a, b) => a - b);
      var pos = (data.length - 1) * q;
      var base = Math.floor(pos);
      var rest = pos - base;
      if (data[base + 1] !== undefined) {
        return data[base] + rest * (data[base + 1] - data[base]);
      } else {
        return data[base];
      }
    }

    /** calculates the Styling for the features in the initial districtsMap or countryMap view
     *  ATTENTION needs a binded quartileObject to work!!
     */
      function calculateInitialCountriesMapStyling(feature) {
      let quartileObject = this; // Magic with bind
      return [
        calculatePolygonStyleAccordingFeatureValues(feature),
        calculateTextStyleAccordingCountriesFeatureValues(feature, quartileObject),
      ];
    }

    /** 
     *  calculates the styling of a polygon
     */
    function calculatePolygonStyleAccordingFeatureValues(feature) {
      let fillColor;

      if (feature.values_.infected == null) {
        fillColor = "rgba(255, 255, 255, 0.1)";
      } else {
        fillColor = "rgba(204, 51, 51,0.1)";
      }
      return new Style({
        stroke: new Stroke({
          color: "rgb(76, 81, 102)",
          width: 1,
        }),
        fill: new Fill({
          color: fillColor,
        }),
      });
    }

    function chooseTextPoint(feature) {
      var retPoint;
      if (feature.getGeometry().getType() === "MultiPolygon") {
        // exception for Tyrol - it is a Multipolygon with 3 elements
        if (feature.values_.BL === "Tirol") {
          retPoint = feature
            .getGeometry()
            .getPolygons()[2]
            .getInteriorPoint();
        } else {
          retPoint = feature
            .getGeometry()
            .getPolygons()[0]
            .getInteriorPoint();
        }
      } else if (feature.getGeometry().getType() === "Polygon") {
        retPoint = feature.getGeometry().getInteriorPoint();
      }
      return retPoint;
    }


    function calculateTextStyleAccordingCountriesFeatureValues( feature, quartileObject ) {
      //adjust labels according to the quartile's of all infected persons
      let colorPerQuartile = "rgba(204, 51, 51, 0.2)";
      let fontPerQuartile = "bold 10px Roboto";
      if (feature.values_.infected <= quartileObject.quartile_25) {
        colorPerQuartile = "rgba(0, 0, 0,0.7)";
        fontPerQuartile = `${me.countryMap.getView().getZoom()*1.5}px Roboto`; 
        // because both maps have the same extent it does not matter if district of 
      } else if (
        feature.values_.infected > quartileObject.quartile_25 &&
        feature.values_.infected <= quartileObject.quartile_50
      ) {
        colorPerQuartile = "rgba(0, 0, 0,0.8)";
        fontPerQuartile = `${me.countryMap.getView().getZoom()*1.6}px Roboto`;
      } else if (
        feature.values_.infected > quartileObject.quartile_50 &&
        feature.values_.infected <= quartileObject.quartile_75
      ) {
        colorPerQuartile = "rgba(0, 0, 0,0.9)";
        fontPerQuartile = `${me.countryMap.getView().getZoom()*1.7}px Roboto`;
      } else if (feature.values_.infected > quartileObject.quartile_75) {
        // > quartile_75
        colorPerQuartile = "rgba(0, 0, 0,1)";
        fontPerQuartile = `${me.countryMap.getView().getZoom()*1.8}px Roboto`;
      }

      let stroke = new Stroke({
        color: "rgba(255, 255, 255, 0.5)",
        width: 3,
      });
      return new Style({
        text: new Text({
          text: `${
            feature.values_.infected == null
              ? " kein Wert"
              : feature.values_.infected
          }`,
          fill: new Fill({ color: colorPerQuartile }),
          font: fontPerQuartile,
          stroke: stroke,
          padding: [5, 5, 5, 5], // text should not overlap in the districtsMap
          //overflow: true, // text is shown, even when the text is larger than the polygon
          placement: "point",
        }),
        geometry: chooseTextPoint,
        zIndex: 10,
      });
    }

    function getActualDataForCountryID(actualData, id) {
      return actualData.bundesland[id];
    }

    function createCountryVectorSource(actualData, countries) {
        let vectorSource = new VectorSource({
          features: new GeoJSON().readFeatures(countries),
          attributions:
            "COVID-19 Datenquelle: BMSGPK via <a href='https://corin.at'>corin.at</a>, Einwohnerzahlen: STATISTIK AUSTRIA Erstellt am 20.1.2020.",
        });
        
        vectorSource.getFeatures().forEach((feature) => {
          feature.setId(feature.values_.ID);
          feature.setProperties({
            infected: getActualDataForCountryID(actualData, feature.values_.ID),
          });
        });
        return vectorSource;
    }
    function calculateStatistics(vectorSource) {
      let datafeatures = [];
      vectorSource.getFeatures().forEach(function (feature) {
        datafeatures.push(Number(feature.values_.infected));
      });
      return {
        quartile_25: Quartile_25(datafeatures),
        quartile_50: Quartile_50(datafeatures),
        quartile_75: Quartile_75(datafeatures),
      };
    }

    // Fetching Data
    let actualDataPromise = fetch('https://gisolutions.at/covid19info/facade.php');
    let countriesPromise = fetch('https://gisolutions.at/covid19info/data/countries_3857.geojson');
    Promise.all([actualDataPromise, countriesPromise])
      .then((res) => Promise.all(res.map((r) => r.json())))
      .then(([actualData, countries]) => {
        // console.log(actualData);
        // console.log(countries.features);
        
        // Define vector source
        let vectorSource = createCountryVectorSource(actualData, countries);
        document.getElementById("timeOfDate").innerHTML = actualData.dataTime;
        let counrtryStatistics = calculateStatistics(vectorSource);
        
        // Define vector layer
        var vectorLayer = new VectorLayer ({
          source: vectorSource,
          style: calculateInitialCountriesMapStyling.bind(counrtryStatistics),
        });

        me.countryMap.getView().fit(vectorLayer.getSource().getExtent());
        me.countryMap.addControl(
          new ZoomToExtent({
            extent: vectorLayer.getSource().getExtent(),
          })
        );
        me.countryMap.addLayer(vectorLayer);

      })
      .catch((err) => console.error(err));

      function calculateInteractionStyleforFeature(feature) {
        let promile = Math.round(
          (feature.values_.infected / feature.values_.inhab) * 1000
        );
        let stroke = new Stroke({
          color: "rgba(255, 255, 255, 0.5)",
          width: 1.5,
        });
        return [
          new Style({
            text: new Text({
              text: `${feature.values_.BL} \n  Einwohner: ${
                feature.values_.inhab
              }\n Infektionen: ${
                feature.values_.infected == null
                  ? " kein Wert"
                  : feature.values_.infected
              } \n Verhältnis: ${promile == 0 ? "< 1" : "≈" + promile}‰`,
              fill: new Fill({ color: "rgb(76, 81, 102)" }),
              font: "14px Roboto",
              stroke: stroke,
              padding: [10, 10, 10, 10],
              overflow: true,
              backgroundFill: new Fill({
                color: "rgba(255, 255, 255, 1)",
              }),
            }),
            geometry: chooseTextPoint,
            zIndex: 20,
          }),
          new Style({
            fill: new Fill({
              color: "rgba(204, 51, 51,0.5)",
            }),
            stroke: new Stroke({
              color: "rgb(204, 51, 51)",
              width: 3,
            }),
          }),
        ];
      }

       //events on the country map
      var selectedCountry = null;
      me.countryMap.on(["click", "pointermove"], function (e) {
        if (selectedCountry !== null) {
          selectedCountry.setStyle(undefined);
          selectedCountry = null;
        }
        //change the style of a polygon if there was an event on it
        me.countryMap.forEachFeatureAtPixel(e.pixel, function (feature) {
          selectedCountry = feature;
          feature.setStyle(calculateInteractionStyleforFeature(feature));
          return true;
        });
        // TODO add here code for a table view
      });

  }
}
</script>

<style scoped>
  #covid19-country-map {
    height: 600px;
    width: 100%;
  }

  .data-date {
    background-color: #929292;
    color: #fff;
    display: flex;
    flex-direction: row;
    align-items: center;
  }

  p {
    padding: 0 1rem 0 1rem;
  }
</style>