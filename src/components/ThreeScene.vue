<template>
  <div class="app-container">
    <header class="app-header">
      <h1 class="app-title">3D Text Animator</h1>
      <p class="app-description">
        Create animated 3D text scenes with customizable options for Instagram Reels.
      </p>
    </header>
    <div class="controls">
      <input v-model="titleText" @input="updateText" placeholder="Enter title here" class="title-input" />
      <textarea v-model="inputText" @input="updateText" placeholder="Enter text here" class="text-input"></textarea>
      <div class="control-group">
        <label for="lineIntervalInput">Line Interval (ms):</label>
        <input id="lineIntervalInput" type="number" v-model.number="lineInterval" class="number-input" />
      </div>
      <div class="control-group">
        <label for="maxLinesInput">Max Lines per Group:</label>
        <input id="maxLinesInput" type="number" v-model.number="maxLines" class="number-input" />
      </div>
      <div class="control-group">
        <label for="textColorInput">Text Color:</label>
        <input id="textColorInput" type="color" v-model="textColor" class="color-input" />
      </div>
      <div class="control-group">
        <label for="bgColorInput">Background Color:</label>
        <input id="bgColorInput" type="color" v-model="bgColor" class="color-input" @input="updateBackgroundColor"/>
      </div>
      <div class="button-group">
        <button @click="startCapture" class="capture-button">Start Capture</button>
        <button @click="stopCapture" class="stop-button">Stop Capture</button>
      </div>
    </div>
    <div ref="threeContainer" class="three-container"></div>
  </div>
</template>
<script>
import { onMounted, ref, watch } from 'vue';
import * as THREE from 'three';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
import helvetiker from 'three/examples/fonts/helvetiker_regular.typeface.json';

export default {
  setup() {
    const threeContainer = ref(null);
    let renderer, scene, camera, titleMesh, mediaRecorder, chunks = [];
    let textMeshes = [];
    const titleText = ref('Title Text');
    const inputText = ref('Hello Three.js!\nThis is a sample text.\nIt spans multiple lines.');
    const lineInterval = ref(500);
    const maxLines = ref(6);
    const textColor = ref('#ffffff'); // Default white text color
    const bgColor = ref('#343a40'); // Default dark background color
    let currentLineGroupIndex = 0;
    let lines = [];
    let intervalId = null;
    let font;

    const width = 1080;
    const height = 1920;
    const displayWidth = width / 2;
    const displayHeight = height / 2;
    const fixedFontSize = 55;
    const titleFontSize = 100;
    const titleHeightPercentage = 0.4;
    const titleHeight = height * titleHeightPercentage;
    const lineDisplayDuration = 3000;

    const initScene = () => {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(width, height);
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.domElement.style.width = `${displayWidth}px`;
      renderer.domElement.style.height = `${displayHeight}px`;
      threeContainer.value.appendChild(renderer.domElement);

      addBackground();
      addImage();

      const loader = new FontLoader();
      font = loader.parse(helvetiker);
      lines = inputText.value.split('\n');
      createTitleMesh();
      createTextMeshes();

      camera.position.z = 1000;
      renderScene();
    };

    const addBackground = () => {
      const innerMaterial = new THREE.MeshBasicMaterial({ color: bgColor.value });
      const innerGeometry = new THREE.PlaneGeometry(width, height);
      const innerMesh = new THREE.Mesh(innerGeometry, innerMaterial);
      scene.add(innerMesh);
    };

    const updateBackgroundColor = () => {
      scene.children.forEach((child) => {
        if (child.isMesh && child.material.color.getHexString() === bgColor.value.replace('#', '')) {
          child.material.color.set(bgColor.value);
        }
      });
      renderScene();
    };

    const addImage = () => {
      const textureLoader = new THREE.TextureLoader();
      const imagePath = require('@/assets/img.png');
      textureLoader.load(imagePath, (texture) => {
        const imageMaterial = new THREE.MeshBasicMaterial({ map: texture });
        const imageWidth = 800;
        const imageHeight = (texture.image.height / texture.image.width) * imageWidth;
        const imageGeometry = new THREE.PlaneGeometry(imageWidth, imageHeight);
        const imageMesh = new THREE.Mesh(imageGeometry, imageMaterial);
        imageMesh.position.y = -height / 2 + imageHeight / 2 + 225;
        imageMesh.position.z = 0.1;
        scene.add(imageMesh);
      });
    };

    const createTitleMesh = () => {
      if (titleMesh) {
        scene.remove(titleMesh);
        titleMesh.geometry.dispose();
        titleMesh.material.dispose();
      }

      const material = new THREE.MeshBasicMaterial({ color: textColor.value });
      const geometry = new TextGeometry(titleText.value, {
        font: font,
        size: titleFontSize,
        depth: 1, 
        curveSegments: 24,
        bevelEnabled: true,
        bevelThickness: 1,
        bevelSize: 0.5,
        bevelOffset: 0,
        bevelSegments: 8
      });

      geometry.computeBoundingBox();
      const titleWidth = geometry.boundingBox.max.x - geometry.boundingBox.min.x;

      titleMesh = new THREE.Mesh(geometry, material);
      titleMesh.position.x = -titleWidth / 2;
      titleMesh.position.y = height / 2 - titleHeight / 2;
      scene.add(titleMesh);
    };

    const createTextMeshes = () => {
      textMeshes.forEach(mesh => {
        scene.remove(mesh);
        mesh.geometry.dispose();
        mesh.material.dispose();
      });

      textMeshes = [];
      const material = new THREE.MeshBasicMaterial({ color: textColor.value });

      const startLine = currentLineGroupIndex * maxLines.value;
      const endLine = Math.min(startLine + maxLines.value, lines.length);

      for (let i = startLine; i < endLine; i++) {
        const line = lines[i];
        const geometry = new TextGeometry(line, {
          font: font,
          size: fixedFontSize,
          depth: 1,
          curveSegments: 24,
          bevelEnabled: true,
          bevelThickness: 1,
          bevelSize: 0.5,
          bevelOffset: 0,
          bevelSegments: 8
        });

        geometry.computeBoundingBox();
        const lineWidth = geometry.boundingBox.max.x - geometry.boundingBox.min.x;

        let xPosition = -lineWidth / 2;
        let yPosition = height / 2 - titleHeight - 50 - (i - startLine) * (fixedFontSize + 30);

        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.x = xPosition;
        mesh.position.y = yPosition;
        mesh.visible = false;
        textMeshes.push(mesh);
        scene.add(mesh);
      }
    };

    const renderScene = () => {
      renderer.render(scene, camera);
    };

    const updateText = () => {
      lines = inputText.value.split('\n');
      currentLineGroupIndex = 0;
      createTitleMesh();
      createTextMeshes();
      renderScene();
    };

    const startCapture = () => {
      chunks = [];
      const stream = renderer.domElement.captureStream(30);
      mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm' });
      mediaRecorder.ondataavailable = (e) => chunks.push(e.data);
      mediaRecorder.onstop = () => {
        const blob = new Blob(chunks, { type: 'video/webm' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.style.display = 'none';
        a.href = url;
        a.download = 'capture.webm';
        document.body.appendChild(a);
        a.click();
        window.URL.revokeObjectURL(url);
      };
      mediaRecorder.start();
      currentLineGroupIndex = -1;
      showNextLineGroup();
    };

    const stopCapture = () => {
      if (mediaRecorder && mediaRecorder.state !== "inactive") {
        mediaRecorder.stop();
      }
      clearInterval(intervalId);
    };

    const showNextLineGroup = () => {
      currentLineGroupIndex++;
      if (currentLineGroupIndex * maxLines.value >= lines.length) {
        currentLineGroupIndex = 0;
      }

      createTextMeshes();

      let lineIndex = 0;

      intervalId = setInterval(() => {
        if (lineIndex < textMeshes.length) {
          textMeshes[lineIndex].visible = true;
          renderScene();
          lineIndex++;
        } else {
          clearInterval(intervalId);
          setTimeout(showNextLineGroup, lineDisplayDuration);
        }
      }, lineInterval.value);
    };

    onMounted(() => {
      initScene();
    });

    watch(textColor, updateText);
    watch(bgColor, updateBackgroundColor);

    return {
      threeContainer,
      titleText,
      inputText,
      lineInterval,
      maxLines,
      textColor,
      bgColor,
      startCapture,
      stopCapture,
      updateText,
    };
  },
};
</script>
<style>
.app-container {
  font-family: 'Arial', sans-serif;
  text-align: center;
  color: #333;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.app-header {
  background-color: #007bff;
  padding: 20px;
  margin-bottom: 20px;
  color: white;
  border-radius: 10px;
}

.app-title {
  font-size: 32px;
  margin: 0;
  font-weight: bold;
}

.app-description {
  font-size: 16px;
  margin: 10px 0 0;
}

.three-container {
  width: 540px;
  height: 960px;
  overflow: hidden;
  position: relative;
  margin: 20px auto;
  background-color: #343a40;
  border: 5px solid #007bff;
  border-radius: 10px;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin: 0 auto;
  width: 100%;
}

.control-group {
  display: flex;
  flex-direction: column;
}

.number-input,
.title-input,
.text-input,
.color-input {
  width: 100%;
  padding: 12px;
  font-size: 16px;
  border: 1px solid #ced4da;
  border-radius: 5px;
  transition: border-color 0.3s;
}

.number-input:focus,
.title-input:focus,
.text-input:focus,
.color-input:focus {
  border-color: #007bff;
  outline: none;
}

.text-input {
  height: 120px;
}

.button-group {
  display: flex;
  justify-content: space-between;
}

.capture-button,
.stop-button {
  padding: 12px;
  font-size: 18px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  width: 48%;
  transition: background-color 0.3s;
}

.capture-button:hover {
  background-color: #0056b3;
}

.stop-button {
  background-color: #dc3545;
}

.stop-button:hover {
  background-color: #c82333;
}
</style>
