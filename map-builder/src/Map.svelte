<script>
import { drawCustomPaths, parseAndUnprojectPath } from './paths';
import './style.css';
// import style from './style.css';
// console.log(style)
import {Pane} from 'tweakpane';
import {geoSatellite} from 'd3-geo-projection';
import * as topojson from 'topojson-client';
import * as topojsonSimplify from 'topojson-simplify';
import * as d3 from "d3";
import bbox from 'geojson-bbox';
import union from '@turf/union';
import rewind from '@turf/rewind';
import SVGO from 'svgo/dist/svgo.browser';
import svgoConfig from './svgo.config';


// https://greensock.com/docs/v3/Plugins/MotionPathHelper/static.editPath()
import MotionPathHelper from "./util/MotionPathHelper.js";
// console.log(MotionPathHelper);
const params = {
    longitude: 15,
    latitude: 36,
    altitude: 1000,
    rotation: 0,
    tilt: 25,
    fieldOfView: 50,
    useCanvas: false,
    firstGlow: {
        innerGlow: {blur: 4, strength: 1.4, color: "#7c490eff"},
        outerGlow: {blur: 4, strength: 3, color: '#ffffffff'},
    },
    secondGlow: {
        innerGlow: {blur: 4, strength: 1.4, color: "#A88FAFff"},
        outerGlow: {blur: 4, strength: 3, color: '#ffffffff'},
    },
    useGraticule: true,
    graticuleStep: 3,
    seaColor: "#b4d0ff8d",
    land: {
        show: true,
        fillColor: "#ffffffff",
        strokeColor: "#00000000",
        strokeWidth: 1,
        filter: 'firstGlow'
    },
    countries: {
        show: true,
        hover: true,
        hoverColor: "#fbbc0023",
        fillColor: "#f3efec80",
        strokeColor: "#D1BEB038",
        strokeWidth: 2,
    },
    provided: {
        show: true,
        hover: true,
        hoverColor: "#fbbc0023",
        fillColor: "#ffffff00",
        strokeColor: "#D1BEB038",
        strokeWidth: 1,
    },
    providedBorders: {
        show: true,
        fillColor: "#ffffffff",
        strokeColor: "#00000000",
        strokeWidth: 1,
        filter: ''
    },
    bgNoise: true,
};

const bounds = {
    longitude: [-180, 180],
    latitude: [-90, 90],
    rotation: [-180, 180],
    tilt: [0, 90],
    altitude: [8, 6000],
    fieldOfView: [1, 180],
    blur: [0.5, 10],
    strength: [0.1, 10],
    graticuleStep: [0.1, 10],
    strokeWidth: [0.1, 5],
};
const filterOptions = {
    none: '',
    firstGlow: 'firstGlow',
    secondGlow: 'secondGlow',
};

const positionVars = ['longitude', 'latitude', 'rotation', 'tilt', 'altitude', 'fieldOfView'];
let width = 1800;
const numPixelsY = width * 0.4;
const earthRadius = 6371;
const degrees = 180 / Math.PI;
const offCanvasPx = 5;
let timeoutId;
function gui(opts, redraw) {
    const pane = new Pane({title: 'Options', expanded: false});
    Object.entries(params).forEach(([key, value]) => {
        add(opts, key, value);
    })

    function add(opts, key, value, folder = pane) {
        if (typeof value == 'object') {
            const group = folder.addFolder({title: key, expanded: false});
            Object.entries(value).forEach(([innerKey, innerValue]) => {
                add(value, innerKey, innerValue, group);
            });
        } else {
            let params = {};
            const boundsDef = bounds[key];
            if (boundsDef) {
                params = {min: boundsDef[0], max:boundsDef[1]}
            }
            if (key.includes('filter')) params = { options: filterOptions };
            folder.addInput(opts, key, params);
        }
    }
    pane.on('change', (e) => {
        const prop = e.presetKey;
        if (positionVars.includes(prop)) {
            redraw(true);
        }
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => {
            redraw(false);
        }, 300);
    });
    return pane;
}
const pane = gui(params, draw);

import bg from './assets/img/bg.png';
// import bg from './assets/img/connection-signal.svg';
import uploadIcon from './assets/img/icon-upload.svg';

function appendGlow(selection, id="glows",
                    innerParams = {blur: 2, strength: 1, color: "#7c490e"},
                    outerParams = {blur: 4, strength: 1, color: '#998'}) {
    const colorInner = d3.rgb(innerParams.color);
    const colorOuter = d3.rgb(outerParams.color);
    // let defsElem = d3.select('#map-defs');
    // if (defsElem.empty()) {
    //     defsElem = d3.select('body')
    //     .append('svg')
    //         .attr('width', 0)
    //         .attr('height', 0)
    //     .append('defs')
    //         .attr('id', 'map-defs');
    // }
    const existing = d3.select(`#${id}`);
    // const defs = !existing.empty() ? existing.select(function() { return this.parentNode} ) : 
    //         .append('defs');
    if (!existing.empty()) existing.remove();
    let defs = selection.select('defs');
    if (defs.empty()) defs = selection.append('defs')
    const filter = defs.append('filter').attr('id', id);

    // OUTER GLOW
    // filter.append('feMorphology')
    //     .attr('in', 'SourceGraphic')
    //     .attr('radius', outerParams.strength)
    //     .attr('operator', 'dilate')
    //     .attr('result', 'MASK_OUTER');
    // filter.append('feColorMatrix')
    //     .attr('in', 'MASK_OUTER')
    //     .attr('type', 'matrix')
    //     .attr('values', `0 0 0 0 ${colorOuter.r / 255} 0 0 0 0 ${colorOuter.g / 255} 0 0 0 0 ${colorOuter.b / 255} 0 0 0 ${colorOuter.opacity} 0`) // apply color
    //     .attr('result', 'OUTER_COLORED');
    // filter.append('feGaussianBlur')
    //     .attr('in', 'OUTER_COLORED')
    //     .attr('stdDeviation', outerParams.blur)
    //     .attr('result', 'OUTER_BLUR');
    // filter.append('feComposite')
    //     .attr('in', 'OUTER_BLUR')
    //     .attr('in2', 'SourceGraphic')
    //     .attr('operator', 'out')
    //     .attr('result', 'OUTGLOW');
    filter.append('feDropShadow')
        .attr('flood-color', `${colorOuter}`)
        // .attr('flood-opacity', `${colorOuter.opacity}`)
        .attr('stdDeviation', `${outerParams.strength}`)
        .attr('result', 'OUTGLOW');

    // INNER GLOW
    filter.append('feMorphology')
        .attr('in', 'SourceAlpha')
        .attr('radius', innerParams.strength)
        .attr('operator', 'erode')
        .attr('result', 'INNER_ERODED');

    filter.append('feGaussianBlur')
        .attr('in', 'INNER_ERODED')
        .attr('stdDeviation', innerParams.blur)
        .attr('result', 'INNER_BLURRED');

    filter.append('feColorMatrix')
        .attr('in', 'INNER_BLURRED')
        .attr('type', 'matrix')
        .attr('values', `0 0 0 0 ${colorInner.r / 255} 0 0 0 0 ${colorInner.g / 255} 0 0 0 0 ${colorInner.b / 255} 0 0 0 -1 ${colorInner.opacity }`) // inverse color
        .attr('result', 'INNER_COLOR');

    filter.append('feComposite')
        .attr('in', 'INNER_COLOR')
        .attr('in2', 'SourceGraphic')
        .attr('operator', 'in')
        .attr('result', 'INGLOW');

    // Merge
    const merge = filter.append('feMerge');
    merge.append('feMergeNode').attr('in', 'OUTGLOW');
    merge.append('feMergeNode').attr('in', 'SourceGraphic');
    merge.append('feMergeNode').attr('in', 'INGLOW');
    filter.append(() => merge.node());

    defs.append(() => filter.node());
}

function appendBgPattern(selection, id, imageSize=500) {
    // let defsElem = d3.select('#map-defs');
    // if (defsElem.empty()) {
    //     defsElem = d3.select('body')
    //     .append('svg')
    //         .attr('width', 0)
    //         .attr('height', 0)
    //     .append('defs')
    //         .attr('id', 'map-defs');
    // }
    let defs = selection.select('defs');
    if (defs.empty()) defs = selection.append('defs')
    const existing = d3.select(`#${id}`);
    if (!existing.empty()) existing.remove();
    const pattern = defs.append('pattern')
        .attr('id', id)
        .attr('patternUnits', 'userSpaceOnUse')
        .attr('width', imageSize)
        .attr('height', imageSize);

    pattern.append('image')
        .attr('href', bg)
        .attr('x', 0).attr('y', 0)
        .attr('width', imageSize).attr('height', imageSize);

    pattern.append('rect')
        .attr('width', imageSize).attr('height', imageSize)
        .attr('fill', params.seaColor);
    defs.append(() => pattern.node());
    const clipPath = selection.append('clipPath').attr('id', 'clip');
    clipPath.append('rectangle')
        .attr('x', 0).attr('y', 0)
        .attr('width', 300).attr('height', 300)
        .attr('rx', 15)
    selection.append(() => clipPath.node());
}

let countries = null;
let land = null;
let providedBorders = null;
let provided = null;
let simpleLand = null;
fetch("https://cdn.jsdelivr.net/npm/world-atlas@2/countries-50m.json")
// fetch("https://cdn.jsdelivr.net/npm/world-atlas@2/land-10m.json")
// fetch("https://cdn.jsdelivr.net/npm/world-atlas@2/countries-10m.json")
    .then(response => response.json())
    // .then(world => topojson.merge(world, world.objects.countries.geometries.filter(feat => feat.id !== '462'))) // remove buggy maldives
    // .then(world => topojson.feature(world, world.objects.countries))
    // .then(world => topojson.feature(world, world.objects.land))
    .then(world => {
        simpleLand = topojsonSimplify.simplify(topojsonSimplify.presimplify(world), 0.001);
        simpleLand = topojson.merge(simpleLand, simpleLand.objects.countries.geometries);
        land = topojson.feature(world, world.objects.land);
        countries = topojson.feature(world, world.objects.countries);
        // console.log('land', land);
        // const newLand = splitMultiPolygons(land);
        land = splitMultiPolygons(land);
        console.log('land', land);
        console.log('countries', countries);
        draw();
    });


const container = d3.select('#container');
container.call(d3.drag()
    .on("drag", dragged)
);
container.call(d3.zoom()
    .scaleExtent([...bounds.altitude])
    .on("zoom", zoomed)
);

function zoomed(event) {
    params.altitude += event.sourceEvent.deltaY;
    draw(true);
    pane.refresh();
}

function dragged(event) {
    params.longitude += event.dx / 3;
    params.latitude -= event.dy / 3;
    draw(true);
    pane.refresh();

}

let path = null;
let projection = null;
let svg = null;
let addingPath = false;
let currentPath = [];
let providedPaths = [];

function draw(simplified = false) {
    const snyderP = 1.0 + params.altitude / earthRadius;
    const dY = params.altitude * Math.sin(params.tilt / degrees);
    const dZ = params.altitude * Math.cos(params.tilt / degrees);
    const visibleYextent = 2 * dZ * Math.tan(0.5 * params.fieldOfView / degrees);
    const yShift = dY * numPixelsY / visibleYextent;
    const scale = earthRadius * numPixelsY / visibleYextent;
    const tilt = params.tilt / degrees;
    const alpha = Math.acos(snyderP * Math.cos(tilt) * 0.999);
    const clipDistance = d3.geoClipCircle(Math.acos(1 / snyderP) - 1e-6);
    const preclip = alpha ? geoPipeline(
        clipDistance,
        geoRotatePhi(Math.PI + tilt),
        d3.geoClipCircle(Math.PI - alpha - 1e-4), // Extra safety factor needed for large tilt values
        geoRotatePhi(-Math.PI - tilt)
    ) : clipDistance;

    function geoRotatePhi(deltaPhi) {
        const cosDeltaPhi = Math.cos(deltaPhi);
        const sinDeltaPhi = Math.sin(deltaPhi);
        return sink => ({
            point(lambda, phi) {
                const cosPhi = Math.cos(phi);
                const x = Math.cos(lambda) * cosPhi;
                const y = Math.sin(lambda) * cosPhi;
                const z = Math.sin(phi);
                const k = z * cosDeltaPhi + x * sinDeltaPhi;
                sink.point(Math.atan2(y, x * cosDeltaPhi - z * sinDeltaPhi), Math.asin(k));
            },
            lineStart() { sink.lineStart(); },
            lineEnd() { sink.lineEnd(); },
            polygonStart() { sink.polygonStart(); },
            polygonEnd() { sink.polygonEnd(); },
            sphere() { sink.sphere(); }
        });
    }

    function geoPipeline(...transforms) {
        return sink => {
            for (let i = transforms.length - 1; i >= 0; --i) {
                sink = transforms[i](sink);
            }
            return sink;
        };
    }

    projection = geoSatellite()
        .scale(scale)
        .translate([(width / 2), (yShift + numPixelsY / 2)])
        .rotate([-params.longitude, -params.latitude, params.rotation])
        .tilt(params.tilt)
        .distance(snyderP)
        .preclip(preclip)
        .postclip(d3.geoClipRectangle(0, 0, width, numPixelsY))
        .precision(0.1)
    
    const container = d3.select('#map-container');
    container.html('');
    const outline = {type: "Sphere"};
    const graticule = d3.geoGraticule().step([params.graticuleStep, params.graticuleStep])();
    if (!params.useGraticule) graticule.coordinates = [];
    if (params.useCanvas || simplified) {
        let canvas = container.select('#canvas');
        if (canvas.empty()) canvas = container.append('canvas').attr('id', 'canvas').attr('width', width).attr('height', numPixelsY);
        const context = canvas.node().getContext('2d');
        context.clearRect(0, 0, width, numPixelsY);
        path = d3.geoPath(projection, context);
        context.fillStyle = "#88d";
        context.beginPath(), path(simplified ? simpleLand : land), context.fill();
        context.beginPath(),
            path(graticule),
            (context.strokeStyle = "#ddf"),
            (context.globalAlpha = 0.8),
            context.stroke();
        return context.canvas;
    }
    svg = container.select('svg');
    if (svg.empty()) svg = container.append('svg')
        .attr('viewBox', `${offCanvasPx} ${offCanvasPx} ${width - offCanvasPx * 2} ${numPixelsY  - offCanvasPx*2}`)
        .attr('xmlns', "http://www.w3.org/2000/svg")
        .attr('xmlns:xlink', "http://www.w3.org/1999/xlink")
        .attr('id', 'map');
        
    path = d3.geoPath(projection);
    svg.html('');
    svg.on("click", function(e) {
        const pos = projection.invert(d3.pointer(e));
        if (!addingPath) return;
        let elem = svg.select('#paths');
        if (elem.empty()) elem = svg.append('g').attr('id', 'paths');
        currentPath.push(pos);
        if (currentPath.length === 2) {
            const pathIndex = providedPaths.length;
            const id = `path-${pathIndex}`;
            const initialPath = `M${projection(currentPath[0])}L${projection(currentPath[1])}`;
            elem.append('path')
                .attr('id', id)
                .attr('d', initialPath);
            providedPaths.push(parseAndUnprojectPath(initialPath, projection));
            currentPath = [];
            addingPath = false;
            MotionPathHelper.editPath(`#${id}`, {
                onRelease: function() {
                    const parsed = parseAndUnprojectPath(this.path.getAttribute('d'), projection);
                    providedPaths[pathIndex] = parsed;
                }
            });
        }
    });
    

    const groupData = [];
    groupData.push({ name: 'outline', data: [outline], id: null, props: [], class: 'outline', filter: null });
    groupData.push({ name: 'graticule', data: [graticule], id: null, props: [], class: 'graticule', filter: null });
    if (params.land.show)
        groupData.push({ name: 'land', data: land, id: null, props: [], class: 'land', filter: params.land.filter });
    if (params.countries.show && countries)
        groupData.push({ name: 'countries', data: countries, id: { prefix: 'iso-3166-1-', field: 'id' }, props: ['name'], class: 'country', filter: null });
    if (providedBorders && params.providedBorders.show)
        groupData.push({ name: 'provided-borders', data: [providedBorders], id: null, props: [], class: 'provided-borders', filter: params.providedBorders.filter });
    if (provided && params.provided.show)
        groupData.push({ name: 'provided', data: provided, id: null, props: [], class: 'provided', filter: null });
        
    const groups = svg.selectAll('g').data(groupData).join('g').attr('id', d => d.name);
    
    function drawPaths(data) {
        const pathElem = d3.select(this).selectAll('path')
        .data(data.data.features ? data.data.features : data.data)
        .join('path')
        .attr('d', (d) => {return path(d)});
        if (data.id) pathElem.attr('id', (d) => data.id.prefix + d[data.id.field]);
        if (data.class) pathElem.attr('class', data.class);
        if (data.filter) pathElem.attr('filter', `url(#${data.filter})`);
        data.props.forEach((prop) => pathElem.attr(prop, (d) => d.properties[prop]))
    }
        
    groups.each(drawPaths);

    drawCustomPaths(providedPaths, svg, projection);
    // removeNotVisible();

    appendGlow(svg, 'firstGlow', params.firstGlow.innerGlow, params.firstGlow.outerGlow);
    appendGlow(svg, 'secondGlow', params.secondGlow.innerGlow, params.secondGlow.outerGlow);
    appendBgPattern(svg, 'noise');
    if (params.bgNoise)
        d3.select('#outline').style('fill', "url(#noise)");
    else d3.select('#outline').style('fill', "var(--sea-color)");

    styleLayer(params.countries, 'country');
    styleLayer(params.land, 'land');
    styleLayer(params.provided, 'provided');
    styleLayer(params.providedBorders, 'provided-borders');
    // console.log(`Size of lands (in chars): ${d3.select('#land').node().innerHTML.length}`);
        // setTimeout(() => {
    //     MotionPathHelper.editPath("#iso-3166-1-470");
    // }, 3000);
}
function styleLayer(styleParams, prefix) {
    const map = d3.select('#map');
    if (styleParams.fillColor)      map.style(`--${prefix}-fill-color`, styleParams.fillColor);
    if (styleParams.strokeColor)    map.style(`--${prefix}-stroke-color`, styleParams.strokeColor)
    if (styleParams.strokeWidth)    map.style(`--${prefix}-stroke-width`, styleParams.strokeWidth);

    if (styleParams.hover === undefined) return;
    if (styleParams.hover) map.style(`--${prefix}-hover-color`, styleParams.hoverColor);
    else map.style(`--${prefix}-hover-color`, styleParams.fillColor);
}

function fixOrder(geojson) {
    if (geojson.features) {
        const fixed = {...geojson};
        fixed.features = fixed.features.map(f =>
            rewind(f, { reverse: true })
        );
        return fixed;
    }
    return rewind(geojson);
}

function splitMultiPolygons(geojson) {
    const newGeojson = {type: 'FeatureCollection', features: []};
    geojson.features.forEach(feat => {
        if (feat.geometry.type == 'MultiPolygon') {
            feat.geometry.coordinates.map(coords => {
                newGeojson.features.push(
                {
                    type: 'Feature', 
                    properties: feat.properties, 
                    geometry: {
                        type: 'Polygon', coordinates: coords
                    }
                });
            });
        }
        else newGeojson.features.push(feat);
    });
    return newGeojson;
}

let providedProps = null;
function handleInput(e) {
    const file = e.target.files[0];
    const reader = new FileReader();
    reader.addEventListener('load', () => {
        // winding order matters !!!
        provided = fixOrder(JSON.parse(reader.result));
        providedBorders = provided.features.length > 1 ? provided.features.reduce((acc, cur)  => {
            return union(acc, cur);
        }, provided.features[0]) : provided.features[0];
        // const simpOptions = {tolerance: 0.005, highQuality: false};
        // providedBorders = simplify(buffer(providedBorders, 2), simpOptions); // buffer 1km and simplify
        const boundingBox = bbox(providedBorders);

        providedBorders = fixOrder({ type: "FeatureCollection", features: [{...providedBorders}] });
        
        params.longitude = (boundingBox[2] + boundingBox[0]) / 2;
        params.latitude = (boundingBox[3] + boundingBox[1]) / 2;
        params.tilt = 0;
        pane.refresh();
        draw();
    });
    reader.readAsText(file);
}

function addPath() {
    addingPath = true;
}

function exportSvg() {
    console.log(svg.node().outerHTML)
    const out = SVGO.optimize(svg.node().outerHTML, svgoConfig);
    console.log(out.data);
}

</script>

<h1 class="has-text-centered is-size-2"> Map builder </h1>
<div id="map-container"></div>

<div>
    <div class="file m-4">
        <label class="file-label">
            <input class="file-input" type="file" accept=".geojson,.json" on:change={handleInput}>
            <span class="file-cta">
                <span class="file-icon"> <img src={uploadIcon} alt="upload-geojson"> </span>
                <span class="file-label"> Select geojson </span>
            </span>
        </label>
    </div>
    <div class="button" on:click={addPath}> Add path </div>
    <div class="button" on:click={exportSvg}> Export </div>
</div>



<style>
/* :root {
    --sea-color: #dde9ff66; 
    --country-hover-color: #ffe1cc8e;
    --country-stroke-color: #ffe1cc8e;
    --country-stroke-width: 1;
    --country-fill-color: #fff;

    --land-fill-color: #fff;
    --land-stroke-color: #D1BEB0;
    --land-stroke-width: 1;

    --provided-borders-fill-color: #fff;
    --provided-borders-stroke-color: #D1BEB0;
    --provided-borders-stroke-width: 1;

    --provided-fill-color: #ffe1cc8e;
    --provided-hover-color: #ffe1cc8e;
    --provided-stroke-color: #D1BEB0;
    --provided-stroke-width: 1;
}

#map {
    background-repeat: repeat;
}
#paths {
    stroke: black;
    stroke-width: 5;
    fill: none;
}
.graticule {
    fill: none;
    stroke: #777;
    stroke-width: .5px;
    stroke-opacity: .5;
}
.land {
    stroke: var(--land-stroke-color);
    stroke-width: var(--land-stroke-width);
    fill: var(--land-fill-color);
}
.provided-borders {
    stroke: var(--provided-borders-stroke-color);
    stroke-width: var(--provided-borders-stroke-width);
    fill: var(--provided-borders-fill-color);
}
.country {
    stroke: var(--country-stroke-color);
    stroke-width: var(--country-stroke-width);
    fill: var(--country-fill-color);
}
.provided {
    stroke: var(--provided-stroke-color);
    stroke-width: var(--provided-stroke-width);
    fill: var(--provided-fill-color);
}
.country:hover {
    fill: var(--country-hover-color);
}
.provided:hover {
    fill: var(--provided-hover-color);
}
.tp-dfwv {
  min-width: 360px;
} */
</style>
