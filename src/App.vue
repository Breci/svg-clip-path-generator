<script setup>
import {ref} from "vue";

function prettifySVG(svgString) {
  return svgString
      .replace(/(<svg[^>]*>)/, '$1\n  ')
      .replace(/(<clipPath[^>]*>)/, '$1\n    ')
      .replace(/(<path[^>]*\/>)/, '$1')
      .replace(/<\/clipPath>/, '\n  </clipPath>')
      .replace(/<\/svg>/, '\n</svg>')
      .replace(/^\s*[\r\n]/gm, '');
}

/**
 * Converts SVG content to a normalized clipPath with objectBoundingBox units
 * @param {string} svgString - SVG file content as string
 * @returns {string} - SVG with normalized clipPath
 */
function convertToNormalizedClipPath(svgString) {
  // remove <?xml ... ?> if present
  svgString = svgString.replace(/<\?xml.*?\?>\s*/g, '');

  // remove defs section if present
  svgString = svgString.replace(/<defs[\s\S]*?<\/defs>\s*/g, '');

  // remove comments
  svgString = svgString.replace(/<!--[\s\S]*?-->\s*/g, '');

  // Parse SVG string using DOMParser (browser native API)
  const parser = new DOMParser();
  const doc = parser.parseFromString(svgString, 'image/svg+xml');
  const svg = doc.querySelector('svg');

  if (!svg) {
    throw new Error('Invalid SVG content');
  }

  // remove all attributes from svg
  Array.from(svg.attributes).forEach(attr => svg.removeAttribute(attr.name));

  // Check if clipPath already exists
  let clipPath = svg.querySelector('clipPath');

  if (!clipPath) {
    clipPath = doc.createElementNS('http://www.w3.org/2000/svg', 'clipPath');
    clipPath.setAttribute('clipPathUnits', 'objectBoundingBox');

    const shapes = svg.querySelectorAll('path, circle, rect, ellipse, polygon, polyline');

    shapes.forEach(shape => {
      const pathData = shapeToPath(shape);
      if (pathData) {
        const pathElement = doc.createElementNS('http://www.w3.org/2000/svg', 'path');
        pathElement.setAttribute('d', pathData);
        clipPath.appendChild(pathElement);
        shape.remove();
      }
    });

    svg.insertBefore(clipPath, svg.firstChild);
  } else {
    clipPath.setAttribute('clipPathUnits', 'objectBoundingBox');
  }

  const paths = clipPath.querySelectorAll('path');
  paths.forEach(path => {
    const originalPath = path.getAttribute('d');
    const normalizedPath = normalizePathCoordinates(originalPath);
    path.setAttribute('d', normalizedPath);
    path.removeAttribute('transform');
  });

  // pretty print the SVG output
  const serializer = new XMLSerializer();
  const output = serializer.serializeToString(svg)
      .replace(/ xmlns="http:\/\/www\.w3\.org\/2000\/svg"/, '');

  return prettifySVG(output);
}

/**
 * Converts various SVG shapes to path data
 */
function shapeToPath(element) {
  const tag = element.tagName.toLowerCase();

  switch(tag) {
    case 'path':
      return element.getAttribute('d');

    case 'circle': {
      const cx = parseFloat(element.getAttribute('cx') || 0);
      const cy = parseFloat(element.getAttribute('cy') || 0);
      const r = parseFloat(element.getAttribute('r') || 0);
      return circleToPath(cx, cy, r);
    }

    case 'rect': {
      const x = parseFloat(element.getAttribute('x') || 0);
      const y = parseFloat(element.getAttribute('y') || 0);
      const width = parseFloat(element.getAttribute('width') || 0);
      const height = parseFloat(element.getAttribute('height') || 0);
      return `M${x},${y} L${x+width},${y} L${x+width},${y+height} L${x},${y+height} Z`;
    }

    case 'ellipse': {
      const cx = parseFloat(element.getAttribute('cx') || 0);
      const cy = parseFloat(element.getAttribute('cy') || 0);
      const rx = parseFloat(element.getAttribute('rx') || 0);
      const ry = parseFloat(element.getAttribute('ry') || 0);
      return ellipseToPath(cx, cy, rx, ry);
    }

    case 'polygon':
    case 'polyline': {
      const points = element.getAttribute('points').trim().split(/[\s,]+/);
      let path = `M${points[0]},${points[1]}`;
      for (let i = 2; i < points.length; i += 2) {
        path += ` L${points[i]},${points[i+1]}`;
      }
      if (tag === 'polygon') path += ' Z';
      return path;
    }

    default:
      return null;
  }
}

/**
 * Convert circle to path
 */
function circleToPath(cx, cy, r) {
  return `M${cx-r},${cy} A${r},${r} 0 1,0 ${cx+r},${cy} A${r},${r} 0 1,0 ${cx-r},${cy} Z`;
}

/**
 * Convert ellipse to path
 */
function ellipseToPath(cx, cy, rx, ry) {
  return `M${cx-rx},${cy} A${rx},${ry} 0 1,0 ${cx+rx},${cy} A${rx},${ry} 0 1,0 ${cx-rx},${cy} Z`;
}

/**
 * Normalize path coordinates to 0-1 range for objectBoundingBox
 */
function normalizePathCoordinates(pathString) {
  // Extract all numeric values from the path
  const numbers = [];
  const numberRegex = /-?\d+\.?\d*/g;
  let match;

  // Store positions of numbers for later replacement
  const positions = [];
  while ((match = numberRegex.exec(pathString)) !== null) {
    numbers.push(parseFloat(match[0]));
    positions.push({ start: match.index, end: match.index + match[0].length });
  }

  if (numbers.length === 0) return pathString;

  // Separate x and y coordinates (assuming they alternate)
  const xCoords = numbers.filter((_, i) => i % 2 === 0);
  const yCoords = numbers.filter((_, i) => i % 2 === 1);

  // Find bounds
  const minX = Math.min(...xCoords);
  const maxX = Math.max(...xCoords);
  const minY = Math.min(...yCoords);
  const maxY = Math.max(...yCoords);

  const width = maxX - minX;
  const height = maxY - minY;

  // Prevent division by zero
  if (width === 0 || height === 0) return pathString;

  // Normalize coordinates
  const normalizedNumbers = numbers.map((num, index) => {
    const isX = index % 2 === 0;
    if (isX) {
      return ((num - minX) / width).toFixed(6);
    } else {
      return ((num - minY) / height).toFixed(6);
    }
  });

  // Rebuild path string with normalized values
  let result = pathString;
  let offset = 0;

  positions.forEach((pos, index) => {
    const oldValue = pathString.substring(pos.start, pos.end);
    const newValue = normalizedNumbers[index];
    const before = result.substring(0, pos.start + offset);
    const after = result.substring(pos.end + offset);
    result = before + newValue + after;
    offset += newValue.length - oldValue.length;
  });

  return result;
}

const fileInput = ref(null);
const normalizedSVG = ref('')

function onFileChange(event) {
  const file = event.target.files[0];
  console.log(file)
  if (file && file.type === 'image/svg+xml') {
    fileInput.value = file;
    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const svgString = e.target.result;
        normalizedSVG.value = convertToNormalizedClipPath(svgString);
        
        console.log('Normalized SVG ClipPath:\n', normalizedSVG.value);
      } catch (error) {
        console.error('Error processing SVG:', error);
      }
    };
    reader.readAsText(file);
  } else {
    alert('Please select a valid SVG file.');
    normalizedSVG.value = '';
  }
}

function onDownloadFile() {
  if (!normalizedSVG.value) return;

  const blob = new Blob([normalizedSVG.value], { type: 'image/svg+xml' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = fileInput.value ? `clipPath-${fileInput.value.name}` : 'clipPath.svg';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}
</script>

<template>
  <div>
    <div style="margin-bottom: 2rem;">
      <div>
        SVG File
       
      </div>
      <input type="file" @change="onFileChange" accept=".svg" />
    </div>
    <div v-if="normalizedSVG" style="color:#4aaf50; margin-bottom: 0.5rem; font-weight: bold; font-size: 0.8rem">
      File ready!
    </div>
    <div>
      <button :disabled="!normalizedSVG" @click="onDownloadFile">Download</button>
    </div>
  </div>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
