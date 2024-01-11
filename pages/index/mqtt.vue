<template>
	<view class="container">
		<view class="uni-common-mt">
			<view class="uni-form-item uni-column">
				<view class="title">地址</view>
				<input class="uni-input" placeholder="输入地址" :value="connection.host" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">端口</view>
				<input class="uni-input" placeholder="输入端口" :value="connection.port" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">路径</view>
				<input class="uni-input" placeholder="输入路径" :value="connection.endpoint" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">客户端ID</view>
				<input class="uni-input" placeholder="输入客户端ID(可选)" :value="connection.clientId" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">用户名</view>
				<input class="uni-input" placeholder="输入用户名(可选)" :value="connection.username" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">密码</view>
				<input class="uni-input" placeholder="输入密码(可选)" :value="connection.password" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">存活时间</view>
				<input class="uni-input" placeholder="输入存活时间(默认60)" :value="connection.keep_alive" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">订阅主题</view>
				<input class="uni-input" placeholder="输入需要订阅的主题" :value="subscription.topic" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">发布主题</view>
				<input class="uni-input" placeholder="输入需要发布到的主题" :value="publish.topic" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title">发布内容</view>
				<input class="uni-input" placeholder="输入需要发布的内容(常为json)" :value="publish.payload" />
			</view>
			<view class="uni-form-item uni-column">
				<view class="title"></view>
				<checkbox-group class="cbox" @change="changeUrl">
					<view>
						<label class="cbox-label">
							<checkbox value="CleanSession" checked="true" />清除会话
						</label>
						<label class="cbox-label">
							<!-- #ifndef MP-WEIXIN -->
							<checkbox value="SSL" />SSL
							<!-- #endif -->
							<!-- #ifdef MP-WEIXIN -->
							<checkbox checked="true" disabled />SSL
							<!-- #endif -->
						</label>
					</view>
					<view>
						<label>
							<text class="uri-text">{{uri}}</text>
						</label>
					</view>
				</checkbox-group>
			</view>
			<view class="btn-group">
				<!-- 连接 -->
				<button v-if="!connected" @click="createConnection" class="btn-conn" type="default">连接</button>
				<button v-else class="btn-conn" type="default" disabled>连接</button>
				<button @click="destroyConnection" type="warn">断开连接</button>
				<view class="state-text">
					<text>当前状态:</text>
					<text v-if="!connected" style="color: rgb(255, 109, 109);">未连接</text>
					<text v-else style="color: rgb(66, 216, 133);">已连接</text>
				</view>
			</view>
			<view class="btn-group">
				<!-- 获取主题信息 -->
				<button @click="doSubscribe" type="primary">订阅主题</button>
				<button @click="doUnSubscribe" type="warn">取消订阅</button>
				<button disabled></button>
			</view>
			<view class="msg_list">
				<text>消息列表</text>
				<p v-for="(item,index) in msgs">{{item.name}}：{{item.msg}}
					<hr>
				</p>
				
			</view>
		</view>
	</view>
</template>
                
              

<script setup>
	import * as mqtt from 'mqtt/dist/mqtt.js';
	import {
		reactive,
		ref,
		onMounted
	} from "vue";

	// https://github.com/mqttjs/MQTT.js#qos
	const qosList = [0, 1, 2];
	let client = ref({
		connected: false
	});
	const receivedMessages = ref("");
	const subscribedSuccess = ref(false);
	const btnLoadingType = ref("");
	const retryTimes = ref(0);

	let protocal = "";
	//如你的链接是wss:则修改为wxs:,如果你的链接是ws:则修改为wx:
	// #ifdef H5
	protocal = "ws"
	// #endif
	// #ifdef MP-WEIXIN
	protocal = "wx"
	// #endif
	const connection = reactive({
		// ws or wss
		protocol: protocal,
		host: "jqrjq.cn",
		// ws -> 8083; wss -> 8084
		port: 8083,
		clientId: "emqx_vue3_" + Math.random().toString(16).substring(2, 8),
		/**
		 * By default, EMQX allows clients to connect without authentication.
		 * https://docs.emqx.com/en/enterprise/v4.4/advanced/auth.html#anonymous-login
		 */
		username: "admin",
		password: "public",
		clean: true,
		connectTimeout: 30 * 1000, // ms
		reconnectPeriod: 4000, // ms
		// for more options and details, please refer to https://github.com/mqttjs/MQTT.js#mqttclientstreambuilder-options
	});
	const handleOnReConnect = () => {
		retryTimes.value += 1;
		if (retryTimes.value > 5) {
			try {
				client.value.end();
				initData();
				console.log("connection maxReconnectTimes limit, stop retry");
			} catch (error) {
				console.log("handleOnReConnect catch error:", error);
			}
		}
	};
	const createConnection = () => {
		try {
			const {
				protocol,
				host,
				port,
				...options
			} = connection;
			const connectUrl = `${protocol}://${host}:${port}/mqtt`;

			/**
			 * if protocol is "ws", connectUrl = "ws://broker.emqx.io:8083/mqtt"
			 * if protocol is "wss", connectUrl = "wss://broker.emqx.io:8084/mqtt"
			 * 
			 * /mqtt: MQTT-WebSocket uniformly uses /path as the connection path,
			 * which should be specified when connecting, and the path used on EMQX is /mqtt.
			 * 
			 * for more details about "mqtt.connect" method & options,
			 * please refer to https://github.com/mqttjs/MQTT.js#mqttconnecturl-options
			 */
			client.value = mqtt.connect(connectUrl, options);

			if (client.value.on) {
				// https://github.com/mqttjs/MQTT.js#event-connect
				client.value.on("connect", () => {
					btnLoadingType.value = "";
					console.log("connection successful");
				});

				// https://github.com/mqttjs/MQTT.js#event-reconnect
				client.value.on("reconnect", handleOnReConnect);

				// https://github.com/mqttjs/MQTT.js#event-error
				client.value.on("error", (error) => {
					console.log("connection error:", error);
				});

				// https://github.com/mqttjs/MQTT.js#event-message
				client.value.on("message", (topic, message) => {
					receivedMessages.value = receivedMessages.value.concat(
						message.toString()
					);
					console.log(`received message: ${message} from topic: ${topic}`);
				});
			}
		} catch (error) {
			console.log("mqtt.connect error:", error);
		}
	};

	onMounted(() => {
		createConnection();
	})
</script>

<style>

</style>