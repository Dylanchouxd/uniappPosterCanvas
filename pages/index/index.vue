<template>
	<view class="index-wrapper">
		<button @click="createPoster">点击生成</button>
		
		<view class="qrcode-wrapper" v-if="shareQrcodeUrl">
			<image class="qrcode-save__image" :src="shareQrcodeUrl" mode="aspectFit"></image>
			<view class="qrcode-save__btn">
				<button text="保存图片" @click="saveShareQrcode">保存图片</button>
			</view>
		</view>
		
		<canvas
			:style="{
				visibility: 'hidden',
				zIndex: '-1',
				left: '-99999px',
				top: '-999999px',
				position: 'absolute',
				background: '#fff',
				width: `${shareQrcodeInfo.canvasWidth}rpx`,
				height: `${shareQrcodeInfo.canvasHeight}rpx`
			}"
			id="qrCodeCanvas"
			canvas-id="qrCodeCanvas"
		/>
	</view>
</template>

<script>
	const bufferToJson = (arraybuffer) => {
		try {
			const uint8Arr = new Uint8Array(arraybuffer);
			const encodedString = String.fromCharCode.apply(null, uint8Arr);
			const decodedString = decodeURIComponent(escape(encodedString))
			
			return JSON.parse(decodedString)
		} catch(e) {
			return {}
		}
	}
	const bufferToBase64 = (arraybuffer) => {
		return `data:image/jpeg;base64,${uni.arrayBufferToBase64(arraybuffer)}`;
	}
	/**
	 * 绘制文字自动换行
	 * @param String canvas ctx对象
	 * @param String text 绘制的文字
	 * @param Number x , y 绘制的坐标
	 * @param Number maxWidth 绘制文字的宽度
	 * @param Number lineHeight 行高
	 * @param Number maxRowNum 最大行数
	 */
	const canvasWraptitleText = (canvas, text, x, y, maxWidth, lineHeight, maxRowNum) => {
		if (typeof text != 'string' || typeof x != 'number' || typeof y != 'number') {
			return;
		}
		// canvas.font = '20px Bold PingFang SC'; //绘制文字的字号和大小
		// 字符分隔为数组
		var arrText = text.split('');
		var line = '';
		var rowNum = 1
		for (var n = 0; n < arrText.length; n++) {
			var testLine = line + arrText[n];
			var metrics = canvas.measureText(testLine);
			var testWidth = metrics.width;
			if (testWidth > maxWidth && n > 0) {
				if (rowNum >= maxRowNum) {
					var arrLine = line.split('')
					arrLine.splice(-1)
					var newTestLine = arrLine.join("")
					newTestLine += "..."
					canvas.fillText(newTestLine, x, y);
					//如果需要在省略号后面添加其他的东西，就在这个位置写（列如添加扫码查看详情字样）
					//canvas.fillStyle = '#2259CA';
					//canvas.fillText('扫码查看详情',x + maxWidth-90, y);
					return
				}
				canvas.fillText(line, x, y);
				line = arrText[n];
				y += lineHeight;
				rowNum += 1
			} else {
				line = testLine;
			}
		}
		canvas.fillText(line, x, y);
	}
	
	/**
	 * 注：
	 * 如果wxAccessToken有值则会用该值去请求接口；
	 * 如没有Token则需要填写 wxAppid、wxAppSecret，会自动请求Token方便后续使用。
	 * 
	 * 但是请注意！！！该操作仅适用于与测试，生产的话还需要后端/云函数去获取Token和生成微信小程序码；
	 */
	export default {
		data() {
			return {
				wxAppid: '',  // 微信小程序 appid
				wxAppSecret: '',  // 微信小程序 appsecret
				wxAccessToken: '', // 微信小程序 accessToken （如果填了这个值，则可以直接使用生成二维码）
				createQrcodeInfo: {
					page: 'pages/index/index',
					scene: 'msg=HelloWorld',
					env_version: 'develop' // 正式版为 release，体验版为 trial，开发版为 develop
				},
				shareQrcodeInfo: {
					canvasWidth: 650,
					canvasHeight: 700,
					title: '都城大饭店',
					desc: '😋目标是吃上都城大饭店最贵的菜',
					qrCode: '',
					tips: '长按或扫描二维码，立刻加入群聊'
				},
				shareQrcodeUrl: ''
			}
		},
		onLoad() {

		},
		methods: {
			async createPoster() {
				uni.showLoading({
					title: '加载中'
				})
				
				if (!this.wxAccessToken) {
					this.wxAccessToken = await this.getWxAccessToken()
				}
				
				// 1. 生成微信小程序二维码，并临时保存到本地
				const wxQrcode = await this.createWxQrcode(this.createQrcodeInfo)
				// 由于小程序canvas无法绘制base64图片，所以先保存到本地，绘制完再删除
				const filePath = `${wx.env.USER_DATA_PATH}/tmp-share-qrcode-${Date.now()}`
				const imageData = wxQrcode.replace(/^data:image\/\w+;base64,/, "");
				const fs = uni.getFileSystemManager()
				fs.writeFileSync(filePath, imageData, 'base64')
				
				this.shareQrcodeInfo.qrCode = filePath
				
				// 2. 绘制Canvas画布内容（标题、描述、SLOGEN，二维码，提示语）
				this.drawShareQrcode(
					// 3. Canvas画布转临时地址
					() => {
						uni.canvasToTempFilePath({
							canvasId: 'qrCodeCanvas',
							quality: 1,
							fileType: 'jpg',
							success: (res) => {
								this.shareQrcodeUrl = res.tempFilePath
								uni.hideLoading()
								
								// 删除临时保存的文件
								fs.unlinkSync(filePath)
							}
						})
					}
				)
			},
			/**
			 * 获取WxAccessToken
			 * https://developers.weixin.qq.com/miniprogram/dev/api-backend/open-api/access-token/auth.getAccessToken.html
			 * @description 请勿用于生产，AccessToken应该是由 后端/云函数 来获取
			 */
			async getWxAccessToken() {
				if (!this.wxAppid || !this.wxAppSecret) {
					console.error('必须要有 wxAppid, wxAppSecret 才可以获取微信AccessToken')
					return
				}
				
				return await uni.request({
					method: 'GET',
					url: 'https://api.weixin.qq.com/cgi-bin/token',
					data: {
						appid: this.wxAppid,
						secret: this.wxAppSecret,
						grant_type: 'client_credential'
					}
				})
					.then((data) => {
						const [err, res] = data;
						
						if (err) {
							console.log(err)
							uni.hideLoading()
							throw new Error('获取WxAccessToken错误')
						}
						
						return res.data.access_token
					})
			},
			/**
			 * 生成微信小程序二维码
			 * https://developers.weixin.qq.com/miniprogram/dev/api-backend/open-api/qr-code/wxacode.getUnlimited.html
			 * @description 请勿用于生产，生成微信小程序二维码应该是由 后端/云函数 去生成，因为前端不允许直接调用api.weixin.qq.com
			 * @param {string} scene 页面参数
			 * @param {string} page 页面路径
			 * @param {string} env_version 打开的小程序版本
			 * @param {...} 更多参数请自行根据文档扩展
			 * @return {string} 二维码临时路径
			 */
			async createWxQrcode({scene, page, env_version}) {
				return await uni.request({
					url: `https://api.weixin.qq.com/wxa/getwxacodeunlimit?access_token=${this.wxAccessToken}`,
					method: 'POST',
					data: {
						scene,
						page,
						env_version
					},
					responseType: "arraybuffer", // 返回是arraybuffer格式
				})
					.then((data) => {
						const [err, res] = data;
						
						if (err) {
							console.log(err)
							uni.hideLoading()
							throw new Error('生成微信小程序二维码错误')
						}
						
						const jsonRes = bufferToJson(res.data)
						if (!jsonRes.errcode) {
							return bufferToBase64(res.data)
						} else {
							return null
						}
					})
			},
			/**
			 * 绘制Canvas画布内容
			 */
			drawShareQrcode(callback) {
				const ctx = uni.createCanvasContext('qrCodeCanvas')
				const canvasInfo = this.shareQrcodeInfo
				const upx2px = uni.upx2px
				
				// 设置背景色
				ctx.setFillStyle('#fff')
				ctx.fillRect(0, 0, upx2px(canvasInfo.canvasWidth), upx2px(canvasInfo.canvasHeight))
				
				// 绘制群信息模块
				// 群标题
				ctx.font = `bold ${upx2px(54)}px arial`
				ctx.setFillStyle('#000')
				canvasWraptitleText(ctx, canvasInfo.title, upx2px(40), upx2px(100), upx2px(canvasInfo.canvasWidth) - upx2px(70), upx2px(60), 1)
				// 群描述
				ctx.font = `${upx2px(42)}px arial`
				ctx.setFillStyle('#999')
				canvasWraptitleText(ctx, canvasInfo.desc, upx2px(40), upx2px(190), upx2px(canvasInfo.canvasWidth) - upx2px(70), upx2px(60), 2)
				// Logo
				ctx.font = `bold ${upx2px(46)}px arial`
				ctx.setFillStyle('#000')
				ctx.fillText('群搜搜', upx2px(canvasInfo.canvasWidth - 180), upx2px(canvasInfo.canvasHeight - 325))
				ctx.font = `${upx2px(28)}px arial`
				ctx.setFillStyle('#999')
				ctx.fillText('海量微信群，免费加入，发布！', upx2px(canvasInfo.canvasWidth - 410), upx2px(canvasInfo.canvasHeight - 270))
				
				// 绘制底部二维码模块
				const footerWidth = upx2px(canvasInfo.canvasWidth)
				const footerHeight = upx2px(canvasInfo.canvasHeight - 220)
				
				// 绘制虚线
				ctx.setLineDash([7, 10], 0)
				ctx.beginPath()
				ctx.moveTo(0, footerHeight)
				ctx.lineTo(footerWidth, footerHeight)
				ctx.setStrokeStyle('#d8d8d8')
				ctx.stroke()
				
				// 绘制二维码图片
				const qrCodeSize = upx2px(150)
				const qrCodeTop = footerHeight + upx2px(40)
				const qrCodeLeft = upx2px(35)
				ctx.drawImage(canvasInfo.qrCode, qrCodeLeft, qrCodeTop, qrCodeSize, qrCodeSize)
				// 绘制二维码提示
				const tipsWidth = qrCodeLeft + qrCodeSize + upx2px(18)
				ctx.font = `${upx2px(26)}px arial`
				ctx.setFillStyle('#666')
				ctx.fillText(canvasInfo.tips, tipsWidth, qrCodeTop + qrCodeSize / 2 + 4)
				
				ctx.draw(false, () => {
					callback()
				})
			},
			/**
			 * 4. 保存图片功能
			 */
			saveShareQrcode() {
				uni.saveImageToPhotosAlbum({
					filePath: this.shareQrcodeUrl,
					success: () => {
						uni.showToast({
							title: '保存成功'
						})
					}
				});
			}
		}
	}
</script>

<style lang="scss" scoped>
	.index-wrapper {
		padding: 30rpx;
		.qrcode-wrapper {
			text-align: center;
			margin-top: 30rpx;
			.qrcode-save__image {
				width: 650rpx;
				height: 700rpx;
			}
			.qrcode-save__btn {
				margin-top: 20rpx;
			}
		}
	}
</style>
