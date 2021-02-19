<template>
	<view>
		<view align="center">
			<image class='esand_logo' src="../../static/logo.png"></image>
		</view>
		<view class="btn-row">
			<button type="primary" class="primary item" @tap="userReg()">用户注册</button>
		</view>

		<view class="btn-row">
			<button type="primary" class="primary item" @tap="startVerify()">刷脸认证</button>
		</view>

		<div align="center">
			<textarea :value="msg" />
		</div>
	</view>
</template>

<script>
 /* 
  * Esand-FaceIDModule 插件状态码(code字段)
     * 执行成功
  FACEID_SUCCESS,
     * 执行失败
  FACEID_FAILED,
     * 未获取到权限
  FACEID_PERMISSION,
     * 无效参数
  FACEID_INVALID_PARAM,
     * 执行抛异常
  FACEID_EXCEPTION;
  */
  let faceIDModule = uni.requireNativePlugin("Esand-FaceIDModule")
  // TODO 替换成您自己的APPCODE(切勿泄漏), 如何获取APPCODE,可参考：https://esandinfo.yuque.com/docs/share/13ad611e-b9c3-4cf8-a9a8-fe23a419312e?#
  let APPCODE = '替换为你的APPCODE'
  export default {
  	data() {
  		return {
  			msg: 'logs'
  		}
  	},
  	methods: {
    // 用户注册，此段逻辑应该放在业务服务器端(一个用户只注册一次即可)
    userReg: function(e) {
    	let _this=this;
    	uni.chooseImage({
	 		count: 1, //默认9
			sizeType: ['original', 'compressed'], //可以指定是原图还是压缩图，默认二者都有
	     	sourceType: ['album'], //从相册选择
	     	success: function (res) {
	     		var path = res.tempFilePaths[0];
	     		console.log(JSON.stringify(res.tempFilePaths[0]));
	     		console.log("path : " + path)
	  			// 具体服务器端的请求协议，可参考：https://market.aliyun.com/products/57000002/cmapi00042648.html
	  			uni.uploadFile({
	  				url: 'http://zimfaceid.market.alicloudapi.com/faceId/app/user/add',
	  				method: 'POST',
	  				filePath: path,
	  				name: 'image_ref1',
	  				header: {
	  					'Authorization': 'APPCODE ' + APPCODE,
						'X-Ca-Nonce': _this.randomString(8) // 防重放攻击
					},
					formData: {
						//每个人脸照片对应的uuid,请确保UUID唯一
						'uuid': '11599652555540',
					},
					success: (res) => {
						console.log(res);
					}})
	  		}
	  	})
    },
    // 开始进行实人认证，可参考文档：https://esandinfo.yuque.com/books/share/1b12aca9-d3d6-4011-ac9c-d379c84b71ab?# 
    startVerify: function(e) {
    	let _this = this;
		// 认证初始化
		let ret = faceIDModule.verifyInit();
	    /*
	    * ret.code 执行结果: FACEID_SUCCESS/FACEID_EXCEPTION
	    * ret.msg 结果描述：成功
		* ret.data 初始化json数据，直接透传给服务器即可
		*/
		console.log("初始化数据：" + JSON.stringify(ret));
		// 请求阿里云初始化接口获取token（为了保护APPCODE,次端逻辑应该放在服务器端）
		uni.request({
			url: 'http://zimfaceid.market.alicloudapi.com/faceId/app/init',
			method: 'POST',
			header: {
				'Authorization': 'APPCODE ' + APPCODE,
	  			'X-Ca-Nonce': _this.randomString(8),  // 防重放攻击
	  			'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
	  		},
	  		data: {
	  			//初始化接口返回参数
	  			'device_info': ret.data,
	  			// TODO 此处的UUID需要和用户注册时候的uuid一致(UUID代表用户的id,用于取注册时候上传的比对图片进行图片比对)
	  			'uuid': '11599652555540',
	 			 // 认证类型: 静默活体still    炫彩活体flash
	 			 'liveness_type': 'still',
	  			// 业务id（用于日志及业务跟踪，最好填入一个唯一值）
	  			'biz_no': _this.randomString(16)
	  		},
	  		success: (res) => {
	      	/*
	 		 * 认证初始化响应数据，如下
	  		 * {
	  		 "code": "0000",
	 		 "msg": "初始化成功",
	  		 "biz_token": "1596451807,c871b48a-0714-4855-a5e3-52ae90e588fe",
			  "requestId": "20200803185005692-sbhwae79",
			  "time_used": "406"
	  		}
	 		 * code：
	 		 *     0000 ： 成功
	  		 *     L998: 参数异常
	 		 *     L999: 业务异常（常见于注册时uuid对应的图片异常）
	 		 */
	  		//获取token成功
	  		console.log('网络请求成功' + JSON.stringify(res))
	  		var rsp = res.data;
	  		// 初始化成功
	  		if (rsp.code == '0000') {
	  		//唤起认证
	  		console.log(rsp.biz_token);
	  		faceIDModule.startVerify({
	   		//初始话返回的token
	   		'token': rsp.biz_token
	   	}, (ret) => {
		   /*
		    * ret.code 执行结果: FACEID_SUCCESS/FACEID_FAILED/FACEID_PERMISSION/FACEID_INVALID_PARAM/FACEID_EXCEPTION
		    * ret.msg 结果描述：成功
		    * ret.token 认证token: 
		    * ret.data 认证数据（数据量较大，数据存储在ret.file中）
		    * ret.file 认证数据：(要通过路径获取数据，需要在前面加 ’file:///‘)
		    */
		    console.log(ret);
		    console.log('插件调用成功: code: ' + ret.code + '; msg: ' + ret.msg + '; token: ' + ret.token +'; file: ' + ret.file )
		    uni.uploadFile({
		    	url: 'http://zimfaceid.market.alicloudapi.com/faceId/app/verify',
		    	method: 'POST',
		    	filePath: "file:///" + ret.file,
		    	name: 'meglive_data',
		    	header: {
		    		'Authorization': 'APPCODE ' + APPCODE,
		 			'X-Ca-Nonce': _this.randomString(8)  // 防重放攻击
		 		},
		 		formData: {
		  			//每个人脸照片对应的uuid,请确保UUID唯一
		  			'biz_token': ret.token,
		  			// 业务id（用于日志及业务跟踪，最好填入一个唯一值）
		  			'biz_no': _this.randomString(16)
		  		},
		  		success: (res) => {
				   /**
				   * 获取认证结果响应数据，如下
				   * {
				   "code":"", -- 执行状态码
				   "image_best":"", -- 活体最清晰的那张照片
				   "msg":"获取结果成功", -- 执行结果描述
				   "requestId":"20210201233534587-01o3t2xr", -- 请求ID(用于跟踪业务)
				   "time_used":"1421", -- 总耗时
				   "video_key":"ujBYG94rAtIe7wruw1Uymopys5Hzqybo" -- 活体视频解密密钥
				   }
				   * code：
				   *     0000 ： 成功（活体成功，并且人脸比对是本人）
				    *     L025: 不是本人（和注册上传的照片不是本人）
				   *     L026：活体检测失败
				   *     L998: 参数异常
				   */
				   let dataStr  = res.data
				   let data = JSON.parse(dataStr)
				   console.log("刷脸认证响应数据: " + dataStr)
				   _this.msg = dataStr;
				   if (data.code == '0000') {
		 		   // 获取活体认证视频
		 		   let decryptionKey = data.video_key;
		 		   let ret = faceIDModule.getLivenessVideo(decryptionKey)
					/*
					* ret.code 执行结果: FACEID_SUCCESS
					* ret.msg 结果描述：成功
					* ret.token 
					* ret.data 视频BASE64字符串：(数据量较大，ret.file)
					* ret.file 视频路径：(要通过路径获取数据，需要在前面加 ’file:///‘)
					*/
				}
			}
		});
		});
	  	}
	  }  
	})
	},
	randomString: function(len) {
		len = len || 32;
		var $chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
		var maxPos = $chars.length;
		var randomStr = '';
		for (let i = 0; i < len; i++) {
			randomStr += $chars.charAt(Math.floor(Math.random() * maxPos));
		}
		randomStr = 'FACEID_' + randomStr;
		return randomStr;
	}
}
}
</script>
<style>
	.content {
		position: absolute;
		width: 100%;
		height: 100%;
	}

	.esand_logo {
		margin-top: 100rpx;
		margin-bottom: 100rpx;
		width: 200rpx;
		height: 200rpx;
		align-self: center;
	}

	.item {
		margin-bottom: 15rpx;
		margin-top: 15rpx;
		width: 400rpx;
	}

	label {
		display: block;
		margin-top: 10rpx;
	}

	textarea {
		margin-top: 20rpx;
		height: 800rpx;
		font-size: small;
	}
</style>
