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
        input#title_input(v-model="title", type="text", :disabled="from_exists_doc")
        label(for="title_input", v-if="!from_exists_doc") 地点名称
      .input-field.col.s4(v-if="from_exists_doc")
        input#fake_type_input(disabled, type="text", :value="loc_type_info[loctype].name")
      .input-field.col.s4(v-show="!from_exists_doc")
        select#loctype_select
          template(v-for="t in types_list")
            option(:value="t.type") {{t.name}}
        label 地点类别
    .row.grey-text(v-if="loctype != 'unknown'")
      .col.s12 {{loc_type_info[loctype].name}}: {{loc_type_info[loctype].desc}}
    .row
      .input-field.col(:class="{s6: from_exists_doc, s12: !from_exists_doc}")
        input#username_input(v-model="username", type="text", @input = "change_username")
        label(for="username_input") 尊姓大名
      .input-field.col.s6(v-if="from_exists_doc")
        | 其他作者:
        template(v-for="a in authors")
          .chip(@click="username = a") {{a}}
    .row
      .col.s3
        input#wp_cb(v-model="with_pos", type="checkbox", @click="with_pos = !with_pos")
        label(for="wp_cb") 添加坐标信息
      template(v-if="with_pos")
        span.col.s6.grey-text 经度为正东负西、纬度为正北负南
        a.col.s3.btn(target="_blank", :href="`${planet_info[planet].url}#${pos_lat},${pos_lng},2`") 在地图中查看
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
      mavon-editor#editor.col.s12(v-model="markdown")
</template>

<script>
import types_list from '../types.yaml'
import planets_list from '../planets.yaml'
import save_file from './helpers/save_file'
import upload_file from './helpers/upload_file'
import paste_pos from './helpers/paste_pos'
let loc_type_info = {}
for(let t of types_list){
  loc_type_info[t.type] = t
}
let planet_info = {}
for(let p of planets_list){
  planet_info[p.planet] = p
}
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
      planet: 'yipolis',
      types_list,
      planets_list,
      loc_type_info,
      planet_info
    }
  },
  methods: {
    // splited files
    save_file,
    upload_file,
    paste_pos,

    change_username(){
      localStorage['yipolis.map.editor.username'] = this.username
    },
    preproc(text){
      return text.split(/\n+/).join("\n\n")
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
    change_type(type){
      this.loctype  = type
    }
  },
  mounted(){
    $(document).ready(() => {
      $('select').material_select();
      $('ul.tabs').tabs('select_tab', 'tab_edit');
    
      $("#loctype_select").on('change', evt => {
        this.change_type(evt.target.value)
      })

      $("#loctype_select").val('unknown')
    });
  },
  created(){
    if (window.File && window.FileReader && window.FileList && window.Blob) {
      // pass
    } else {
      alert('您的浏览器无法支持文件上传功能')
    }
    this.username = localStorage['yipolis.map.editor.username']
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
#editor{
  z-index: 10;
}
</style>
