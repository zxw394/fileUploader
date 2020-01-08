<template>
  <div class="about">
    <uploader :options="options" class="uploader-example"
              ref="uploader"
              :autoStart="false"
              :file-status-text="statusText"
              @file-added="onFileAdded"
              @file-success="onFileSuccess"
              @file-progress="onFileProgress"
              @file-error="onFileError">
      <uploader-unsupport></uploader-unsupport>
      <uploader-drop class="uploader-drop-box">
        <p>将文件拖到此处，或点击上传</p>
        <uploader-btn :attrs="attrs" ref="uploaderBtn"><el-icon class="el-icon-upload" style="font-size:16px"></el-icon></uploader-btn>
      </uploader-drop>
      <uploader-list class="file-list">
          <ul slot-scope="props">
              <li v-for="file in props.fileList" :key="file.id">
                  <uploader-file :file="file" :class="'file_'+file.id" :id="'file_'+file.id"></uploader-file>
              </li>
          </ul>
      </uploader-list>
    </uploader>
  </div>
</template>
<script>
import SparkMD5 from 'spark-md5'
import axios from 'axios'
import $ from 'jquery'

// const uploaderApi = "http://10.8.18.9:8080/rest/plm/v2/foundation/fileUpload/uploadChunkFile";
// const mergeApi = "http://10.8.18.9:8080/rest/plm/v2/foundation/fileUpload/mergeFile";

const uploaderApi = "/api/uploader";
const mergeApi = "/api/merge";
const chunkSize = 100 * 1024 * 1000

export default {
    data() {
      return {
        statusText: {
          success: '上传成功',
          error: '出错了',
          uploading: '上传中',
          paused: '暂停',
          waiting: '等待中'
        },
        options: {
            target: uploaderApi,
            testChunks: true,
            chunkSize,
            // checkChunkUploadedByResponse: function (chunk, message) {//服务器分片校验函数，秒传及断点续传基础
            //     let objMessage = JSON.parse(message);
            //     if (objMessage.isExist) {
            //         return true;
            //     }
            //     return (objMessage.uploaded || []).indexOf(+chunk.offset + 1 + '') >= 0
            // },
        },
        attrs: {
          accept: '*'
        }
      };
    },
    methods: {
        //文件添加到上传框，但是还未上传
        onFileAdded (file) {
            console.log('file added')
            this.computeMD5(file);
        },
        onFileProgress (rootFile, file, chunk) {
            // console.log(`上传中 ${file.name}，chunk：${chunk.startByte / 1024 / 1024} ~ ${chunk.endByte / 1024 / 1024}`)
        },
        onFileSuccess(rootFile, file, response, chunk){
            console.log('upload success');
            let res = {};
            try{
                res = JSON.parse(response);
            }catch(error){
                res.needMerge = false;
                console.error("onFileSuccess func response format error")
            }
            // 如果服务端返回需要合并
            if (res.needMerge) {
              axios({
                  method: "post",
                  url: mergeApi,
                  data : {
                      chunkSize,
                      identifier: file.uniqueIdentifier,
                      totalSize:Math.floor(file.size/chunkSize) === 0 ? 1 : Math.floor(file.size/chunkSize),
                      type: file.type,
                      filename: file.name,
                  },
                  headers: {
                      'Content-Type': 'application/json',
                  },
              }).then(function (response) {
              }).catch(function (error) {
              });
            }
        },
        onFileError(){
            console.log("upload error");
        },
        computeMD5(file) {
            let fileReader = new FileReader();
            let time = new Date().getTime();
            let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;
            let currentChunk = 0;
            let chunks = Math.ceil(file.size / chunkSize);
            let spark = new SparkMD5.ArrayBuffer();
            file.pause();
            //正在md5校验中
            this.statusSet(file.id, 'md5');
            hideResumeBtn(file.id);
            loadNext();
            fileReader.onload = (e => {
                spark.append(e.target.result);
                if (currentChunk < chunks) {
                    currentChunk++;
                    //获取当前上传片的md5信息
                    // let md5 = SparkMD5.ArrayBuffer.hash(e.target.result);
                    loadNext();
                } else {
                    let md5 = spark.end();
                    this.computeMD5Success(md5, file);
                }
            });
            fileReader.onerror = function () {
                console.error(`文件${file.name}读取出错，请检查该文件`)
                file.cancel();
            };
            function loadNext() {
                let start = currentChunk * chunkSize;
                let end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;
                fileReader.readAsArrayBuffer(blobSlice.call(file.file, start, end));
            }
        },
        computeMD5Success(md5, file) {
            //校验完毕，删除提示'校验中'
            this.statusRemove(file.id);
            displayResumeBtn(file.id);
            
            file.uniqueIdentifier = md5;
            file.resume();
        },
        statusSet(id, status) {
            let statusMap = {
                md5: {
                    text: '校验MD5中',
                    bgc: '#fff'
                },
            }
            setTimeout(() => {
                $(`<p class="myStatus_${id}"></p>`).appendTo(`.file_${id} .uploader-file-status`).css({
                    'position': 'absolute',
                    'top': '0',
                    'left': '0',
                    'right': '0',
                    'bottom': '0',
                    'zIndex': '1',
                    'lineHeight': '29px',
                    'backgroundColor': statusMap[status].bgc
                }).text(statusMap[status].text);
            })
        },
        statusRemove(id) {
            setTimeout(() => {
                $(`.myStatus_${id}`).remove();
            })
        },
    }
}
function displayResumeBtn (fileId) {
  setTimeout(() => {
      document.querySelector(`.file_${fileId} .uploader-file-info .uploader-file-actions .uploader-file-resume`)
          .classList.remove("display-none");
  })
}
function hideResumeBtn (fileId) {
  setTimeout(() => {
      document.querySelector(`.file_${fileId} .uploader-file-info .uploader-file-actions .uploader-file-resume`)
          .classList.add("display-none");
  })
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
    /deep/.uploader-file[status="waiting"] .uploader-file-resume{
        display: none !important;
    }
  }
    /deep/.display-none{
        display: none !important;
    }
</style>
