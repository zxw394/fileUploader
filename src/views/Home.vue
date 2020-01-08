<template>
<div>
  <input type="file" multiple  id="fileContainer"/>
  <button @click="getFiles">获取文件</button>
</div>
</template>

<script>
  let chunkSize = 10 * 1024 * 1024;
  function File () {

  }
  export default {
    data() {
      return {
      }
    },
    methods: {
      getFiles () {
        let files = document.querySelector("#fileContainer").files;
        for (let file of files) {
          console.log(file);
        }

      },
    },
    mounted () {
    }
  }
</script>
