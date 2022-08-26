<template>
	<view class="content">
		<view class="contentItem" v-for="item in packageInfos" @click="showDetail(item)">
			<img :src="item.iconBase64" alt="图片" class="img">
			<view class="title">
				{{item.appName}}
			</view>
		</view>
		<uni-popup ref="popup">
			<view class="popup-content">
				<uni-forms ref="baseForm">
					<uni-forms-item label="名称:">
						<text class="text">{{showItem.appName}}</text>
					</uni-forms-item>
					<uni-forms-item label="版本:">
						<text class="text">{{showItem.versionName}}（{{showItem.versionCode}}）</text>
					</uni-forms-item>
					<uni-forms-item label="包名:">
						<text class="text">{{showItem.packageName}}</text>
					</uni-forms-item>
					<uni-forms-item label="SHA1:">
						<text class="text">{{showItem.SHA1}}</text>
					</uni-forms-item>
					<!-- <uni-forms-item label="key:">
						<uni-easyinput type="textarea" v-model="showItem.key" placeholder="请输入key" />
					</uni-forms-item> -->
				</uni-forms>
			</view>
		</uni-popup>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				packageInfos: [],//除系统应用外的所有app
				showItem: {
					appName: '',
					versionName: '',
					versionCode: '',
					packageName: '',
					SHA1: '',
					key: '',//暂时没用
				}
			}
		},
		onLoad() {
			this.getApps()
		},
		methods: {
			showDetail(row) {
				this.showItem = row
				this.$refs.popup.open('center')
			},
			getApps() {
				//#ifdef APP-PLUS
				//转为16进制
				let Bytes2HexString = (arrBytes) => {
					let str = "";
					for (var i = 0; i < arrBytes.length; i++) {
						var tmp;
						var num = arrBytes[i];
						if (num < 0) {
							//当byte因为符合位导致数值为负时候，需要对数据进行处理  
							tmp = (255 + num + 1).toString(16);
						} else {
							tmp = num.toString(16);
						}
						if (tmp.length == 1) {
							tmp = "0" + tmp;
						}
						if (i !== arrBytes.length - 1) {
							tmp = tmp + ':'
						}
						str += tmp;
					}
					return str;
				}
				//通过包名获取该应用信息
				let getPackageInfo = (packageName) => {
					return pm.getPackageInfo(packageName, signaturesObj)
				}

				let Build = plus.android.importClass("android.os.Build");
				var Base64 = plus.android.importClass("android.util.Base64");
				var Bitmap = plus.android.importClass('android.graphics.Bitmap');
				var ByteArrayOutputStream = plus.android.importClass("java.io.ByteArrayOutputStream");
				var ApplicationInfo = plus.android.importClass('android.content.pm.ApplicationInfo')

				let pm = plus.android.runtimeMainActivity().getPackageManager();
				let PackageManager = plus.android.importClass(pm);
				let signaturesObj = PackageManager.GET_SIGNING_CERTIFICATES || PackageManager.GET_SIGNATURES
				let packages = pm.getInstalledPackages(signaturesObj);
				let len = plus.android.invoke(packages, 'size');
				for (let i = 0; i < len; i++) {
					// 安装包信息
					let packageInfo = plus.android.invoke(packages, 'get', i);
					let applicationInfo = plus.android.getAttribute(packageInfo, "applicationInfo");

					// 包名
					let packageName = plus.android.getAttribute(packageInfo, "packageName");

					let packageInfoItem = getPackageInfo(packageName)
					let isSysApk = packageInfoItem.plusGetAttribute('applicationInfo').plusGetAttribute("flags") & ApplicationInfo.FLAG_SYSTEM
					if (!isSysApk) {
						// 应用名称
						let appName = plus.android.invoke(applicationInfo, "loadLabel", pm)
						
						// 版本号
						let versionCode = plus.android.getAttribute(packageInfo, "versionCode");
						let versionName = plus.android.getAttribute(packageInfo, "versionName");
						// 图标
						let appIcon = plus.android.invoke(applicationInfo, "loadIcon", pm);
						//getBitmap
						let bimp = plus.android.invoke(appIcon, "getBitmap")
						let iconBase64 = null
						try {
							//Bitmap类型的图片转为base64，AdaptiveIconDrawable这种类型的需要转为Bitmap再转为base64，还没搞
							let baos = new ByteArrayOutputStream();
							bimp.compress(Bitmap.CompressFormat.JPEG, 5, baos);//5代表压缩质量，不怕卡可以调到100
							baos.flush();
							baos.close();
							let bitmapBytes = baos.toByteArray();
							iconBase64 = "data:image/jpeg;base64," + Base64.encodeToString(bitmapBytes, Base64.DEFAULT);
						} catch (e) {}
						
						let signatures = null,
							byteArr = null,
							type = 'SHA1',
							SHA1
						let md = plus.android.invoke("java.security.MessageDigest", "getInstance", type);
						//Android 28以后获取包签名信息方法改了
						if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
							let signingInfo = packageInfoItem.plusGetAttribute('signingInfo');
							signatures = plus.android.invoke(signingInfo, "getApkContentsSigners")
						} else {
							signatures = packageInfoItem.plusGetAttribute('signatures');
						}
						byteArr = plus.android.invoke(signatures[0], "toByteArray");
						plus.android.invoke(md, "update", byteArr);
						SHA1 = Bytes2HexString(plus.android.invoke(md, "digest")).toUpperCase();
						this.packageInfos.push({
							appName,
							packageName,
							versionName,
							versionCode,
							SHA1,
							iconBase64,
							key: ''
						})
					}
				}
				console.log('packageInfos',this.packageInfos.length);
				//#endif
			}
		},
	}
</script>

<style lang="scss">
	.contentItem {
		display: flex;
		align-items: center;
		margin: 10upx;
		padding-bottom: 5px;
		border-bottom: 1px solid #ccc;

		img {
			width: 15vw;
		}

		.title {
			margin-left: 10upx;
			font-size: 32upx;
		}
	}

	.popup-content {
		width: 80vw;
		height: 50vh;
		padding: 15px;
		background-color: #fff;

		.uni-forms-item {
			align-items: center;
			margin-bottom: 20upx;

			/deep/.uni-forms-item__content {
				width: 70%;
				white-space: pre-wrap;
				word-wrap: break-word;
			}

			.text {
				font-size: 30upx;
			}
		}
	}
</style>
