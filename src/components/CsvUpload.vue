<template>
  <div>
    <h2>Upload File</h2>
    <input type="file" ref="fileInput" @change="handleFileChange"/>
    <button @click="startUpload">Start Upload</button>
    <div v-if="uploadProgress > 0">
      <progress :value="uploadProgress" max="100"></progress>
    </div>
    <div v-if="uploadResult">
      <h3>Upload Result</h3>
      <p>Correct Records: {{ uploadResult.correct }}</p>
      <p>Incorrect Records: {{ uploadResult.incorrect }}</p>
      <p>{{ finishUpload }}</p>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      selectedFile: null,
      uploadProgress: 0,
      chunkSize: 1024 * 1024, // 1 MB chunk size
      uploadResult: null,
      correctChunkResult: null,
      incorrectChunkResult: null,
      finishUpload: null,
    };
  },
  methods: {
    handleFileChange(event) {
      this.selectedFile = event.target.files[0];
    },
    async startUpload() {
      if (!this.selectedFile) {
        return;
      }

      const totalChunks = Math.ceil(this.selectedFile.size / this.chunkSize);
      let currentChunk = 0;

      try {
        while (currentChunk < totalChunks) {
          const start = currentChunk * this.chunkSize;
          const end = Math.min(start + this.chunkSize, this.selectedFile.size);
          const chunk = this.selectedFile.slice(start, end);
          const chunkFileName = `${this.selectedFile.name}_${currentChunk}`;
          const formData = new FormData();

          formData.append('fileChunk', chunk, chunkFileName);

          const chunkResponse = await axios.post('http://sellerassistantapp.lc/api/upload', formData, {
            headers: {
              'Content-Type': 'multipart/form-data',
            },
            onUploadProgress: (progressEvent) => {
              this.uploadProgress = Math.round(
                  (progressEvent.loaded / progressEvent.total) * 100
              );
            },
          });

          this.correctChunkResult += chunkResponse.data.correct;
          this.incorrectChunkResult += chunkResponse.data.incorrect;
          this.uploadResult = {correct: this.correctChunkResult, incorrect: this.incorrectChunkResult};
          currentChunk++;
        }

        const assembleResponse = await axios.post('http://sellerassistantapp.lc/api/assemble', {
          totalChunks: totalChunks,
          filename: this.selectedFile.name,
        });
        this.finishUpload = assembleResponse.data.message;
      } catch (error) {
        console.error(error);
      }

      this.uploadProgress = 0;
    },
  },
};
</script>
