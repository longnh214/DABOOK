<template>
  <div>
    <v-treeview
      v-model="tree"
      :open="initiallyOpen"
      :items="items"
      activatable
      item-key="name"
      open-on-click
      @input="openFile"
    >
      <template slot="label" slot-scope="{ item }" >
        <div  @mouseover="mouseAction(item,true);" @mouseleave="mouseAction(item,false);">
            <v-icon small style="padding: 0 5px;" v-if="!item.file">
              {{ 'mdi-folder' }}
            </v-icon>
            <v-icon v-else small style="padding: 0 5px;" >
              {{ files[item.file] }}
            </v-icon>
            <span @click="openFile(item);">{{ item.name }}</span>
          <span id="toc" v-if="deleteBtn[item.name]">
            <v-btn
            class="align-self-center rounded-sm"
            icon
            x-small
            @click="deleteDialog=true; deleteitem=item">
              <v-icon>mdi-trash-can-outline</v-icon>
            </v-btn>
          </span>
        </div>
      </template>
    </v-treeview>

    <Confirm 
      :dialog="saveDialog"
      title="confirm.save-title"
      content1="confirm.save-content"
      content2=""
      confirm="confirm.save-confirm"
      cancel="confirm.save-cancel"
      @cancel="saveFile(false)"
      @confirm="saveFile(true)"
    />
    
    <Confirm 
      :dialog="deleteDialog"
      title="deleteChapter.title"
      content1="deleteChapter.content-1"
      content2="deleteChapter.content-2"
      confirm="deleteChapter.delete-btn"
      cancel="deleteChapter.cancel"
      @cancel="deleteChapter(false)"
      @confirm="deleteChapter(true)"
    />
  </div>
</template>

<script>
import eventBus from '@/eventBus.js';
import * as edit from '@/functions/edit.js';
import Vue from 'vue'
import Confirm from '@/components/mainpage/Confirm'
import * as file from '@/functions/file.js';

const fs = require('fs');

export default {
  name: 'Directory',
  components: {
    Confirm
  },
  watch: {
    items: function () {
      this.initiallyOpen = ['EPUB', 'text']
      for (let j = 0; j < this.items[0].children[4].children.length; j++) {
        if (this.items[0].children[4].children[j].file === "xhtml") {
          Vue.set(this.deleteBtn, this.items[0].children[4].children[j].name, false);
        }
      }
    },
  },
  computed: {
    items: function () {
      return this.$store.state.ebookDirectoryTree;
    },

    // 확인 버튼 한/영 변환
    GET_CONFIRM_BUTTON() {
      return this.$t('confirm.save-confirm');
    },

    // 취소 버튼 한/영 변환
    GET_CANCEL_BUTTON() {
      return this.$t('confirm.save-cancel');
    },
  }, 
  data: function () {
    return {
      initiallyOpen: ['EPUB', 'text'],
      files: {
        xhtml: 'mdi-book-open-page-variant',
        html: 'mdi-book-open-page-variant',
        ncx: 'mdi-table-of-contents',
        opf: 'mdi-cogs',
        ttf: 'mdi-format-font',
        png: 'mdi-file-image',
        jpg: 'mdi-file-image',
        xml: 'mdi-xml',
        css: 'mdi-language-css3',
      },
      tree: [],
      deleteBtn: new Object(),
      saveDialog: false,
      deleteDialog:false,
      deleteitem:{},
      value : ''
    };
  },
  methods: {
    // 목차 삭제
    deleteChapter: function (res) {
      this.deleteDialog=false;
      if (!res)
        return;
      let item = this.deleteitem;
      fs.unlinkSync(item.dirPath, (err) => {
        console.log(err);
      });
      Vue.nextTick();

      //content 변경
      let content = fs.readFileSync(this.$store.state.ebookDirectory + '/EPUB/content.opf').toString();
      let startText = '<item id="'+item.name;
      let endText = "/>"
      let start = content.indexOf(startText);
      let end = content.indexOf(endText,start);
      content = content.slice(0,start) + content.slice(end+3);

      start = content.indexOf('<itemref idref="' + item.name);
      end = content.indexOf("/>", start);
      content = content.slice(0, start) + content.slice(end + 3);

      fs.writeFileSync(this.$store.state.ebookDirectory + '/EPUB/content.opf', content, (err) => {
        if (err) {
          console.log(err);
        }
      });

      // toc 변경
      let temp = fs.readFileSync(this.$store.state.ebookDirectory + '/EPUB/toc.ncx').toString();

      startText = 'chapter';
      endText = '.xhtml';
      start = item.name.indexOf(startText);
      end = item.name.indexOf(endText, start);
      let searchNumber = item.name.slice(start + startText.length, end);

      let point = 0;
      let searchDirTextStart = '<navPoint id="navPoint-';
      let searchDirTextEnd = '">';
      for (;;) {
        if (temp.length <= point)
          break;
        let dirStart = temp.indexOf(searchDirTextStart, point);
        let dirEnd = temp.indexOf(searchDirTextEnd, dirStart);
        let number = temp.slice(dirStart + searchDirTextStart.length, dirEnd);

        if (searchNumber == number) {
          let t = '</navPoint>';
          let last = temp.indexOf(t, dirEnd);
          temp = temp.slice(0, dirStart) + temp.slice(last + t.length);
          fs.writeFileSync(this.$store.state.ebookDirectory + '/EPUB/toc.ncx', temp, (err) => {
            if (err) {
              console.log(err);
            }
          });
          break;
        }
        point = dirEnd;
      }
      this.$store.state.selectedFileDirectory = '';
      eventBus.$emit('toc');
    },

    // mouse over -> 삭제버튼 활성화
    mouseAction: function (item, set) {
      if (item.file == "xhtml") {
        this.deleteBtn[item.name] = set;
        this.$nextTick();
      }
    },

    // 디렉토리에서 선택한 파일을 텍스트로 읽는 함수
    openFile: function (val) {
      this.value = val;
      if (val.children) {
        return  // 폴더면 그냥 return
      } else {  // 파일을 클릭한 것이면
        if (this.$store.state.selectedFileDirectory !== '' && this.$store.state.selectedFileDirectory !== val.dirPath) {  
          // 처음 여는게 아니거나 다른 파일을 열려고 하는 것이면, 변경여부 확인하고 저장여부를 물어본다.
          let original = fs.readFileSync(this.$store.state.selectedFileDirectory).toString();  // original: 원래 작성중이던 파일의 원본
          original = original.slice(original.indexOf("<body"), original.indexOf("</body>") + 7);
          if (!this.compareFile(original, this.$store.state.editingText)) {
            this.saveDialog = true;
          } else {
            this.moveFile(val);
          }
        }
        else {
          this.moveFile(val);
          }

        edit.reset();
      }
    },

    // 저장
    saveFile: function (flag) { 
      if (flag) {
        const updatedText = this.$store.state.editingHTMLText + this.$store.state.editingText + '</html>';
        fs.writeFileSync(this.$store.state.selectedFileDirectory, updatedText);
        this.$store.dispatch('setAlertMessage', 'success.save-ebook');
        // 서버 업로드
        let email = localStorage.getItem('email');
        let filePath = this.$store.state.selectedFileDirectory
        let bookName = this.$store.state.ebookTitle
        let serverPath = "/EPUB/text"
        file.uploadFile(filePath, serverPath, bookName, email);
        console.log('서버에 저장!')
      } 
      this.saveDialog = false;
      this.moveFile(this.value);
    },

    // 파일 이동
    moveFile: function (val) {
      if (this.$store.state.selectedFileDirectory !== val.dirPath) {  // 자기 자신을 다시 클릭한 것이 아니라면 새 파일을 가져온다.
        if (val.dirPath.slice(-5) === 'xhtml') {    // xhtml 파일이라면 불러온다.
          const temp = fs.readFileSync(val.dirPath).toString();
          this.$store.dispatch('setSelectedFileDirectory', val.dirPath);
          eventBus.$emit('loadData', temp);
        } else {    // xhtml 파일이 아니라면 alert 를 띄운다. 
          this.$store.dispatch('setAlertMessage', 'error.select-text');
        }
      }
    },

    // 파일 비교
    compareFile: function (pre, post) {
      if (pre === post) return true;
      return false;
    }
  },
}
</script>
