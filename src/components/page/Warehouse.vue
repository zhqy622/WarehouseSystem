<template>
  <div>
    <div class="container">
      <div class="form-box">
        <el-form ref="form" :model="data" label-width="80px">
          <el-form-item label="仓库名称">
            <el-input class="el-form1" v-model="data.name" :disabled="component.editable"></el-input>
            <el-button class="btn-online" type="primary" @click="handleOnline($event)" :disabled="component.onlineable">
              {{ component.onlineval }}
            </el-button>
            <el-tag class="status" :type="data.status === 'online' ? 'success' : data.status === 'offline' ? 'warning' : ''">
              {{ data.status }}
            </el-tag>
          </el-form-item>
          <el-form-item label="仓库地址">
            <el-input class="el-form1" v-model="data.address" :disabled="component.editable"></el-input>
          </el-form-item>
          <el-form-item label="最大容量">
            <el-input class="el-form1" v-model="data.maxSize" :disabled="component.editable"></el-input>
          </el-form-item>
          <el-form-item label="当前库存">
            <el-input class="el-form1" v-model="data.currSize" disabled></el-input>
            <el-button class="btn-add" type="primary" icon="el-icon-lx-add" @click="handleAdd()" :disabled="component.addable">{{
              component.addval
            }}</el-button>
            <el-button type="danger" icon="el-icon-lx-move" @click="handleSub()" :disabled="component.subable">{{
              component.subval
            }}</el-button>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" :disabled="component.submitable" @click="handleSubmit()">提交修改</el-button>
            <el-button @click="handleCancle()" :disabled="component.cancelable">取消</el-button>
          </el-form-item>
        </el-form>
      </div>
    </div>
  </div>
</template>

<script>
import { fetchData } from '../../api/index'
export default {
  name: 'baseform',
  data() {
    return {
      id: 10,
      client: {},
      data: {
        id: -1,
        name: '',
        address: '',
        maxSize: null,
        currSize: -1,
        status: ''
      },
      component: {
        editable: false,
        addable: true,
        subable: true,
        submitable: false,
        cancelable: false,
        onlineable: false,
        addval: '商品入库',
        subval: '商品出库',
        onlineval: '运行',
        timer: null,
        time: 0
      },
      server: {},
      websocket: {},
      heartbeattimer: null,
      messagetype: {
        type_client_online: 0,
        type_client_offline: 1,
        type_client_heartbeat: 2,
        type_client_update: 3,
        type_client_load_add: 4,
        type_client_load_sub: 5,
        type_client_info: 6
      }
    }
  },
  created() {
    this.id = this.$route.params.id
    this.getData(this.id)
  },
  mounted() {
    // console.log("我被调用了")
  },
  destroyed() {
    for (var id in this.client) {
      // console.log(id)
      this.saveStatus(id, true)
      if (this.client[id]['websocket'] != null) {
        this.client[id]['websocket'].close()
        this.client[id]['websocket'] = null
      }
    }
  },
  watch: {
    //var that=this
    $route(to, from) {
      // if(from.name=='warehouse'&&this.data.status=='online')
      // {
      //    // this.saveStatus()
      // }
      // for(var key in this.client)
      // {
      //     console.log("key="+key)
      //     console.log(this.client[key]['component'])
      // }
      //saveStatus()
      if (from.name == 'warehouse') {
        var nowid = this.id
        this.client[nowid]['data'] = JSON.parse(JSON.stringify(this.data))
        this.client[nowid]['component'] = JSON.parse(JSON.stringify(this.component))
        this.saveStatus(nowid, false)
      }
      if (to.name == 'warehouse') {
        this.id = to.params.id
        var id = this.id
        if (this.client[id] == null) this.getData(id)
        else {
          //console.log(this.id)
          this.data = this.client[id]['data']
          //console.log(this.client[this.id]['component'])
          this.component = this.client[id]['component']
        }
        // console.log("to 的id："+id)
        // console.log(this.client[id]['component'])
      }
    }
  },
  methods: {
    getval(id, type, key) {
      return this.client[id][type][key]
    },
    onSubmit() {
      this.$message.success('提交成功!')
    },
    getData(id) {
      //var id=this.id
      //this.id=id
      if (this.client[id] == null) {
        this.client[id] = {
          data: {},
          component: {},
          websocket: null,
          server: {},
          heartbeattimer: null
        }
      }
      this.client[id]['data'] = this.data
      this.client[id]['component'] = this.component
      var url = 'client/' + id
      var method = 'get'
      fetchData(url, method, null).then(res => {
        if (res.data != null) {
          this.client[id]['data'] = res.data
          this.data = this.client[id]['data']
          // console.log(this.client[this.id]['data'])
          //this.client[this.]
          var serverapi = 'server'
          var method = 'get'
          fetchData(serverapi, method, null).then(res => {
            if (res.data != null) this.client[id]['server'] = res.data
            // console.log(res.data)
          })

          if (this.client[id]['data']['status'] == 'offline') {
            this.setOffline(id)
            //this.component=this.client[this.id]['component']
          } else if (this.client[id]['data']['status'] == 'online') {
            this.setOnline(id)
            //this.component=this.client[this.id]['component']
          }
        } else {
          this.client[id]['data'] = {}
          this.$message.error('数据非法')
          this.$router.go(-1)
        }
        // console.log(res.list);
      })

      //  }
      //this.data=this.client[this.id]['data']
    },
    saveStatus(id, offline) {
      var url =
        'save/' + this.client[id]['data']['id'] + '/' + this.client[id]['data']['status'] + '/' + this.client[id]['data']['currSize']
      if (offline) url = 'save/' + this.client[id]['data']['id'] + '/offline/' + this.client[id]['data']['currSize']
      var method = 'post'
      fetchData(url, method, this.client[id]['data']).then(res => {
        if (res.code == 10000) {
        } // this.$message.success('状态自动保存成功')
        else this.$message.error(res.msg)
      })
    },
    setOnline(id) {
      this.client[id]['data']['status'] = 'online'
      this.client[id]['component']['editable'] = true
      this.client[id]['component']['onlineval'] = '停止'
      this.client[id]['component']['addable'] = false
      this.client[id]['component']['subable'] = false
      this.client[id]['component']['submitable'] = true
      this.client[id]['component']['cancelable'] = true
      this.client[id]['component']['onlineable'] = false
      this.component = this.client[id]['component']
    },
    setOffline(id) {
      this.client[id]['data']['status'] = 'offline'
      this.client[id]['component']['editable'] = false
      this.client[id]['component']['onlineval'] = '运行'
      this.client[id]['component']['addable'] = true
      this.client[id]['component']['subable'] = true
      this.client[id]['component']['submitable'] = false
      this.client[id]['component']['cancelable'] = false
      this.client[id]['component']['onlineable'] = false
      this.component = this.client[id]['component']
    },
    handleOnline(e) {
      var id = this.id
      this.client[id]['component']['onlineable'] = true
      this.client[id]['component']['onlineval'] = 'waiting...'
      if (this.client[id]['data']['status'] == 'offline') {
        // console.log('我被执行了')
        console.log(this.client)
        var wsapi = 'ws://' + this.client[id]['server']['ip'] + this.client[id]['server']['api'] + '/' + id
        this.client[id]['websocket'] = new WebSocket(wsapi)
        this.client[id]['websocket'].onmessage = this.websocketonmessage
        this.client[id]['websocket'].onopen = this.websocketonopen
        this.client[id]['websocket'].onclose = this.websocketonclose
        this.client[id]['websocket'].onerror = this.websocketonerror
        var url = 'status/' + id + '/online'
        var method = 'post'
        //后端请求上线后设置
        if (true) {
          //打开websocket成功后，修改上线状态
          fetchData(url, method, null).then(res => {
            if (res.code == 10000) {
              //上线成功
              this.setOnline(id)
            } else {
              this.setOffline(id)
              //断开websocket连接或者重试
            }
          })
        }
      } else if (this.client[id]['data']['status'] == 'online') {
        if (this.client[id]['websocket'] != null) {
          this.client[id]['websocket'].close()
          clearInterval(this.client[id]['heartbeattimer'])
          this.client[id]['heartbeattimer'] = null
        }
        var url = 'status/' + id + '/offline'
        var method = 'post'
        if (true) {
          //关闭websocket成功后，修改上线状态
          fetchData(url, method, null).then(res => {
            if (res.code == 10000) {
              //上线成功
              this.setOffline(id)
            } else {
              this.setOnline(id)
              //断开websocket连接或者重试
            }
          })
        }
      }
    },
    handleAdd() {
      var id = this.id
      if (this.client[id]['data']['currSize'] < this.client[id]['data']['maxSize']) {
        this.client[id]['component']['addable'] = true
        this.client[id]['component']['subable'] = true
        this.client[id]['component']['time'] = 0
        this.addval = this.time + 's后可用'
        if (!this.client[id]['component']['timer']) {
          this.client[id]['component']['timer'] = setInterval(() => {
            if (this.client[id]['component']['time'] > 0) {
              this.client[id]['component']['time']--
              this.client[id]['component']['addable'] = true
              this.client[id]['component']['subable'] = true
              this.client[id]['component']['addval'] = this.client[id]['component']['time'] + 's后可用'
            } else {
              this.client[id]['component']['time'] = 0
              this.client[id]['component']['addable'] = false
              this.client[id]['component']['subable'] = false
              this.client[id]['component']['addval'] = '商品入库'
              clearInterval(this.client[id]['component']['timer'])
              this.client[id]['component']['timer'] = null
            }
          }, 1000)
        }
        this.client[id]['data']['currSize']++
        var message = {
          type: this.messagetype.type_client_load_add,
          data: {
            currSize: this.client[id]['data']['currSize']
          }
        }
				console.log(message)
        this.client[id]['websocket'].send(JSON.stringify(message))
        this.$message.success('入库成功')
      } else {
        this.$message.error('入库失败,请稍后重试')
      }
    },
    handleSub() {
      var id = this.id
      if (this.client[id]['data']['currSize'] > 0) {
        this.client[id]['component']['addable'] = true
        this.client[id]['component']['subable'] = true
        this.client[id]['component']['time'] = 0
        this.subval = this.time + 's后可用'
        if (!this.client[id]['component']['timer']) {
          this.client[id]['component']['timer'] = setInterval(() => {
            if (this.client[id]['component']['time'] > 0) {
              this.client[id]['component']['time']--
              this.client[id]['component']['addable'] = true
              this.client[id]['component']['subable'] = true
              this.client[id]['component']['subval'] = this.client[id]['component']['time'] + 's后可用'
            } else {
              this.client[id]['component']['time'] = 0
              this.client[id]['component']['addable'] = false
              this.client[id]['component']['subable'] = false
              this.client[id]['component']['subval'] = '商品出库'
              clearInterval(this.client[id]['component']['timer'])
              this.client[id]['component']['timer'] = null
            }
          }, 1000)
        }
        this.client[id]['data']['currSize']--
        var message = {
          type: this.messagetype.type_client_load_sub,
          data: {
            currSize: this.client[id]['data']['currSize']
          }
        }
        this.client[id]['websocket'].send(JSON.stringify(message))
        this.$message.warning('出库成功')
        this.client[id]['component']['subable'] = true
      } else {
        this.$message.error('仓库为空，出库失败')
      }
    },
    handleSubmit() {
      var id = this.id
      console.log(this.client[id]['data'])
      var url = 'update/'
      var method = 'post'
      fetchData(url, method, this.client[id]['data']).then(res => {
        if (res.code == 10000) this.$message.success('修改成功')
        else this.$message.error(res.msg)
        // this.submitable=false
      })
    },
    handleCancle() {
      var id = this.$route.params.id
      var url = 'client/' + id
      var method = 'get'
      fetchData(url, method, null).then(res => {
        if (res.data != null) {
          this.client[id]['data'] = res.data
          if (this.client[id]['data']['status'] == 'offline') {
            this.client[id]['component']['onlineable'] = false
            this.client[id]['component']['onlineval'] = '运行'
            this.client[id]['component']['editable'] = false
            this.client[id]['component']['submitable'] = false
          } else {
            this.client[id]['component']['onlineable'] = false
            this.client[id]['component']['onlineval'] = '停止'
            this.client[id]['component']['editable'] = true
            this.client[id]['component']['submitable'] = true
          }
          this.data = this.client[id]['data']
          this.component = this.client[id]['component']
        } else {
          this.client[id]['data'] = {}
          this.$message.error('数据非法')
          this.$router.go(-1)
        }
      })
    },
    websocketonopen(e) {
      console.log(e)
    },
    websocketonclose(e) {
      console.log(e)
      for (var key in this.client) if (e.target == this.client[key]['websocket']) var id = key
      this.$message.warning('您已掉线')
      if (this.client[id]['websocket'] != null) {
        this.client[id]['websocket'].close()
        clearInterval(this.client[id]['heartbeattimer'])
        this.client[id]['heartbeattimer'] = null
      }
      console.log('status:' + this.client[id]['websocket'].readyState)
      this.setOffline(id)
      this.saveStatus(id, true)
      console.log('连接关闭了')
    },
    websocketonmessage(e) {
			console.log(e)
      var res = JSON.parse(e.data)
      for (var key in this.client) if (e.target == this.client[key]['websocket']) var id = key
      var type = res.type
      switch (type) {
        case this.messagetype.type_heartbeat: 
          var heartbeat = {
            type: 2,
            data: null
          }
          this.client[id]['heartbeattimer'] = setInterval(() => {
            if (this.client[id]['websocket'] != null && this.client[id]['websocket'].readyState == 1) {
              this.client[id]['websocket'].send(JSON.stringify(heartbeat))
              console.log('id ' + id + ' send heartbeat')
            } else {
              clearInterval(this.client[id]['heartbeattimer'])
              this.client[id]['heartbeattimer'] = null
            }
          }, 100000)
          break
        case this.messagetype.type_client_info:
          var info = {
            type: this.messagetype.type_client_info,
            data: {
              name: this.client[id]['data']['name'],
              currSize: this.client[id]['data']['currSize'],
              address: this.client[id]['data']['address'],
              maxSize: this.client[id]['data']['maxSize']
            }
          }
          if (this.client[id]['websocket'] != null && this.client[id]['websocket'].readyState == 1) {
            this.client[id]['websocket'].send(JSON.stringify(info))
            console.log('send:' + JSON.stringify(info))
          }
        default:
          break
      }
    },
    websocketonerror(e) {
			console.log(e)
		}
  }
}
</script>
<style scoped>
.el-form1 {
  width: 50%;
}
.btn-online {
  margin-left: 20px;
}
.btn-add {
  margin-left: 20px;
}
.status {
  margin-left: 10px;
}
</style>
