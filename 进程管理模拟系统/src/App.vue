<template>
  <div id="app">
	  <div v-show="logShow" class="logShow">
		  <div style="width: 85vw; height: 80vh; align-items: center;">
			  <i class="el-icon-close" style="float: right;margin-bottom: 0.5rem;" v-on:click="logShow = false"></i>
			  <textarea style="width: 99%; height: 95%; font-size: 1.25rem;" readonly="true">{{serverInfo.log}}</textarea>
		  </div>
	  </div>
	<el-row :gutter="20">
		<el-col :span="4">
			<div class="left" v-bind:style="serverInfo.connected == false?'background-color: #F56C6C;':'background-color: #409EFF'">
				<ul>
					<li>
						<el-button v-on:click="connectServer" round v-if="serverInfo.connected == false" class="buttonLeft">连接服务器</el-button>
						<el-button v-on:click="disconnect" round v-if="serverInfo.connected == true" type="danger" class="buttonLeft">断开连接</el-button>
					</li>
					<li>
						<el-button v-on:click="showLog" round class="buttonLeft" v-bind:disabled="buttonsCanClick?false:'disabled'">查看系统日志</el-button>
					</li>
					<li>
						<el-button v-on:click="fifo" round class="buttonLeft"  v-bind:disabled="buttonsCanClick?false:'disabled'">运行FIFO</el-button>
					</li>
					<li>
						<el-button v-on:click="sjf" round class="buttonLeft"  v-bind:disabled="buttonsCanClick?false:'disabled'">运行SJF</el-button>
					</li>
					<li>
						<el-button v-on:click="rr" round class="buttonLeft"  v-bind:disabled="buttonsCanClick?false:'disabled'">运行RR</el-button>
					</li>
				</ul>
			</div>
		</el-col>
		<el-col :span="20">
			<div class="right">
				<div style="width: 100%; height: 50%;">
					<el-card v-bind:style="setPCBCardBkColor(item)" class="PCB" v-for="(item, index) in PCBList" :key='index' shadow="always">
						<div style="text-align: center;" >PCB {{item.pid}}</div>
						<div>name: {{item.name}}</div>
						<div>priority: {{item.priority}}</div>
						<div>status: {{item.processStatus}} </div>
						<div>serviceTime: {{item.serviceTime}}</div>
						<div>arriveTime: {{item.arriveTime}}</div>
						<div>runTime: {{item.runTime}}</div>
						<div>finishTime: {{item.finishTime}}</div>
						<div>requestTime: {{item.requestTime}}</div>
					</el-card>
				</div>
				<div style="height: 50%; margin-top: 1.25rem;">
					<div v-for="(items, index) in queues" :key='index' style="margin-bottom: 1.5rem;">
						<div v-show="index == 0">等待队列</div>
						<div v-show="index == 1">就绪队列</div>
						<div v-show="index == 2">运行队列</div>
						<el-card style="width: 100%; height: 12.5rem; margin-top: 1em;">
							<el-card v-bind:style="setPCBCardBkColor(item)" class="PCB_small" v-for="(item, index) in items" :key='index' shadow="always">
								<div style="text-align: center;" >PCB {{item.pid}}</div>
								<div>priority: {{item.priority}}</div>
								<div>status: {{item.processStatus}} </div>
								
							</el-card>
						</el-card>
					</div>
				</div>
			</div>
		</el-col>
	</el-row>
   
  </div>
</template>

<script>
export default {
	data() {
		return {
			socket: "",
			path: "ws://127.0.0.1:7000",
			//path: "ws://192.168.0.105:7000",
			requestbody: {
				msg: "123"
			},
			serverInfo: {
				connected: false,
				show: "未连接",
				log: ""
			},
			PCBList:[],
			buttonsCanClick: false,
			logShow: false,
			queues:{
			},
			algorithmFinished: false,
			PCBCardStatus: [
				{
					Status: "FINISH",
					style: "background-color: #67C23A;"
				},
				{
					Status: "WAIT",
					style: "background-color: #FFD08A;"
				},
				{
					Status: "RUN",
					style: "background-color: #F56C6C;"
				},
				{
					Status: "READY",
					style: "background-color: #78B8FB"
				},
			]
		}
		
	},
  methods: {
	  disconnect() {
		  this.socket.close();
		  this.serverInfo.connected = false;
		  this.buttonsCanClick = false;
	  },
	  onOpen(evt) {
		  this.serverInfo.connected = true;
		  this.buttonsCanClick = true;
		  this.requestbody.header = "PCB";
		  this.requestbody.msg = "";
		  this.send(this.requestbody);
		  this.showQueue();
	  },
	  onMessage(evt) {
		let res = JSON.parse(evt.data);
		console.log(res);
		switch(res.header) {
			case "log":
				this.serverInfo.log = res.msg;
				break;
			case "systemError":
				console.log(res.msg)
				this.serverInfo.log = res.msg;
				break;
			case "PCB":
				this.PCBList = JSON.parse(res.msg);
				console.log(this.PCBList);
				break;
			case "queues":
				console.log(res.msg);
				this.queues = JSON.parse(res.msg);
			case "finished":
				this.algorithmFinished = true;
				
				
		}
	  },
	  onError(evt) {
		  console.log(evt.data);
	  },
	  onClose(evt) {
		  this.serverInfo.connected = false;
		  this.serverInfo.show = "未连接";
	  },
	  sjf() {
		  this.requestbody.header = "processpool";
		  this.requestbody.msg = "sjf";
		  this.send(this.requestbody);
	  },
	  rr() {
	  		  this.requestbody.header = "processpool";
	  		  this.requestbody.msg = "rr";
	  		  this.send(this.requestbody);
	  },
	  send(obj) {
		this.socket.send(JSON.stringify(obj));  
	  },
	  setPCBCardBkColor(item) {
		  console.log(item.processStatus);
		  if(item.processStatus == "READY") {
			  return this.PCBCardStatus[3].style;
		  }
		  switch(item.processStatus.toString()) {
			case "FINISH":
				return this.PCBCardStatus[0].style;
			case "WAIT":
				return this.PCBCardStatus[1].style;
			case "RUN":
				return this.PCBCardStatus[2].style;
			case "READY":
				return this.PCBCardStatus[3].style;
			default: return "";
		  }
	  },
	 showLog() {
		this.requestbody.header = "log";
		this.send(this.requestbody);
		this.logShow = true;
		
	 },
	 showQueue() {
		this.requestbody.header = "processpool";
		this.requestbody.msg = "queue_show";
		this.send(this.requestbody);
		
	 },
	 fifo() {
		 this.algorithmFinished = false;
		 this.requestbody.header = "processpool";
		 this.requestbody.msg = "fifo";
		 this.send(this.requestbody);
	 },
	 connectServer() {
		 this.socket = new WebSocket(this.path);
		 this.socket.onopen = this.onOpen;
		 this.socket.onmessage = this.onMessage;
		 this.socket.onerror = this.onError;
		 this.socket.onclose = this.onClose;
	 }
  },
  created() {
	  this.socket.onopen
	  this.socket.onmessage
  }
}
</script>

<style>
body {
	margin: 0;
	padding: 0;
}
ul li {
	list-style: none;
	
	
}
.PCB_small {
	height: 9.375rem;
	width: 8.5rem;
	margin-left: 1.75rem;
	position: relative;
	float: left;
}
.text {
    font-size: 14px;
  }

  .item {
    padding: 18px 0;
  }

.PCB {
	height: 12.5rem;
	width: 18.75rem;
	position: relative;
	float: left;
	margin-left: 2rem;
	margin-top: 2rem;
	text-align: left;
	padding: 0.4rem;
}
.right {
	width: auto;
	height: 100vh;
}

.logShow {
	background-color: rgba(0, 0, 0, 0.4);
	height: 100vh;
	width: 100vw;
	position: absolute;
	z-index: 2;
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	
}
#app {
  font-family: Helvetica, sans-serif;
  text-align: center;
}
.left {
	width: auto;
	height: 120vh;
	transition: 1s linear; 

	
}
.tag {
	width: 6.25rem;
}
.buttonLeft {
	width: 70%;
	float: left;
	margin-top: 1rem;
	overflow: hidden;
}
</style>
