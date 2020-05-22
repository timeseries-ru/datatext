<template>
  <div id="app">
    <vue-simplemde v-model="content" ref="editor" :configs="configs"/>
    <div ref="nodes" style="display: none;"></div>
  </div>
</template>

<script>
import Vue from 'vue/dist/vue.esm.js'
import VueSimplemde from 'vue-simplemde'
import Chartkick from 'vue-chartkick'
import Chart from 'chart.js'

import * as alasql from '../node_modules/alasql/dist/alasql.min.js'
const csvParser = require('csv-parser')

// require to not optimize by builder
/* eslint-disable no-unused-vars */
const sqlite_requirement = require('sqlite3')
const mysql_requirement = require('mysql')
/* eslint-enable no-unused-vars */

Vue.use(Chartkick.use(Chart))

const LineChart = (charted, xtitle, ytitle) => Vue.extend({
  data () {
    return { charted, xtitle, ytitle }
  },
  template: `<line-chart class="padded" :data="charted" :xtitle="xtitle" :ytitle="ytitle"></line-chart>`
});

const BarChart = (charted, xtitle, ytitle) => Vue.extend({
  data () {
    return { charted, xtitle, ytitle }
  },
  template: `<column-chart class="padded" :data="charted" :xtitle="xtitle" :ytitle="ytitle"></column-chart>`
});

const ScatterChart = (charted, xtitle, ytitle) => Vue.extend({
  data () {
    return { charted, xtitle, ytitle }
  },
  template: `<scatter-chart class="padded" :data="charted" :xtitle="xtitle" :ytitle="ytitle"></scatter-chart>`
});

var fs = require('fs');
const { dialog } = require('electron').remote;

export default {
  name: 'app',
  components: {
    'vue-simplemde': VueSimplemde
  },
  data () {
    return {
      content: "",
      configs: {
        autofocus: true,
        spellChecker: false,
        status: false,
        toolbar: [
          'bold', 'italic', 'strikethrough', 'heading',
          '|',
          'code', 'quote', 'unordered-list', 'ordered-list',
          '|',
          'link', 'image', 'table', 'horizontal-rule',
          '|',
          'preview',
          '|',
          {
            name: 'new',
            action: function customFunction(editor) {
              const result = dialog.showMessageBoxSync({
                type: "question",
                message: "All unsaved data will be lost. Proceed?",
                buttons: ["Create new", "Cancel"]
              })
              if (result === 0) {
                editor.value('');
              }
            },
            className: 'fa fa-plus',
            title: 'New Markdown'
          },
          {
            name: 'load',
            action: function customFunction(editor) {
              var options = {
                title: "Open file",
                buttonLabel: "Load",
                filters: [
                  {name: 'md', extensions: ['md',]},
                  {name: 'All Files', extensions: ['*']}
                ]
              }
              dialog.showOpenDialog(options, result => {
                if (result)
                  editor.value(fs.readFileSync(result[0], 'utf-8'));
              })
            },
            className: 'fa fa-file',
            title: 'Open Markdown'
          },
          {
            name: 'save',
            action: function customFunction(editor) {
              var options = {
                title: "Save file",
                defaultPath: "content.md",
                buttonLabel: "Save",
                filters: [
                  {name: 'md', extensions: ['md',]},
                  {name: 'All Files', extensions: ['*']}
                ]
              }
              dialog.showSaveDialog(options, filename => {
                fs.writeFileSync(filename, editor.value(), 'utf-8');
              })
            },
            className: 'fa fa-save',
            title: 'Save Markdown'
          },
          '|',
          {
            name: 'Export',
            action: editor => {
              var options = {
                title: "Export file",
                defaultPath: "content.html",
                buttonLabel: "Save",
                filters: [
                  {name: 'html', extensions: ['html',]},
                  {name: 'All Files', extensions: ['*']}
                ]
              }
              dialog.showSaveDialog(options, filename => {
                fs.writeFileSync(filename, this.previewRender(editor.value(), null, true), 'utf-8');
              })
            },
            className: 'fa fa-download',
            title: 'Export with images'
          }
        ],
        previewRender: this.previewRender
      },
      mirror: null,
      blocks: []
    }
  },
  mounted () {
    this.mirror = this.$refs.editor.simplemde.codemirror
  },
  watch: {
    content () {
      this.checkBlocks()
    }
  },
  methods: {
    async getDataBlock (text) {
      var code = text.trim()
      const db_names = {
        'sqlite': 'sqlite3',
        'mysql': 'mysql',
        'sqlite3': 'sqlite3',
        'csv': 'csv'
      }
      var db = db_names[code.toLowerCase().trim().split(' ')[0]]
      if ((db !== 'sqlite3') && (db !== 'mysql') && (db !== 'csv'))
        return { error: 'Unsupported DB type' };
      let liner = (line, index) => line.trim().split(' ').filter(word => word.trim().length)[index];
      let connection
      let codes = code.split(';')
      let csvData = []
      try {
        if (db !== 'csv') {
          if (db === 'sqlite3' && !fs.existsSync(liner(codes[0], 1))) return {
            error: 'Database not found'
          };
          connection = require('knex')({
            'client': db,
            connection: db !== 'sqlite3' ? {
              host: liner(codes[0], 1),
              database: liner(codes[0], 2),
              user: liner(codes[0], 3),
              password: liner(codes[0], 4)
            } : {
              filename: liner(codes[0], 1)
            },
            useNullAsDefault: db === 'sqlite3'
          })
        } else {
          let objects = []
          let stream = fs.createReadStream(liner(codes[0], 1)).pipe(csvParser())
          await new Promise(resolve => {
            stream.on('data', data => objects.push(data))
            stream.on('end', resolve)
          })
          csvData = objects
        }
      } catch (error) {
        return { error: error.toString() }
      }
      let plots = []
      for (let index = 2; index < codes.length; ++index) {
        if (!codes[index]) continue;
        let type = liner(codes[index], 0)
        let x, y, group
        if (liner(codes[index], 1)) x = liner(codes[index], 1).split(',')[0]
        if (liner(codes[index], 2)) y = liner(codes[index], 2).split(',')[0]
        if (liner(codes[index], 3)) group = liner(codes[index], 3).split(',')[0]
        plots.push({
          type, x, y, group
        })
      }
      try {
        if (codes[1].trim().substring(0, 6) !== 'select') return {
          error: 'Only SQL SELECT is supported'
        };
        if (plots) {
          for (var plot in plots) {
            if (!plot.type) continue
            if (!['lines', 'bars', 'scatter'].includes(plot.type.toLowerCase())) {
              return { error: 'Unknown plot type: ' + plot.type }
            }
          }
        }
        let data = db === 'csv' ?
          await alasql.promise(codes[1].replace('  ', ' ').replace('from csv', 'from ?'), [csvData]) :
          await connection.raw(codes[1]);
        if (db === 'mysql') {
          data = data[0]
        }
        return { data, plots }
      } catch (error) {
        return { error: error.toString() }
      }
    },
    async processBlock (index, current, line) {
      if (index < this.blocks.length) {
        if (this.blocks[index].code === current) return;
        for (let error of this.blocks[index].errors || []) {
          error.node.remove();
          error.clear();
        }
        if (this.blocks[index].widget) {
          this.blocks[index].widget.node.remove();
          this.blocks[index].widget.clear();
        }
      }
      let data = await this.getDataBlock(current)

      if (index === this.blocks.length) this.blocks.push({})
      if (data.error || !data.data) {
        let node = document.createElement('div')
        if (data.error)
          node.innerHTML = '<span style="color: red">' + data.error + '</span>';
        else
          node.innerHTML = '<span style="color: black">No records</span>';
        this.blocks[index].errors = []
        this.blocks[index].errors.push(this.mirror.doc.addLineWidget(
          this.mirror.doc.getLineHandle(line), node
        ))
        this.blocks[index].code = current
        return
      }

      var node = document.createElement('div')
      this.$refs.nodes.appendChild(node)

      if (this.blocks[index].widget) {
        this.blocks[index].widget.node.remove();
        this.blocks[index].widget.clear();
      }

      let components = []
      for (var plot of data.plots || []) {
        if (!plot.type) continue
        var component
        var chart
        const type = plot.type.toLowerCase()
        const keys = Object.keys(data.data[0])
        if (keys.length < 2) continue;
        if (!plot.x && !plot.y && keys[2]) {
          plot.group = keys[2]
        }
        if (!plot.x) plot.x = keys[0]
        if (!plot.y) plot.y = keys[1]

        var inner = document.createElement('div')
        node.appendChild(inner)

        if (plot.group) {
          chart = []
          const groups = new Set(data.data.map(entry => entry[plot.group]))
          for (let group of Array.from(groups)) {
            let current = type === 'lines' ? {} : []
            if (type === 'lines') {
              for (let entry of data.data) {
                if (group === entry[plot.group]) current[entry[plot.x]] = entry[plot.y]
              }
            } else {
              for (let entry of data.data) {
                if (group === entry[plot.group]) current.push([
                  entry[plot.x], entry[plot.y]
                ])
              }
            }
            chart.push({
              name: group,
              data: current
            })
          }
        } else {
          if (type === 'lines') {
            chart = {}
            for (let entry of data.data) {
              chart[entry[plot.x]] = entry[plot.y]
            }
          } else {
            chart = []
            for (let entry of data.data) {
              chart.push([entry[plot.x], entry[plot.y]])
            }
          }
        }
        if (type === 'lines') {
          component = new (LineChart(chart, plot.x, plot.y))().$mount(inner)
        } else if (type === 'bars') {
          component = new (BarChart(chart, plot.x, plot.y))().$mount(inner)
        } else if (type === 'scatter') {
          component = new (ScatterChart(chart, plot.x, plot.y))().$mount(inner)
        }
        if (!component) {
          inner.remove()
          continue
        }
        components.push(component.$root.$children[0].chart)
      }
      this.blocks[index].components = components
      this.$nextTick(() => {
        this.blocks[index].widget = this.mirror.doc.addLineWidget(
          this.mirror.doc.getLineHandle(line), node
        )
        if (this.blocks[index].errors)
        for (let error of this.blocks[index].errors) {
          error.node.remove();
          error.clear();
        }
        this.blocks[index].errors = []
        this.blocks[index].code = current
      })
    },
    checkBlocks () {
      let opened = false
      let current = ''
      let counter = 0
      for (let index = 0; index < this.mirror.doc.lineCount(); ++index) {
        var text = this.mirror.doc.getLine(index) + '\n'
        if (text.trim() === '```') {
          if (opened) {
            this.processBlock(counter, current, index);
            ++counter
          }
          opened = !opened
          current = ''
          continue
        }
        if (opened) {
          current += text
        }
      }
    },
    previewRender(content, element, add_styles) {
      let original = content
      let opened = false
      let current = ''
      let counter = 0
      for (let index = 0; index < this.mirror.doc.lineCount(); ++index) {
        var text = this.mirror.doc.getLine(index) + '\n'
        if (text.trim() === '```') {
          if (opened) {
            let images = ''
            for (let component of this.blocks[counter].components) {
              images += '![](' + component.chart.toBase64Image() + ')\n'
            }
            original = original.replace(current, images)
            ++counter
          }
          opened = !opened
          current = ''
          continue
        }
        if (opened) {
          current += text
        }
      }
      const styles = `
        <style>
          * { margin: 0; padding: 0; }
          body { width: 80%; background: #fefefe; margin: 10px auto; }
          img { display: block; margin: 5px auto; max-width: 100%; }
          p {
            padding: 10px 0;
          }
        </style>
      `
      const rendered = this.$refs.editor.simplemde.markdown(original.replace(/```/g, ''))
      if (!add_styles)
        return rendered
      return styles + rendered
    }
  }
}
</script>

<style>
@import '~typeface-roboto/index.css';
html, body {
  margin: 0;
  font-family: Roboto !important;
}
div {
  border-radius: 0!important;
}
@import '~simplemde/dist/simplemde.min.css';
.CodeMirror {
  min-height: calc(100vh - 74px);
  height: calc(100vh - 74px);
}
a.fa {
  outline: 0 !important;
}
.padded {
  padding: 10px 20px;
  max-width: calc(100% - 40px);
}
.editor-preview {
  background: white;
}
</style>
