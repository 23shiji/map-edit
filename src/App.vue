<template lang="jade">
#app
  h1.center 驿宁地点信息生成器
  .container
    template(v-if="from_exists_doc")
      .row
        form.col.s12
          .file-field.input-field
            .btn
              span 上传文件
              input(type="file", @change="upload_file($event)")
            .file-path-wrapper
              input.file-path.validate(type="text")
    .row(v-else)
      span.col.s8 您正在创建一份新文档, 但您也可以
      button.col.s4.btn(@click="from_exists_doc = true") 从现有文件开始编辑
    .row
      .input-field.col.s8
        input#title_input(v-model="title", type="text")
        label(for="title_input") 地点名称
      .input-field.col.s4
        select#loctype_select
          option(value="unknown") 未知地点(请勿选择)
          option(value="pub")     公共场所
          option(value="gov")     政府机构
          option(value="party")   政党财产
          option(value="com")     商业地产
          option(value="pri")     个人房屋(请勿宣告所有权)
        label 地点类别
    .row
      .input-field.col.s6
        input#username_input(v-model="username", type="text")
        label(for="username_input") 尊姓大名
      .input-field.col.s6
        | 其他作者:
        .chip(v-for="a in authors")
          | {{a}}
    .row
      .col.s3
        input#wp_cb(v-model="with_pos", type="checkbox", @click="with_pos = !with_pos")
        label(for="wp_cb") 添加坐标信息
      template(v-if="with_pos")
        span.col.s6.grey-text 经度度为正东负西、纬度为正北负南
        a.col.s3.btn(target="_blank", :href="`https://23shiji.github.io/yining-map/#${pos_lat},${pos_lng},2`") 在地图中查看
    template(v-if="with_pos")
      .row
        .col.s3.input-field
          input#lat_val(v-model="pos_lat", type="number", min="-90", max="90")
          label(for="lat_val") 纬度
        .col.s2.input-field
          span(v-if="pos_lat == 0") 赤道
          span(v-if="pos_lat < 0") 南半球
          span(v-if="pos_lat > 0") 北半球
        .col.s3.input-field
          input#lng_val(v-model="pos_lng", type="number", min="-180", max="180")
          label(for="lng_val") 经度
        .col.s2.input-field
          span(v-if="pos_lng == 0") 本初子午线
          span(v-if="pos_lng < 0") 西半球
          span(v-if="pos_lng > 0") 东半球
        button.btn.blue.col.s2(@click="paste_pos") 粘贴坐标
    .row
      button.col.s2.btn(@click="save_file") 导出
      span.col.s2 Ver {{version.toFixed(1)}}
      span.col.s4(@click="save_now")
        span(v-if="autosave_time")
          | 已保存于 {{format_date(autosave_time)}}
        span(v-else)
          | 尚未保存
    .row
      mavon-editor.col.s12(v-model="markdown")
</template>

<script>
import FileSaver from 'file-saver'
import YAML from 'js-yaml'
import marked from 'marked'
const ALL_TYPES = [
  'unknown',
  'pub',
  'gov',
  'party',
  'com',
  'pri',
]
export default {
  data(){
    return {
      from_exists_doc: false,
      version: 0,
      loctype: 'unknown',
      authors: [],
      username: '',
      // tags: [],
      title:    '',
      markdown: '',
      with_pos: true,
      autosave_time: null,
      pos_lng: 0,
      pos_lat: 0,
    }
  },
  methods: {
    upload_file(evt){
      if(evt.target.files.length == 0){
        return
      }
      const file = evt.target.files[0]
      const reader = new FileReader()
      reader.onload = (evt) => {
        const result = evt.target.result
        const [meta, markdown] = YAML.safeLoadAll(result)
        const v_md = typeof(markdown) === "string"
        const {
          title,
          authors,
          version,
          type,
          pos
          // tags
        } = meta
        const v_authors = Array.isArray(authors)
        // const v_tags = Array.isArray(tags)
        const v_version = typeof(version) === "number" && version >= 0
        const v_type = ALL_TYPES.includes(type)
        const v_pos  = !pos || ( typeof(pos.lng) === "number" && typeof(pos.lat) === "number" )
        if(
            !v_md || 
            !v_authors || 
            // !v_tags || 
            !v_version 
            || !v_type || !v_pos){
          alert("非法文件格式")
          console.error(meta)
          console.error({v_md, v_authors, v_version, v_pos})
          return
        }
        this.title =    title
        this.authors =  authors
        this.version =  version + 1
        if(pos){
          this.with_pos = true
          this.pos_lat = pos.lat
          this.pos_lng = pos.lng
        }else{
          this.with_pos = false
        }
        // this.loctype =     type
        $("#loctype_select").val(type)
        // this.tags =     tags
        this.markdown = markdown
      }
      reader.readAsText(file)
    },
    save_file(){
      let type = $("#loctype_select").val()
      if(!this.username){
        alert("您还没填写名字")
        return
      }
      if(!this.title){
        alert("您还没填写地点名")
        return
      }
      if(type === 'unknown'){
        alert("不可以生成未知类型的地点")
        return
      }
      if(!this.markdown){
        alert("不可以生成空文件")
        return
      }
      console.log(type)
      let authors = []
      for(let a of this.authors){
        if(a != this.username){
          authors.push(a)
        }
      }
      authors.push(this.username)
      const meta = {
        title: this.title,
        authors,
        type,
        version: this.version,
        pos: this.with_pos ? {lat: parseFloat(this.pos_lat), lng: parseFloat(this.pos_lng)} : null
      }
      const date = new Date
      const text = this.preproc(this.markdown)
      const content = "---\n" + YAML.safeDump(meta) + "---\n" + YAML.safeDump(text)
      const filename = `${this.username}_${this.title}_v${this.version}.ymd`
      const blob = new Blob([content], {type: "text/plain;charset=utf-8"});
      FileSaver.saveAs(blob, filename);
    },
    preproc(text){
      return text.split(/\n+/).join("\n\n")
    },
    markdown_preview(text){
      return marked(this.preproc(text))
    },
    add_title(){
      this.markdown += "\n\n## 在此处添加段落标题\n"
      $("#tab_edit").val(this.markdown)
    },
    add_link(){
      this.markdown += "\n\n[链接显示文字](http://链接地址)\n"
      $("#tab_edit").val(this.markdown)
    },
    format_date(date){
      let d = new Date
      let timesep = (d.getTime() - date.getTime()) / 1000
      return `${(timesep / 60).toFixed(1)}分钟 前`
    },
    save_now(){
      localStorage['yipolis.map.editor.backup'] = this.markdown
      this.autosave_time = new Date
    },
    paste_pos(){
      let s = prompt("在此处输入坐标")
      if(!s) return
      let res = s.match(/(\d+\.?\d*)[^\dNS]*([NS]?)[^WESN\d]*(\d+\.?\d*)[^\dWE]*([WE]?)/)
      if(!res){
        alert("不是有效的坐标信息")
        return
      }
      let [_, lat_val, lat_sp, lng_val, lng_sp] = res
      lat_val = parseFloat(lat_val)
      lng_val = parseFloat(lng_val)
      if(lat_sp == "N") lat_val = lat_val
      else if(lat_sp == "S") lat_val = -lat_val
      else lat_val = 0
      if(lng_sp == "E") lng_val = lng_val
      else if(lng_sp == "W") lng_val = -lng_val
      else lng_val = 0
      this.pos_lat = lat_val
      this.pos_lng = lng_val
    }
  },
  mounted(){
    $(document).ready(function() {
      $('select').material_select();
      $('ul.tabs').tabs('select_tab', 'tab_edit');
    
      $("#loctype_select").val('unknown')
    });
  },
  created(){
    if (window.File && window.FileReader && window.FileList && window.Blob) {
      // pass
    } else {
      alert('您的浏览器无法支持文件上传功能')
    }
    this.markdown = localStorage['yipolis.map.editor.backup'] || ''
    setInterval(() => {
      if(this.markdown){
        localStorage['yipolis.map.editor.backup'] = this.markdown
        this.autosave_time = new Date
      }
    }, 90 * 1000)
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  font-size: 1.5rem;
}
#tab_edit{
  min-height: 40rem;
}
</style>
