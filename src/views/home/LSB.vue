<template>
  <div id="app-container">
    <header>
      <h1>图像隐写工具</h1>
      <div class="menu">
        <button @click="loadImage">读取图片</button>
        <button @click="embedData">嵌入</button>
        <button @click="saveImage">存储图片</button>
        <button @click="extractData">提取</button>
        <button @click="addNoise">添加高斯噪声</button>
      </div>
    </header>
    <div class="top-info" v-if="useKey">
      <div class="key-area">
        <input v-model="encryptionKey" placeholder="输入16位密钥" maxlength="16">
        <button @click="generateRandomKey">生成随机密钥</button>
      </div>
      可嵌入信息的最大长度：{{ maxEmbeddableLength }} 字节
    </div>
    <div class="top-info" v-else>
      可嵌入信息的最大长度：{{ maxEmbeddableLength }} 字节
    </div>
    <main>
      <div class="images">
        <div class="image-container">
          <h3>原始图片</h3>
          <img v-if="originalImage" :src="originalImage" alt="原始图片">
        </div>
        <div class="image-container">
          <h3 v-if="!useNoise">嵌入信息后的图片</h3>
          <h3 v-else>添加噪声后的图片</h3>
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
const encryptionKey = ref('');
const useKey = ref(false);
const useNoise = ref(false);
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
        showKeyOptionDialog();
      };
      reader.readAsDataURL(file);
    }
  };
  input.click();
}

function showKeyOptionDialog() {
  const useKeyChoice = confirm("是否使用密钥进行信息嵌入和提取？");
  useKey.value = useKeyChoice;
  calculateMaxEmbeddableLength(originalFile, useKeyChoice);
}

function calculateMaxEmbeddableLength(file, useKey) {
  const formData = new FormData();
  formData.append('image', file);

  const url = useKey ? '/api/lsb/uploadWithKey' : '/api/lsb/upload';

  axios.post(url, formData)
    .then(response => {
      maxEmbeddableLength.value = response.data;
    })
    .catch(error => {
      console.error('Error calculating max embeddable length:', error);
    });
}

function embedData() {
  if (!originalFile) {
    alert('请先选择图片');
    return;
  }
  if (!message.value) {
    alert('请输入要嵌入的信息');
    return;
  }
  if (message.value.length > maxEmbeddableLength.value) {
    alert('信息太长，无法嵌入到图片中');
    return;
  }
  if (useKey.value && encryptionKey.value.length !== 16) {
    alert('请输入16位密钥');
    return;
  }

  const formData = new FormData();
  formData.append('image', originalFile);
  formData.append('message', message.value);
  if (useKey.value) {
    formData.append('key', encryptionKey.value);
  }

  const url = useKey.value ? '/api/lsb/embedWithKey' : '/api/lsb/embed';

  axios.post(url, formData)
    .then(response => {
      useNoise.value = false;
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
  if (useKey.value && encryptionKey.value.length !== 16) {
    alert('请输入16位密钥');
    return;
  }

  const formData = new FormData();
  formData.append('image', originalFile);
  if (useKey.value) {
    formData.append('key', encryptionKey.value);
  }

  const url = useKey.value ? '/api/lsb/extractWithKey' : '/api/lsb/extract';

  axios.post(url, formData)
    .then(response => {
      message.value = response.data;
    })
    .catch(error => {
      alert('提取信息失败，请检查密钥是否正确');
    });
}

function generateRandomKey() {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < 16; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  encryptionKey.value = result;
}

function addNoise() {
  if (!originalFile) {
    alert('请先选择需要添加噪声的图片');
    return;
  }

  const formData = new FormData();
  formData.append('image', originalFile);

  axios.post('/api/lsb/noise', formData)
    .then(response => {
      useNoise.value = true;
      modifiedImage.value = `data:image/bmp;base64,${response.data}`;
    })
    .catch(error => {
      console.error('Error adding noise data:', error);
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
    display: flex;
    align-items: center;
    justify-content: center;

    .key-area {
      display: flex;
      align-items: center;
      margin-right: 20px;

      input {
        width: 200px;
        padding: 5px;
        margin-right: 10px;
      }

      button {
        padding: 5px 10px;
        cursor: pointer;
      }
    }
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