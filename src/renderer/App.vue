<template>
    <div class="container">
        <h2>Help Content Manager</h2>

        <div @dragover="dragOver" @drop="dropOver" class="alert-dark py-5 px-3" id="holder">
            Drag your directory here
        </div>

        <div class="row my-3">
            <div class="col-3">

            </div>
            <div class="col-9">
                <button type="button" class="btn btn-success" @click="addRow()">Add</button>
                <button type="button" class="btn btn-success" @click="validateAndSave()">Save</button>
            </div>
        </div>

        <div class="row">
            <div class="col-3">
                <h4>File List</h4>
                <ul class="list-group">
                    <li v-for="(table, index) in tableData" :key="index"
                        class="list-group-item list-group-item-action d-flex justify-content-between align-items-center"
                        :class="{active: index === activeIndex}"
                        @click="activeIndex = index">
                        <a>{{table.fileName}}</a>
                        <span v-if="getTableErrorCount(index) > 0" class="badge badge-danger badge-pill">{{getTableErrorCount(index)}}</span>
                    </li>
                </ul>
            </div>
            <div class="col-9">
                <template v-for="(table, tableIndex) in tableData">
                    <table v-if="tableIndex === activeIndex" :key="'table-' + tableIndex" class="table table-bordered">
                        <thead>
                        <tr>
                            <th>Key</th>
                            <th>Help Text</th>
                            <th>UI Note</th>
                            <th>Action(s)</th>
                        </tr>
                        </thead>
                        <tbody>
                        <tr v-for="(d, index) in table.data"
                            :title="d.error && d.error.length > 0 ? d.error.join(',') : ''"
                            :class="{'table-warning' : d.error && d.error.length > 0}" :key="index">
                            <td>
                                <app-inline-edit :ref="'key'" v-model="d.key"/>
                                <span>{{d.data}}</span>
                            </td>
                            <td>
                                <app-inline-edit v-model="d.text"/>
                            </td>
                            <td>
                                <app-inline-edit v-model="d.description"/>
                            </td>
                            <td>
                                <button type="button" class="btn btn-danger" @click="deleteRow(tableIndex, index)">
                                    Delete
                                </button>
                            </td>
                        </tr>
                        </tbody>
                    </table>
                </template>

            </div>

        </div>


    </div>
</template>

<script>
  import AppInlineEdit from './components/AppInlineEditor'
  import { each, isArray } from 'lodash'
  import Vue from 'vue'
  import { ipcRenderer } from 'electron'

  export default {
    name: 'help-manager',
    components: {
      AppInlineEdit
    },
    data: () => ({
      test: 1,
      tableData: [],
      activeIndex: 0
    }),
    methods: {
      getTableErrorCount (index) {
        let tableData = this.tableData[index]
        let count = 0
        each(tableData.data, (data) => {
          if (data.error && data.error.length > 0) {
            count++
          }
        })
        return count
      },
      deleteRow (tableIndex, index) {
        let tableData = this.tableData[tableIndex]
        tableData.data.splice(index, 1)
      },
      addRow () {
        this.tableData[this.activeIndex].data.push({key: 'key' + this.tableData[this.activeIndex].data.length})
        this.$nextTick(() => {
          console.log(this.$refs['key'])
          this.$refs['key'][this.tableData[this.activeIndex].data.length - 1].handleDivClick()
        })
      },
      dragOver: function (e) {
        console.log('dragOver')
        e.preventDefault()
        e.stopPropagation()
      },
      dropOver: function (e) {
        e.preventDefault()
        e.stopPropagation()
        this.tableData.splice(0)
        for (let f of e.dataTransfer.items) {
          let item = f.webkitGetAsEntry()
          if (item && !item.isFile) {
            console.log(item)
            this.traverseFileTree(item)
          } else {
            alert('Please drag and drop directory with help files')
          }
        }
      },
      traverseFileTree (item, path) {
        path = path || ''
        let jsonExt = /\.json$/i
        if (item.isFile) {
          // Get file
          item.file((file) => {
            if (jsonExt.test(file.name)) {
              console.log('File:', path + file.name, file.path)
              this.readFile(file, path)
            }
          })
        } else if (item.isDirectory) {
          // Get folder contents
          let dirReader = item.createReader()
          dirReader.readEntries((entries) => {
            for (let i = 0; i < entries.length; i++) {
              this.traverseFileTree(entries[i], path + item.name + '/')
            }
          })
        }
      },
      readFile: function (file, path) {
        let reader = new FileReader()
        reader.addEventListener('load', () => {
          console.log(reader.result)
          let jsondata = JSON.parse(reader.result || '{}')
          let resultObj = {}
          resultObj.filePath = file.path
          resultObj.fileName = file.name
          resultObj.data = []
          each(jsondata, (value, key) => {
            resultObj.data.push({
              ...value,
              key: key
            })
          })
          this.tableData.push(resultObj)
        })
        reader.readAsText(file)
      },
      validateAndSave () {
        let keyMap = {}
        let errCnt = 0
        each(this.tableData, (table, tableIndex) => {
          each(table.data, (dataPoint, dIndex) => {
            let keyData = keyMap[dataPoint.key]
            if (keyData !== undefined) {
              let {tableIndex, dataIndex} = keyData
              if (!isArray(this.tableData[tableIndex].data[dataIndex].error)) {
                Vue.set(this.tableData[tableIndex].data[dataIndex], 'error', [])
              }

              this.tableData[tableIndex].data[dataIndex].error.push(`Duplicate Key: ${table.fileName} dataIndex: ${dIndex + 1}`)

              if (!isArray(dataPoint.error)) {
                Vue.set(dataPoint, 'error', [])
              }

              dataPoint.error.push(`Duplicate Key: ${keyData.fileName} dataIndex: ${keyData.dataIndex + 1}`)
              errCnt++
            } else {
              keyMap[dataPoint.key] = {
                tableIndex: tableIndex,
                fileName: table.fileName,
                dataIndex: dIndex,
                error: []
              }
            }
          })
        })

        if (errCnt === 0) {
          // save to file
          each(this.tableData, (tableData, tableIndex) => {
            let savedata = {}
            each(tableData.data, (dataPoint, index) => {
              savedata[dataPoint.key] = {
                text: dataPoint.text,
                description: dataPoint.description
              }
            })
            if (ipcRenderer.sendSync('save', savedata, tableData.filePath) !== true) {
              alert(`failed to save file ${tableData.filePath}`)
            }
          })
          alert('save operation done.')
        } else {
          alert('Duplicate Keys found. Resolve key conflict.')
        }
      }
    }
  }
</script>

<style scoped>
    .container .th {
        width: 25%;
    }

    .list-group-item {
        word-wrap: break-word;
    }
</style>
