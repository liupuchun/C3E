<template>
  <el-container id="wrapper">
    <el-header height="40px">
      <MenuBar></MenuBar>
    </el-header>
    <el-container>
      <el-aside width="200px">
        <FileMenuBar></FileMenuBar>
      </el-aside>
      <hr />
      <el-container>
        <el-main>
          <EditorsTab height="80%"></EditorsTab>
        </el-main>
        <el-footer height="150px"><WebTerminal></WebTerminal></el-footer>
      </el-container>
    </el-container>
  </el-container>
</template>

<script>
import MenuBar from './menubar/MenuBar'
import EditorsTab from './editorcore/EditorsTab'
import FileMenuBar from './filemenubar/FileMenuBar'
import WebTerminal from './terminal/WebTerminal'
export default {
  name: 'index-page',
  components: { MenuBar, EditorsTab, FileMenuBar, WebTerminal },
  methods: {
    open(link) {
      this.$electron.shell.openExternal(link)
    },
    getFilePath(data) {
      console.log(data)
    }
  }
}
</script>

<style>
html,
body{
  margin: 0;
  padding: 0;
}

#wrapper {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.el-header{
  background-color: #b3c0d1;
  color: #333;
  text-align: center;
  line-height: 40px;
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB',
    'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
}

.el-footer{
  background-color: #000;
}

.el-aside {
  text-align: center;
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB',
    'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
}
.el-main {
  margin: 0%;
  padding: 0%;
}
</style>
