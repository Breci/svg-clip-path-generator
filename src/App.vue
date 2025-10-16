<script setup>
import { ref } from "vue";
import svgpath from 'svgpath';

/**
 * Calculate bounding box from path segments
 * @param {Array} segments - Path segments from svgpath
 * @returns {Object} - {minX, minY, maxX, maxY}
 */
function calculateBounds(segments) {
  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;

  let currentX = 0;
  let currentY = 0;

  segments.forEach(segment => {
    const cmd = segment[0];

    // Extract coordinates based on command type
    switch (cmd) {
      case 'M':
      case 'L':
        currentX = segment[1];
        currentY = segment[2];
        minX = Math.min(minX, currentX);
        minY = Math.min(minY, currentY);
        maxX = Math.max(maxX, currentX);
        maxY = Math.max(maxY, currentY);
        break;

      case 'H':
        currentX = segment[1];
        minX = Math.min(minX, currentX);
        maxX = Math.max(maxX, currentX);
        break;

      case 'V':
        currentY = segment[1];
        minY = Math.min(minY, currentY);
        maxY = Math.max(maxY, currentY);
        break;

      case 'C':
        // Cubic bezier: control point 1, control point 2, end point
        minX = Math.min(minX, segment[1], segment[3], segment[5]);
        minY = Math.min(minY, segment[2], segment[4], segment[6]);
        maxX = Math.max(maxX, segment[1], segment[3], segment[5]);
        maxY = Math.max(maxY, segment[2], segment[4], segment[6]);
        currentX = segment[5];
        currentY = segment[6];
        break;

      case 'S':
        // Smooth cubic bezier: control point 2, end point
        minX = Math.min(minX, segment[1], segment[3]);
        minY = Math.min(minY, segment[2], segment[4]);
        maxX = Math.max(maxX, segment[1], segment[3]);
        maxY = Math.max(maxY, segment[2], segment[4]);
        currentX = segment[3];
        currentY = segment[4];
        break;

      case 'Q':
        // Quadratic bezier: control point, end point
        minX = Math.min(minX, segment[1], segment[3]);
        minY = Math.min(minY, segment[2], segment[4]);
        maxX = Math.max(maxX, segment[1], segment[3]);
        maxY = Math.max(maxY, segment[2], segment[4]);
        currentX = segment[3];
        currentY = segment[4];
        break;

      case 'T':
        // Smooth quadratic bezier: end point only
        minX = Math.min(minX, segment[1]);
        minY = Math.min(minY, segment[2]);
        maxX = Math.max(maxX, segment[1]);
        maxY = Math.max(maxY, segment[2]);
        currentX = segment[1];
        currentY = segment[2];
        break;

      case 'A':
        // Arc: we'll use the end point (approximation)
        currentX = segment[6];
        currentY = segment[7];
        minX = Math.min(minX, currentX);
        minY = Math.min(minY, currentY);
        maxX = Math.max(maxX, currentX);
        maxY = Math.max(maxY, currentY);
        break;

      case 'Z':
      case 'z':
        // Close path - no new coordinates
        break;
    }
  });

  return { minX, minY, maxX, maxY };
}

/**
 * Normalize SVG path data to 0-1 range for objectBoundingBox
 * @param {string} pathString - SVG path data (d attribute)
 * @returns {string} - Normalized path data with coordinates between 0 and 1
 */
function normalizePathTo01(pathString) {
  if (!pathString || pathString.trim() === '') {
    return pathString;
  }

  try {
    const path = svgpath(pathString)
        .abs()      // Convert all commands to absolute coordinates
        .unarc()    // Convert arcs to cubic bezier curves
        .unshort(); // Expand shorthand commands

    // Get path segments
    const segments = path.segments;

    if (!segments || segments.length === 0) {
      return pathString;
    }

    // Calculate bounds
    const bounds = calculateBounds(segments);
    const { minX, minY, maxX, maxY } = bounds;
    const width = maxX - minX;
    const height = maxY - minY;

    // Handle edge cases
    if (width === 0 && height === 0) {
      return 'M0.5,0.5';
    }

    if (width === 0) {
      return path
          .translate(-minX, -minY)
          .scale(1, 1 / height)
          .translate(0.5, 0)
          .round(6)
          .toString();
    }

    if (height === 0) {
      return path
          .translate(-minX, -minY)
          .scale(1 / width, 1)
          .translate(0, 0.5)
          .round(6)
          .toString();
    }

    // Normal case
    return path
        .translate(-minX, -minY)
        .scale(1 / width, 1 / height)
        .round(6)
        .toString();

  } catch (error) {
    console.error('Error normalizing path:', error);
    return pathString;
  }
}

/**
 * Pretty print SVG output
 */
function prettifySVG(svgString) {
  return svgString
      .replace(/(<svg[^>]*>)/, '$1\n  ')
      .replace(/(<clipPath[^>]*>)/, '$1\n    ')
      .replace(/(<path[^>]*\/>)/g, '$1\n    ')
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
  // Clean up the SVG string
  svgString = svgString
      .replace(/<\?xml.*?\?>\s*/g, '')      // Remove XML declaration
      .replace(/<defs[\s\S]*?<\/defs>\s*/g, '') // Remove defs
      .replace(/<!--[\s\S]*?-->\s*/g, '');  // Remove comments

  // Parse SVG
  const parser = new DOMParser();
  const doc = parser.parseFromString(svgString, 'image/svg+xml');
  const svg = doc.querySelector('svg');

  if (!svg) {
    throw new Error('Invalid SVG content');
  }

  // Remove all SVG attributes
  Array.from(svg.attributes).forEach(attr => svg.removeAttribute(attr.name));

  // Get or create clipPath
  let clipPath = svg.querySelector('clipPath');

  if (!clipPath) {
    clipPath = doc.createElementNS('http://www.w3.org/2000/svg', 'clipPath');
    clipPath.setAttribute('clipPathUnits', 'objectBoundingBox');

    // Get all path elements
    const paths = svg.querySelectorAll('path');

    if (paths.length === 0) {
      throw new Error('No path elements found in SVG');
    }

    // Move paths to clipPath
    paths.forEach(path => {
      const pathData = path.getAttribute('d');
      if (pathData) {
        const pathElement = doc.createElementNS('http://www.w3.org/2000/svg', 'path');
        pathElement.setAttribute('d', pathData);
        clipPath.appendChild(pathElement);
      }
      path.remove();
    });

    svg.insertBefore(clipPath, svg.firstChild);
  } else {
    clipPath.setAttribute('clipPathUnits', 'objectBoundingBox');
  }

  // Normalize all paths in clipPath
  const paths = clipPath.querySelectorAll('path');
  paths.forEach(path => {
    const originalPath = path.getAttribute('d');
    const normalizedPath = normalizePathTo01(originalPath);
    path.setAttribute('d', normalizedPath);

    // Remove transform and other attributes
    Array.from(path.attributes).forEach(attr => {
      if (attr.name !== 'd') {
        path.removeAttribute(attr.name);
      }
    });
  });

  // Serialize and clean output
  const serializer = new XMLSerializer();
  const output = serializer.serializeToString(svg)
      .replace(/ xmlns="http:\/\/www\.w3\.org\/2000\/svg"/, '');

  return prettifySVG(output);
}

const fileInput = ref(null);
const normalizedSVG = ref('');

function onFileChange(event) {
  const file = event.target.files[0];

  if (!file) {
    return;
  }

  if (file.type !== 'image/svg+xml') {
    alert('Please select a valid SVG file.');
    normalizedSVG.value = '';
    return;
  }

  fileInput.value = file;

  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const svgString = e.target.result;
      normalizedSVG.value = convertToNormalizedClipPath(svgString);
      console.log('Normalized SVG ClipPath:\n', normalizedSVG.value);
    } catch (error) {
      console.error('Error processing SVG:', error);
      alert(`Error: ${error.message}`);
      normalizedSVG.value = '';
    }
  };
  reader.readAsText(file);
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
      <div style="margin-bottom: 0.5rem;">
        SVG File (Only paths are supported currently)
      </div>
      <input type="file" @change="onFileChange" accept=".svg" />
    </div>

    <div
        v-if="normalizedSVG"
        style="color: #4aaf50; margin-bottom: 0.5rem; font-weight: bold; font-size: 0.8rem"
    >
      File ready!
    </div>

    <div>
      <button :disabled="!normalizedSVG" @click="onDownloadFile">
        Download
      </button>
    </div>
  </div>
</template>

<style scoped>
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>