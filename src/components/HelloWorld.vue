<template>
  <div>
    <div class="head" v-if="!linkSuccess">
      <el-input v-model="username" placeholder="请输入用户名"></el-input>
      <el-button type="primary" @click="linkToServer">连接服务器</el-button>
    </div>
    <div v-else>
      <div class="playPlace" @click="playGame($event)">
        <div v-for="(item, index) in qp" :key="index">
          <div
            v-for="(item2, index2) in item"
            :class="qp[index][index2] === -1 ? 'noQz' : qzClass[qp[index][index2]]"
            :style="{top: index * 30 + 20 + 'px', left: index2 * 30 + 20 + 'px'}"
            @click="sendPosition(index, index2)"
            :key="index2">
          </div>
        </div>
      </div>
      <div class="player">
        <div class="me">
          我：<br/>{{username}}<br/>
          {{!qzColor ? '' : (qzColor === 'black' ? '黑棋' : '白棋')}}
        </div>
        <div class="other">
          对手：<br/>{{otherName}}<br/>
          {{!qzColor ? '' : (qzColor === 'black' ? '白棋' : '黑棋')}}
        </div>
      </div>
      <div v-if="end" class="reBegin">
        <el-button type="primary" @click="reBegin">重新开始</el-button>
        <el-button class="end" type="primary" @click="closeLink">不玩了</el-button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      path: 'ws:127.0.0.1:4000', //websocket连接地址
      socket: null, //websocket
      username: '', //用户名
      linkSuccess: false, //是否连接成功
      messageArr: [], //服务器发来的消息记录
      qzClass: { //棋子的class,0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
        0: 'blackQz',
        1: 'whiteQz',
        2: 'blackQz_just',
        3: 'whiteQz_just'
      },
      otherName: '', //对手的名字
      qp: [], //棋盘(二维数组)
      qzColor: '', //棋子颜色
      turnToYou: false, //是否该你下棋了
      lastPosition: { //上一个棋子的位置
        x: -1,
        y: -1,
        color: ''
      },
      end: false //游戏是否结束
    }
  },
  mounted () {
    this.initQp()
  },
  methods: {
    //重置棋盘
    initQp () {
      this.qp = []
      for (let i = 0; i < 15; i++) {
        let arr = []
        for (let j = 0; j < 15; j++) {
          arr.push(-1)
        }
        this.qp.push(arr)
      }
    },
    init () {
      if (typeof(WebSocket) === 'undefined') {
        console.error('您的浏览器不支持socket')
      } else{
        // 实例化socket
        this.socket = new WebSocket(this.path)
        // 监听socket连接
        this.socket.onopen = this.open
        // 监听socket错误信息
        this.socket.onerror = this.error
        // 监听socket消息
        this.socket.onmessage = this.getMessage
      }
    },
    open () {
      console.log("socket连接成功")
      this.socket.send(JSON.stringify({
				username: this.username,
				mes: '发起连接'
			}))
    },
    error () {
      console.log("连接错误")
    },
    getMessage (msg) {
      let data = JSON.parse(msg.data)
      console.log('服务端发来的消息:', data)
      if (data.code === 500) {
        this.$message.error(data.text)
        this.closeLink()
        this.linkSuccess = false
      } else {
        this.messageArr.push(data)
        this.linkSuccess = true
        if (data.code === 202) { //服务器发过来的是对手的名字和棋子颜色
          this.otherName = data.text
          this.qzColor = data.qz
          if (this.qzColor === 'black') {
            this.turnToYou = true //黑棋先下
          } else {
            this.turnToYou = false
          }
        } else if (data.code === 204) { //发送棋子的位置
          const poArr = data.text.split(',')
          if (poArr[2] !== this.qzColor) { //服务器发来的颜色是对手的颜色,说明对手下棋了
            this.turnToYou = true
          }
          this.showOnQp(poArr[0], poArr[1], poArr[2])
        }
      }
    },
    close () {
      console.log("socket已经关闭")
    },
    linkToServer () {
      this.username = this.username.trim()
      if (!this.username) {
        this.$message.error('请输入用户名')
      } else {
        this.init()
      }
    },
    closeLink () { //关闭连接
      /*
        readyState属性返回实例对象的当前状态，共有四种。
        CONNECTING：值为0，表示正在连接。
        OPEN：值为1，表示连接成功，可以通信了。
        CLOSING：值为2，表示连接正在关闭。
        CLOSED：值为3，表示连接已经关闭，或者打开连接失败。
      */
      if (this.socket && this.socket.readyState && this.socket.readyState !== 3) {
        this.socket.send(JSON.stringify({
          username: this.username,
          mes: 'close'
        }))
        this.socket.close()
        console.log('关闭了连接')
      }
      //所有信息都清空或重置
      this.end = false
      this.socket = null
      this.username = ''
      this.linkSuccess = false
      this.messageArr = []
      this.initQp()
      this.qzColor = ''
      this.otherName = ''
      this.turnToYou = false
      this.lastPosition.x = -1
      this.lastPosition.y = -1
      this.lastPosition.color = ''
      console.log('socket:', this.socket)
    },
    playGame (event) { //点击棋盘(15*15)
      //位置是相对于页面左上角的点
      //一个格子大小为30px*30px
      //棋盘左上角的点的位置：clientX: 315, clientY: 35
      //棋盘右上角的点的位置：clientX: 740, clientY: 35
      //棋盘左下角的点的位置：clientX: 315, clientY: 460
      //棋盘右下角的点的位置：clientX: 740, clientY: 460
      console.log(event)
    },
    //将用户点击的坐标发送给服务器
    sendPosition (x, y) {
      if (this.qzColor <= 0) {
        this.$message.error('对手还没来哦,别急')
        return
      }
      if (!this.end) { //游戏未结束
        if (this.turnToYou) { //该你下棋了
          if (this.qp[x][y] !== -1) { //该坐标已经有棋子
            return
          }
          this.socket.send(JSON.stringify({
            username: this.username,
            mes: x + ',' + y + ',' + this.qzColor
          }))
          this.turnToYou = false
        } else {
          this.$message.error('莫慌,该对手了')
        }
      }
    },
    //服务器发来的棋子坐标,棋子颜色:0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
    showOnQp (x, y, qzColor) {
      if (this.lastPosition.x >= 0) { //将上一步的黑棋/白棋改成黑棋/白棋的样式
        let temp = this.qp[this.lastPosition.x]
        temp[this.lastPosition.y] = this.lastPosition.color === 'black' ? 0 : 1
        this.$set(this.qp, this.lastPosition.x, temp)
      }
      this.lastPosition.x = x
      this.lastPosition.y = y
      this.lastPosition.color = qzColor
      if (qzColor.length <= 0) {
        return
      } 
      //给二维数组设置值
      let temp = this.qp[x]
      temp[y] = qzColor === 'black' ? 2 : 3
      this.$set(this.qp, x, temp)
      //判断输赢
      const win = this.whoWin()
      if (win) {
        this.$message.success(win + '赢了')
        this.end = true
      }
    },
    //判断胜负
    //0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
    whoWin () {
      for (let i = 0; i < this.qp.length; i++) { //横着判断
        let row = this.qp[i] //每行的数据
        let blackNum = 0
        let whiteNum = 0
        for (let j = 0; j < row.length; j++) {
          let q = row[j]
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          if (blackNum === 5) {
            return '黑棋'
          } else if (whiteNum === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //竖着判断
        let blackNum = 0
        let whiteNum = 0
        for (let j = 0; j < 15; j++) {
          let q = this.qp[j][i]
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          if (blackNum === 5) {
            return '黑棋'
          } else if (whiteNum === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //左上角三角形的向右上判断
        let blackNum = 0
        let whiteNum = 0
        let blackNum2 = 0
        let whiteNum2 = 0
        for (let j = 0; j <= i; j++) {
          let q = this.qp[i-j][j] //左上角三角形的向右上判断
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          let q2 = this.qp[14-j][14-(i-j)] //右下角三角形的向右上判断
          if (q2 === -1) { //无棋子
            blackNum2 = 0
            whiteNum2 = 0
          } else if (q2 === 0 || q2 === 2) { //黑棋
            blackNum2++
            whiteNum2 = 0
          } else { //白棋
            whiteNum2++
            blackNum2 = 0
          }
          if (blackNum === 5 || blackNum2 === 5) {
            return '黑棋'
          } else if (whiteNum === 5 || whiteNum2 === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //左下角三角形的向右下判断
        let blackNum = 0
        let whiteNum = 0
        let blackNum2 = 0
        let whiteNum2 = 0
        for (let j = 0; j < 15 - i; j++) {
          let q = this.qp[i+j][j] //左下角三角形的向右下判断
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          let q2 = this.qp[j][i+j] //右上角三角形的向右下判断
          if (q2 === -1) { //无棋子
            blackNum2 = 0
            whiteNum2 = 0
          } else if (q2 === 0 || q2 === 2) { //黑棋
            blackNum2++
            whiteNum2 = 0
          } else { //白棋
            whiteNum2++
            blackNum2 = 0
          }
          if (blackNum === 5 || blackNum2 === 5) {
            return '黑棋'
          } else if (whiteNum === 5 || whiteNum2 === 5) {
            return '白棋'
          }
        }
      }
    },
    //重新开始
    reBegin () {
      this.end = false
      this.initQp()
      this.qzColor = ''
      this.otherName = ''
      this.turnToYou = false
      this.lastPosition.x = -1
      this.lastPosition.y = -1
      this.lastPosition.color = ''
      this.socket.send(JSON.stringify({
        username: this.username,
        mes: 'reBegin'
      }))
    },
  },
  destroyed () {
    // 销毁监听
    if (this.socket) {
      this.closeLink()
      this.socket.onclose = this.close
    }
  }
}
</script>

<style scoped>
.head {
  margin: 0 auto;
}
.el-input {
  width: 20%;
}
.playPlace {
  position: fixed;
  width: 450px;
  height: 450px;
  border: 1px solid red;
  top: 20px;
  left: 20px;
  background-image: url('../assets/five.jpg');
}
.player {
  position: fixed;
	top: 20px;
	left: 473px;
  width: 200px;
  height: 450px;
  border: 1px solid yellow;
}
.me {
  height: 223px;
  width: 100%;
  border: 1px solid green;
}
.other {
  height: 223px;
  width: 100%;
  border: 1px solid lightcoral;
}
.blackQz {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/black.png');
}
.whiteQz {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/white.png');
}
.noQz {
  width: 30px;
  height: 30px;
  position: fixed;
}
.blackQz_just {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/black_pick.png');
}
.whiteQz_just  {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/white_pick.png');
}
.reBegin {
  position: fixed;
  top: 480px;
	left: 20px;
}
.end {
  margin-left: 10px;
}
</style>
