<template lang="pug">
.m4sdisk
  el-tabs(v-model="tabsValue")
    el-tab-pane(label='网盘')
      el-row(type="flex")
        el-col(v-bind:span='2', v-if='utils.pathmanager().getPath() != "/"')
          el-button(type='text',@click='backFileList',icon='arrow-left') BACK
        el-col.breadcrumb
          el-breadcrumb(separator=">",v-bind:replace='true')
            el-breadcrumb-item(v-for='p in utils.pathmanager().pathSegmtt()',v-bind:to="{query:{path:p.path}}",key = 'p')
              | {{p.name}}
      el-row(type="flex")
        //- el-upload(ref="upload", v-bind:auto-upload="false", v-bind:on-change='upfilechange', v-bind:action='uploadUrl', v-bind:show-file-list='false', v-bind:before-upload='beforeupload')
        //-   el-button(type="primary") 上传
        el-col
          el-button(@click='createFolder', icon='plus') 新建文件夹
          el-button(@click='pasteHere', icon='el-icon-paste', v-if='clipboard!=null') 粘贴
          el-button-group(v-if='false')
            el-button(icon='delete') 删除
        el-col(v-bind:span='4')
          span Total: {{m4sfilelist.length}}
          el-button(@click='goFileList')
            i(class='fa fa-refresh', aria-hidden='true', v-bind:class='isLoading ? "fa-spin" : "fa"')
      el-row
        el-col(v-loading='isLoading')
          el-table.filelist(v-bind:data='m4sfilelist', empty-text='文件夹是空的哟')
            el-table-column(type='selection')
            el-table-column(label='文件名',show-overflow-tooltip,min-width='200')
              template(scope="scope")
                el-col(v-bind:span='19').file-name
                  img.fileicon(v-bind:src='_fileTypeUri(scope.row)',height=30)
                  span(v-bind:class="scope.row.dir == 1 ? 'open-enable': 'normal'", @click='changefilepath(scope.row)')
                      | {{scope.row.filename}}
            el-table-column(label='-',show-overflow-tooltip,width='100')
              template(scope="scope")
                el-dropdown(trigger="click")
                  el-button(type='text')
                    i(class='el-icon-more')
                  el-dropdown-menu(slot="dropdown")
                    el-dropdown-item(@click.native.prevent='downloadFile(scope.row)') 直链下载
                    el-dropdown-item(@click.native.prevent='downloadFromAria(scope.row)') aria下载
                    el-dropdown-item(divided, @click.native.prevent='copy2clipboard(scope.row.path)') 复制
                    el-dropdown-item(divided, @click.native.prevent='sharethis(scope.row)') 分享
                    el-dropdown-item(@click.native.prevent='add2plaza($squareAPI,scope.row)') 添加到广场
                    el-dropdown-item(divided, @click.native.prevent='deleteFile(scope.row)') 删除
            el-table-column(label='大小',width='120')
              template(scope='scope')
                span {{utils.transeSize(scope.row.size)}}
            el-table-column(label='创建时间',show-overflow-tooltip,width='160')
              template(scope='scope')
                span {{utils.transeTime(scope.row.ctime)}}
    el-tab-pane(label='离线')
      el-row(type="flex")
        el-col
          el-button(@click='createOffline', icon='plus') 新建离线
          span &nbsp;&nbsp;&nbsp;&nbsp;配额: {{offline.quotacount}}， 过期时间: {{offline.vipTime}}
        el-col
          el-button(@click='offlinetasks')
            i(class='fa fa-refresh', aria-hidden='true', v-bind:class='isLoading ? "fa-spin" : "fa"')
      el-row
        el-col
          el-table(v-bind:data='offline.list', empty-text='暂时没有离线任务')
            el-table-column(label='名称',prop='name',show-overflow-tooltip,min-width='200')
            el-table-column(label='链接',prop='link',show-overflow-tooltip,min-width='200')
            el-table-column(label='状态',width='100')
              template(scope="scope")
                el-tag(type='success', v-if='scope.row.state === 0') 已完成
                el-tag(type='warning', v-if='scope.row.state === 1') 正在初始化
                el-tag(type='primary', v-if='scope.row.state === 2') 下载中
            el-table-column(label='进度',width='100')
              template(scope="scope")
                span {{scope.row.downloadedCount}}/{{scope.row.totalCount}}
            el-table-column(label='速度',width='140')
              template(scope="scope")
                span {{utils.transeSpeed(scope.row.speed)}}
    el-tab-pane(label='分享')
      span 敬请期待
  down-dialog(v-model='dialogDL', v-bind:downlinks='downlinks')
</template>

<script>
import {loginmixin} from '@/components/mixins/loginmixin'
import {diskmixin} from '@/components/mixins/diskmixin'
export default {
  name: 'webdisk',
  mixins: [diskmixin, loginmixin],
  data () {
    return {
      clipboard: null,
      uploadUrl: '',
      tabsValue: '0',
      offline: {
        list: [],
        quotacount: 0,
        vipTime: '1970/1/1'
      }
    }
  },
  methods: {
    goFileList: function () {
      const path = this.utils.pathmanager().getPath()
      this.token = this.getToken()
      this.$store.commit('viewloading', true)
      this.$m4sAPI.filelist(this.token,path)
        .then(list=>{
          this.$store.commit('viewloading', false)
          this.$store.dispatch('filelist',{list: list,ism4s: true})
        })
        .catch((e)=>{
          this.$store.commit('viewloading', false)
          this.$message.error(e.message)
        })
    },
    downloadFile: function (file) {
      this.$store.commit('viewloading', true)
      this.$m4sAPI.downfiles(this.token, file.path)
        .then(links=>{
          this.$store.commit('viewloading', false)
          // this.dialogDL = true
          // this.downlinks = [{
          //   name: file.filename,
          //   urls: links
          // }]
          this.utils.downloadFromFrame(links[0])
        })
        .catch((e)=>{
          this.$store.commit('viewloading', false)
          this.$message.error(e.message)
        })
    },
    downloadFromAria: function (file) {
      this.$store.commit('viewloading', true)
      this.$m4sAPI.downfiles(this.token, file.path)
        .then(links=>{
          this.$store.commit('viewloading', false)
          this.$Aria2API.addUri(links[0], file.filename)
            .then(result=>this.$message.success('下载成功!'))
            .catch(e=>this.$message.error('下载失败!请参照关于中的说明确认aria2配置是否正确。'))
        })
        .catch((e)=>{
          this.$store.commit('viewloading', false)
          this.$message.error(e.message)
        })
    },
    createfolderapi: function (value) {
      const path = `${this.utils.pathmanager().getPath()}/${value}`
      this.$m4sAPI.createFolder(this.token, path)
        .then(data=>{
          this.$message.success(data.errno === 0 ? '创建成功!' : data.message)
          this.goFileList()
        })
    },
    copy2clipboard: function (filepath) {
      this.clipboard = filepath
    },
    pasteHere: function () {
      let topath = this.utils.pathmanager().getPath()
      topath += '/' + this.clipboard.fileName
      this.$m4sAPI.copyFile(this.token,this.clipboard,topath)
        .then(a=>{ this.$message.success('粘贴成功!'); this.clipboard = null; this.goFileList() })
        .catch(e=>this.$message.error(e.message))
    },
    deleteFileapi: function (filepath) {
      this.$m4sAPI.deletefile(this.token, filepath)
        .then((data)=>{
          this.$message.success('删除成功!')
          this.goFileList()
        })
        .catch(e=>{ this.$message.error('删除失败。') })
    },
    sharethis: function (file) {
      this.$prompt('请输入分享密码,留空为公开分享：', '创建分享', {
        inputValue: 'pass',
        confirmButtonText: '确定',
        cancelButtonText: '取消'
      }).then(({ value }) => {
        this.$m4sAPI.share(this.token, [file.path], value)
          .then(data=>{
            this.$message.success(`分享成功! Id: ${data.shareid}, 请牢记。`)
          })
          .catch(e=>{ this.$message.error('分享失败。') })
      }).catch(e => {})
    },
    createOffline: function () {
      this.$prompt('请输入离线磁力链接：', '创建离线任务', {
        inputValue: 'magnet:?xt=urn:',
        confirmButtonText: '确定',
        cancelButtonText: '取消'
      }).then(({ value }) => {
        this.$m4sAPI.createoffline(this.token, value)
          .then(data=>{
            if (data.errno === 0) {
              this.$message.success(`离线成功!`)
              this.offlinetasks()
            } else {
              this.$message.error(data.message)
            }
          })
      }).catch(e => {})
    },
    offlinetasks: function () {
      this.tabsValue = this.utils.pathmanager().getTabsValue()
      this.$m4sAPI.offlinelist(this.token)
        .then((data)=>{
          this.offline.list = data.list
          this.offline.quotacount = data.quotaOfflineCount
          this.offline.vipTime = data.vipTime
        })
    }
  },
  watch: {
    tabsValue () {
      this.$router.push({query: {tab: this.tabsValue}})
    },
    '$route': 'goFileList'
  },
  mounted () {
    this.goFileList()
    this.offlinetasks()
  }
}
</script>

<style scoped lang="scss">
.breadcrumb{
  margin: 10px
}
.file-name{ 
  line-height: 40px;
  .fileicon{
    vertical-align:middle;
    margin-right: 5px;
  }
  .open-enable{
    cursor: pointer;
  }
  .normal{
    cursor: default;
  }
}
</style>