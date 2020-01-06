<template>
  <div class="about">
<!--    <el-upload-->
<!--            action="https://jsonplaceholder.typicode.com/posts/"-->
<!--            list-type="picture-card"-->
<!--            :on-preview="handlePictureCardPreview"-->
<!--            :on-remove="handleRemove">-->
<!--      <i class="el-icon-plus"></i>-->
<!--    </el-upload>-->
<!--    <el-dialog :visible.sync="dialogVisible">-->
<!--      <img width="100%" :src="dialogImageUrl" alt="">-->
<!--    </el-dialog>-->
    <uploader :options="options" class="uploader-example"
              ref="uploader"
              :autoStart="false"
              :file-status-text="statusText"
              @file-added="onFileAdded"
              @file-success="onFileSuccess"
              @file-progress="onFileProgress"
              @file-error="onFileError"
              @file-complete="onFileComplete">
      <uploader-unsupport></uploader-unsupport>
      <uploader-drop class="uploader-drop-box">
        <p>将文件拖到此处，或点击上传</p>
        <uploader-btn :attrs="attrs" ref="uploaderBtn"><el-icon class="el-icon-upload" style="font-size:16px"></el-icon></uploader-btn>
      </uploader-drop>
      <uploader-list class="file-list"></uploader-list>
    </uploader>
  </div>
</template>
<script>
  import {ACCEPT_CONFIG} from '@/config'
  import SparkMD5 from 'spark-md5'
  import axios from 'axios'
  import qs from 'qs'

  // const uploaderApi = "http://10.8.18.9:8080/rest/plm/v2/foundation/fileUpload/uploadChunkFile";
  // const mergeApi = "http://10.8.18.9:8080/rest/plm/v2/foundation/fileUpload/mergeFile";


  const uploaderApi = "/api/uploader";
  const mergeApi = "/api/merge";
  const chunkSize = 10 * 1024 * 1000

  export default {
    data() {
      return {
        dialogImageUrl: '',
        dialogVisible: false,
        statusText: {
          success: '上传成功',
          error: '出错了',
          uploading: '上传中',
          paused: '校验中...',
          waiting: '等待中'
        },
        options: {
          target: uploaderApi,
          testChunks: true,
          chunkSize: chunkSize,
          //服务器分片校验函数，秒传及断点续传基础
          checkChunkUploadedByResponse: function (chunk, message) {
            // console.log(chunk, 'checkChunkUploadedByResponse  step-2 &&');
            // console.log(message);
            // let objMessage = JSON.parse(message);
            // if (objMessage.isExist) {
            //   console.log(objMessage, "~~~ObjectMessage: true~~~~")
            //   return true;
            // }
            // console.log(objMessage, "~~~ObjectMessage: false~~~~")
            return false
            // return (objMessage.uploaded || []).indexOf(chunk.offset + 1) >= 0
            // return true;
          },
        },
        attrs: {
          accept: '*'
        }
      };
    },
    computed : {
      uploader() {
        return this.$refs.uploader.uploader;
      }
    },
    methods: {
      handleRemove(file, fileList) {
        console.log(file, fileList);
      },
      handlePictureCardPreview(file) {
        this.dialogImageUrl = file.url;
        this.dialogVisible = true;
      },
      //文件添加到上传框，但是还未上传
      onFileAdded(file){
        console.log('onFileAdded, step1--')
        this.computeMD5(file);

      },
      onFileProgress(rootFile, file, chunk){
        console.log('onFileProgress, step3··');
        console.log(`上传中 ${file.name}，chunk：${chunk.startByte / 1024 / 1024} ~ ${chunk.endByte / 1024 / 1024}`)
      },
      onFileSuccess(rootFile, file, response, chunk){
        console.log('onFileSuccess, step4~~');
      },
      onFileError(){
        console.log('onFileError');
      },
      onFileComplete(){
        console.log('file complete', arguments)

        const file = arguments[0].file;
        axios.post(mergeApi, qs.stringify({
          chunkSize,
          identifier: arguments[0].uniqueIdentifier,
          totalSize: Math.floor(file.size/chunkSize),
        })).then(function (response) {
          console.log(response);
        }).catch(function (error) {
          console.log(error);
        });
      },
      computeMD5(file) {
        let fileReader = new FileReader();
        let time = new Date().getTime();
        let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;
        let currentChunk = 0;
        const chunkSize = 10 * 1024 * 1000;
        let chunks = Math.ceil(file.size / chunkSize);
        let spark = new SparkMD5.ArrayBuffer();

        file.pause();

        loadNext();
        //"be2deac5998f5b9df9fd68f4a811994c"
        fileReader.onload = (e => {
          spark.append(e.target.result);
          if (currentChunk < chunks) {
            currentChunk++;
            //获取当前上传片的md5信息
            let md5 = SparkMD5.ArrayBuffer.hash(e.target.result);
            e.target.result.md5 = md5;
            // 实时展示MD5的计算进度
            console.log('校验MD5 '+ ((currentChunk/chunks)*100).toFixed(0)+'%'+';MD5:' + md5);
            loadNext();
          } else {
            let md5 = spark.end();
            this.computeMD5Success(md5, file);
            console.log(`MD5计算完毕：${file.name} \nMD5：${md5} \n分片：${chunks} 大小:${file.size} 用时：${new Date().getTime() - time} ms`);
          }
        });

        fileReader.onerror = function () {
          this.error(`文件${file.name}读取出错，请检查该文件`)
          file.cancel();
        };

        function loadNext() {
          let start = currentChunk * chunkSize;
          let end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;

          fileReader.readAsArrayBuffer(blobSlice.call(file.file, start, end));
        }
      },
      computeMD5Success(md5, file) {
        // 将自定义参数直接加载uploader实例的opts上
        Object.assign(this.uploader.opts, {
          query: {
            ...this.params,
          }
        })

        file.uniqueIdentifier = md5;
        file.resume();
      },
    }
  }
</script>
<style scoped lang="less">
  .uploader-example {
    width: 600px;
    padding: 15px;
    margin: 40px auto 0;
    font-size: 12px;
    /*box-shadow: 0 0 10px rgba(0, 0, 0, .4);*/
  }
  .uploader-drop-box{
    background-color: #fbfdff !important;
  }
  .uploader-example .uploader-btn {
    margin-right: 4px;
  }
  .uploader-example .uploader-list {
    max-height: 440px;
    overflow: auto;
    overflow-x: hidden;
    overflow-y: auto;
  }
  .file-list {
    /deep/.uploader-file-icon:before{
      content: "\e785" !important;
      font-family: element-icons!important;
      font-size: 21px;
    }
    /deep/.uploader-file-info{
      border-left:1px solid #cdcdcd;
      border-right:1px solid #cdcdcd;
    }
  }
</style>
