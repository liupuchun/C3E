<template>
<div style="padding: 0;height: 100%;user-select: none;">
  <el-scrollbar style="height:100%">
    <el-tree
      :data="data"
      :default-expanded-keys="defaultExp"
      :props="defaultProps"
      @node-click="handleNodeClick"
      @node-contextmenu="handleNodeRightClick"
      highlight-current
      node-key="id"
      ref="filemenubar"
    ></el-tree>
    <div v-show="dirMenuVisible">
      <ul class="rightmenu" id="dirrightmenu">
        <li @click="RmDir" class="menu_item">删除文件夹(施工中)</li>
      </ul>
    </div>
    <div v-show="editZoneMenuVisible">
      <ul class="rightmenu" id="editrightmenu">
        <li @click="CloseFile" class="menu_item">关闭文件(施工中)</li>
      </ul>
    </div>
    <div v-show="fileMenuVisible">
      <ul class="rightmenu" id="filerightmenu">
        <li @click="RmFile" class="menu_item">删除文件(施工中)</li>
      </ul>
    </div>
  </el-scrollbar>
</div>
</template>

<script>
// 节点指的是菜单栏中的每一个叶子菜单
// 对象格式：{id: 节点编号, label: 显示的文件名, file: 文件名, path: 文件路径（unname文件默认为null）}
import { addNode, getFileNameFromPath } from '../../common/tools.js'
let ipcRenderer = require('electron').ipcRenderer
export default {
  data() {
    return {
      fileMenuVisible: false, // 对文件右键菜单可见性
      dirMenuVisible: false, // 对文件夹右键菜单可见性
      editZoneMenuVisible: false, // 对编辑区文件右键菜单可见性
      dataWhenRightClick: null, // 右键点击时放入的数据
      defaultExp: [], // 默认展开的节点们
      currentId: 0, // 当前高亮的节点
      unnameMount: 0, // unname文件总数（包含已删除的）
      fileMount: 0, // 当前文件总数
      data: [
        {
          label: '编辑区文件',
          children: []
        }
      ],
      defaultProps: {
        children: 'children',
        label: 'label'
      }
    }
  },
  created() {
    let _this = this
    // 信号1，来自MenuBar.vue的打开文件信号，信号名：selectedFile，接收参数：事件、文件路径
    ipcRenderer.on('selectedFile', function(event, path) {
      // console.log(path)
      let strpath = String(path)
      // 判断文件是否已经在文件菜单内
      let filemenu = _this.data[0]['children']
      let ifExist = false
      let existId = null
      for (let index = 0; index < filemenu.length; index++) {
        if (filemenu[index]['path'] === strpath) {
          ifExist = true
          existId = filemenu[index]['id']
          break
        }
      }
      if (ifExist === false) {
        // 获取到文件路径和文件名加入菜单
        let pos = strpath.lastIndexOf('/')
        if (pos === -1) {
          pos = strpath.lastIndexOf('\\')
        }
        let filename = strpath.substring(pos + 1)
        let filelabel = filename + `(${strpath})`
        _this.data[0]['children'].push({
          id: _this.fileMount,
          label: filelabel,
          file: filename,
          path: strpath
        })
        _this.currentId = _this.fileMount
        _this.fileMount++
        let __this = _this
        _this.$nextTick(function() {
          __this.refRecursion(__this.currentId, 1)
        })
      } else {
        // 说明已经存在该文件，高亮处理
        _this.currentId = existId
        _this.$refs['filemenubar'].setCurrentKey(existId)
      }
    })
    // 信号2，来自EditorsTab.vue的新建文件信号，信号名：fileMenuNewFile，接收参数：事件、数据
    ipcRenderer.on('fileMenuNewFile', function(event, data) {
      _this.currentId = _this.fileMount
      _this.data[0]['children'].push({
        id: _this.fileMount,
        label: 'unname' + _this.unnameMount,
        file: 'unname',
        path: null
      })
      _this.unnameMount++
      _this.fileMount++
      let __this = _this
      _this.$nextTick(function() {
        __this.refRecursion(__this.currentId, 1)
      })
    })
    // 信号3，来自EditorsTab.vue的关闭文件信号，信号名：fileMenuRemoveFile，接收参数：事件、数据
    ipcRenderer.on('fileMenuRemoveFile', function(event, data) {
      if (data['path'] === null) {
        // 此时为unname文件
        for (let index = 0; index < _this.data[0]['children'].length; index++) {
          if (_this.data[0]['children'][index]['label'] === data['name']) {
            _this.data[0]['children'].splice(index, 1)
            break
          }
        }
      } else {
        // 此时为非unname文件
        for (let index = 0; index < _this.data[0]['children'].length; index++) {
          if (_this.data[0]['children'][index]['path'] === data['path']) {
            _this.data[0]['children'].splice(index, 1)
            break
          }
        }
      }
      if (data['activetab'] !== null) {
        // 此时EditorsTab关闭的是当前标签页
        if (data['activetab']['path'] === null) {
          // 此时关闭的是unname
          for (
            let index = 0;
            index < _this.data[0]['children'].length;
            index++
          ) {
            if (
              _this.data[0]['children'][index]['label'] ===
              data['activetab']['title']
            ) {
              _this.currentId = _this.data[0]['children'][index]['id']
              let __this = _this
              _this.$nextTick(function() {
                __this.refRecursion(__this.currentId, 1)
              })
              break
            }
          }
        } else {
          for (
            let index = 0;
            index < _this.data[0]['children'].length;
            index++
          ) {
            if (
              _this.data[0]['children'][index]['path'] ===
              data['activetab']['path']
            ) {
              _this.currentId = _this.data[0]['children'][index]['id']
              let __this = _this
              _this.$nextTick(function() {
                __this.refRecursion(__this.currentId, 1)
              })
              break
            }
          }
        }
      }
    })
    // 信号4，来自EditorsTab.vue的信号，目的是通过标签页切换文件菜单状态，信号名：selectedFile，接收参数：事件、{文件名，文件路径}
    ipcRenderer.on('changeFileMenuFromTab', function(event, thedata) {
      if (thedata['path'] == null) {
        // 切换的是unname标签页
        for (let index = 0; index < _this.data[0]['children'].length; index++) {
          if (_this.data[0]['children'][index]['label'] === thedata['title']) {
            _this.currentId = _this.data[0]['children'][index]['id']
            let __this = _this
            _this.$nextTick(function() {
              __this.refRecursion(__this.currentId, 1)
            })
            break
          }
        }
      } else {
        for (let index = 0; index < _this.data[0]['children'].length; index++) {
          if (_this.data[0]['children'][index]['path'] === thedata['path']) {
            _this.currentId = _this.data[0]['children'][index]['id']
            let __this = _this
            _this.$nextTick(function() {
              __this.refRecursion(__this.currentId, 1)
            })
            break
          }
        }
      }
    })
    // 信号5，来自MonacoEditor.vue的“unname文件被第一次保存”事件，信号名：unnamedFileHasSaved，接收参数：事件、{文件名，文件路径}
    ipcRenderer.on('unnamedFileHasSaved', function(event, thedata) {
      for (let index = 0; index < _this.data[0]['children'].length; index++) {
        if (_this.data[0]['children'][index]['label'] === thedata.title) {
          _this.data[0]['children'][index]['path'] = thedata.path
          let filename = getFileNameFromPath(thedata.path)
          _this.data[0]['children'][index]['file'] = filename
          _this.data[0]['children'][index]['label'] =
            filename + `(${thedata.path})`
          break
        }
      }
    })
    // 信号6，来自MenuBar.vue的打开文件夹事件，信号名：selectedDir，接收参数：事件、目录路径
    ipcRenderer.on('selectedDir', function(event, dirpath) {
      let strpath = String(dirpath)
      let slash = '/'
      let pos = strpath.lastIndexOf('/')
      if (pos === -1) {
        pos = strpath.lastIndexOf('\\')
        slash = `\\`
      }
      let dirname = strpath.substring(pos + 1)
      let dirtree = {
        ifdir: true,
        label: dirname,
        path: strpath,
        children: []
      }
      let walk = require('walk')
      let walker = walk.walk(strpath, { followLinks: false })
      walker.on('file', function(root, stat, next) {
        dirtree = addNode(dirtree, root, stat.name, false, slash)
        next()
      })
      walker.on('directory', function(root, stat, next) {
        dirtree = addNode(dirtree, root, stat.name, true, slash)
        next()
      })
      walker.on('errors', function(root, stat, next) {
        next()
      })
      walker.on('end', function() {
        _this.data.push(dirtree)
      })
    })
  },
  mounted() {},
  methods: {
    refRecursion(key, time) {
      if (time > 10) throw new Error('没有找到 vueTree')
      let tree = this.$refs['filemenubar']
      if (tree) {
        this.defaultExp = [key]
        tree.setCurrentKey(key)
      } else {
        this.refRecursion(key, time + 1)
      }
    },
    handleNodeClick(data) {
      if (data.ifdir === undefined) {
        ipcRenderer.send('change-tab-from-file-menu', data)
      } else if (data.ifdir === false) {
        ipcRenderer.send('open-file-by-dir-tree', data.path)
      }
    },
    handleNodeRightClick(event, data) {
      this.dataWhenRightClick = data
      this.editZoneMenuVisible = false
      this.fileMenuVisible = false
      this.dirMenuVisible = false
      let rightMenu = null
      if (data.ifdir === undefined && data.file !== undefined) {
        rightMenu = document.querySelector('#editrightmenu')
        this.editZoneMenuVisible = true
      } else if (data.ifdir === true) {
        this.dirMenuVisible = true
        rightMenu = document.querySelector('#dirrightmenu')
      } else if (data.ifdir === false) {
        this.fileMenuVisible = true
        rightMenu = document.querySelector('#filerightmenu')
      }
      if (rightMenu !== null) {
        rightMenu.style.left = event.clientX + 'px'
        rightMenu.style.top = event.clientY + 'px'
        document.addEventListener('click', this.foo)
      }
    },
    foo() {
      this.editZoneMenuVisible = false
      this.fileMenuVisible = false
      this.dirMenuVisible = false
      document.removeEventListener('click', this.foo)
    },
    RmDir() {
      // TODO
    },
    RmFile() {
      // TODO
    },
    CloseFile() {
      // TODO
    }
  }
}
</script >

<style lang="less">
.el-scrollbar .el-scrollbar__wrap {
  overflow-x: hidden;
}
.el-tree > .el-tree-node {
  min-width: 100%;
  display: inline-block;
}
.menu_item {
  display: block;
  line-height: 20px;
  text-align: left;
  margin-top: 10px;
  margin-bottom: 10px;
  padding-left: 20px;
}
.rightmenu {
  width: 200px;
  z-index: 99999;
  position: fixed;
  border: 1px solid #999999;
  background-color: #f4f4f4;
}
li:hover {
  cursor: default;
  background-color: #1790ff;
  color: white;
}
</style>
