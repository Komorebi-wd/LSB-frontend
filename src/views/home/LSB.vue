<template>
  <div id="app-container">
    <header>
      <h1>图像隐写工具</h1>
      <div class="menu">
        <button @click="loadImage">读取图片</button>
        <button @click="embedData">嵌入</button>
        <button @click="saveImage">存储图片</button>
        <button @click="extractData">提取</button>
      </div>
    </header>
    <div class="top-info">
      可嵌入信息的最大长度：{{ maxEmbeddableLength }} 字节
    </div>
    <main>
      <div class="images">
        <div class="image-container">
          <h3>原始图片</h3>
          <img v-if="originalImage" :src="originalImage" alt="原始图片">
        </div>
        <div class="image-container">
          <h3>嵌入信息后的图片</h3>
          <img v-if="modifiedImage" :src="modifiedImage" alt="修改后的图片">
        </div>
      </div>
      <div class="input-area">
        <textarea v-model="message" placeholder="输入要嵌入的信息 / 显示提取出的信息"></textarea>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import axios from 'axios';

const originalImage = ref(null);
const modifiedImage = ref(null);
const message = ref('');
const maxEmbeddableLength = ref(0);
let originalFile = null;

function loadImage() {
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = 'image/*';
  input.onchange = e => {
    const file = e.target.files[0];
    if (file && file.type.startsWith('image')) {
      const reader = new FileReader();
      reader.onload = (e) => {
        originalImage.value = e.target.result;
        originalFile = file;
        calculateMaxEmbeddableLength(file);
      };
      reader.readAsDataURL(file);
    }
  };
  input.click();
}

function calculateMaxEmbeddableLength(file) {
  const formData = new FormData();
  formData.append('image', file);

  axios.post('/api/lsb/upload', formData)
      .then(response => {
        maxEmbeddableLength.value = response.data;
      })
      .catch(error => {
        console.error('Error calculating max embeddable length:', error);
      });
}

function embedData() {
  if (!originalFile || !message.value) {
    alert('请先选择图片并输入要嵌入的信息');
    return;
  }

  const formData = new FormData();
  formData.append('image', originalFile);
  formData.append('message', message.value);

  axios.post('/api/lsb/embed', formData)
      .then(response => {
        modifiedImage.value = `data:image/bmp;base64,${response.data}`;
      })
      .catch(error => {
        console.error('Error embedding data:', error);
      });
}

function saveImage() {
  if (!modifiedImage.value) {
    alert('没有可保存的嵌入信息图像');
    return;
  }

  const link = document.createElement('a');
  link.href = modifiedImage.value;
  link.download = 'embedded_image.bmp';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function extractData() {
  if (!originalFile) {
    alert('请先选择嵌入了信息的图片');
    return;
  }

  const formData = new FormData();
  formData.append('image', originalFile);

  axios.post('/api/lsb/extract', formData)
      .then(response => {
        message.value = response.data;
      })
      .catch(error => {
        console.error('Error extracting data:', error);
      });
}
</script>

<style scoped>
#app-container {
  max-width: 1200px;
  margin: auto;
  text-align: center;

  header {
    background-color: #007BFF;
    color: white;
    padding: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;

    .menu button {
      padding: 10px 20px;
      margin: 0 5px;
      cursor: pointer;
    }
  }

  .top-info {
    margin: 20px 0;
    font-size: 16px;
    color: #333;
  }

  .images {
    display: flex;
    justify-content: space-around;
    margin-bottom: 20px;
  }

  .image-container {
    width: 45%;
    border: 1px solid #ccc;
    padding: 10px;
    margin: 0 5px;

    h3 {
      color: #333;
      margin-bottom: 10px;
    }

    img {
      max-width: 100%;
      height: auto;
    }
  }

  .input-area {
    margin-top: 20px;
    textarea {
      width: 90%;
      height: 100px;
    }
  }
}
</style>
