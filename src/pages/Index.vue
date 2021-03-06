<template>
  <q-page>
    <div class="row q-px-sm bg-blue-grey-5">
      <!-- ヘッダー (フォルダパス) -->
      <div class="col-xs-12">
        <q-field dense dark borderless>
          <template v-slot:before>
            <q-icon name="save" class="cursor-pointer"
              @click="saveFile()"
            />
          </template>
          <template v-slot:prepend>
            <q-icon name="folder" />
          </template>
          <template v-slot:control class="q-pa-md q-gutter-sm">
            <q-breadcrumbs class="text-no-wrap"
              gutter="xs"
              active-color="white"
              separator-color="black"
              :separator="pathSeparator"
              >
              <q-breadcrumbs-el
                v-for="breadcrumb in breadcrumbs" :key="`breadcrumb_${breadcrumb.label}`"
                :label="breadcrumb.label" class="cursor-pointer"
                @click="moveCurrentDir(breadcrumb.absolutePath)"
                />
            </q-breadcrumbs>
          </template>
          <template v-slot:append>
            <q-icon name="more_horiz" class="cursor-pointer"
              @click="showOpenDialog()"
              />
          </template>
        </q-field>
      </div>
    </div>
    <div class="row">
      <div class="col col-grow list-col">
        <!-- フィルターテキストボックス -->
        <div class="bg-blue-grey-6">
          <q-input dense dark borderless class="q-mx-sm"
            v-model="filterPattern"
            >
          </q-input>
        </div>
        <!-- ファイル一覧 -->
        <q-list dense>
          <q-item clickable
            v-for="item in displayItems" :key="`item_${item.name}`"
            @click="setCurrentItem(item)"
            >
            <q-item-section avatar>
              <q-icon
                :color="item.isFile ? 'blue-grey-3' : 'yellow-14'"
                :name="item.isFile ? 'insert_drive_file' : 'folder'"
                />
            </q-item-section>
            <q-item-section
              :class="item === currentFile ? 'text-bold' : ''"
              >
              {{ item.name }}
            </q-item-section>
          </q-item>
        </q-list>
      </div>
      <div class="col editor-col">
        <!-- ファイルパス -->
        <div class="bg-blue-grey-6">
          <q-input dense dark borderless class="q-mx-sm" readonly
            :value="currentFile && currentFile.name"
            />
        </div>
        <!-- エディター -->
        <div id="editor"></div>
      </div>
    </div>
  </q-page>
</template>

<style lang="stylus" scoped>
.q-page
  height 100%
  min-height 100%
  display flex
  flex-direction column
  > :nth-child(1)
    flex 0 0 auto
  > :last-child
    flex 1 0 0
.editor-col, .list-col
  display flex
  flex-direction column
  > :last-child
    flex 1 0 0
    overflow auto
.list-col
  width 30%
  max-width 30%
  border-right solid 1px $blue-grey-3
</style>

<script>
import fs from 'fs';
import 'ace-min-noconflict';
import path from 'path';
import { remote } from 'electron';

export default {
  name: 'Index',
  props: {
    // 親から渡すプロパティ (書き換え不可)
  },
  data() {
    // このコンポーネント内で使うデータ変数 (書き換え可)
    return {
      currentDir: process.cwd(), // 選択中のフォルダパス
      currentFile: null, // 選択中のファイル
      filterPattern: '', // 絞り込み用フィルター文字列
      list: [], // フォルダのアイテムの配列
      editor: null, // Ace エディターのオブジェクト
    };
  },
  computed: {
    // 自動算出プロパティ (読み取り専用)
    displayItems() {
      if (!this.filterPattern) return this.list;
      const regex = new RegExp(this.filterPattern);
      return this.list.filter(x => regex.test(x.name));
    },
    breadcrumbs() {
      const breadcrumbs = this.currentDir.split(path.sep);
      let absolutePath = '';
      return breadcrumbs.map((label) => {
        absolutePath = absolutePath ? path.join(absolutePath, label) : label;
        return {
          label,
          absolutePath,
        };
      });
    },
    pathSeparator() {
      return path.sep;
    },
  },
  methods: {
    // このコンポーネント内で使うメソッド群
    loadItems() {
      fs.readdir(this.currentDir, (err, files) => {
        if (err) throw err;
        this.list = files.map((file) => {
          const absolutePath = path.resolve(this.currentDir, file);
          const isFile = fs.statSync(absolutePath).isFile();
          return {
            name: file,
            absolutePath,
            isFile,
          };
        }).sort((a, b) => {
          if (a.isFile && !b.isFile) return 1;
          if (!a.isFile && b.isFile) return -1;
          return 0;
        });
      });
    },
    setCurrentItem(item) {
      if (item.isFile) {
        this.currentFile = item;
        fs.readFile(item.absolutePath, 'utf8', (err, data) => {
          this.editor.setValue(data, -1);
        });
      } else {
        this.moveCurrentDir(item.absolutePath);
      }
    },
    moveCurrentDir(dirPath) {
      this.currentDir = dirPath;
      this.loadItems();
    },
    showOpenDialog() {
      const dirs = remote.dialog.showOpenDialog(
        remote.BrowserWindow.getFocusedWindow(), // 現在のウィンドウを親にする
        {
          properties: ['openDirectory'], // ディレクトリを開くモード
          defaultPath: this.currentDir,
        },
      );
      if (dirs) {
        this.moveCurrentDir(dirs.shift());
      }
    },
    saveFile() {
      if (!this.currentFile) return;
      fs.writeFile(this.currentFile.absolutePath, this.editor.getValue(), (error) => {
        if (error != null) console.log(error);
      });
    },
  },
  mounted() {
    // このコンポーネントが最初に初期化されたときに処理する内容
    this.editor = window.ace.edit('editor');
    this.loadItems();
  },
};
</script>
