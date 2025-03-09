<template>
	<view class="content">
		<image class="logo" src="/static/logo.png"></image>
		<view>
			<text class="title">{{title}}</text>
		</view>
		<button type="primary" @click="uploadImg">上传图片</button>
		<ImageCropper v-if="imgSrc" mode="widthFix" img_width="300" img_height="300" :imgSrc="imgSrc" />
	</view>
</template>

<script>
import ImageCropper from "../../components/ImageCropper/index.vue";
	export default {
		components: {
			ImageCropper
		},
		data() {
			return {
				title: 'Hello',
				imgSrc: ''
			}
		},
		onLoad() {

		},
		methods: {
          // 上传图片
          uploadImg() {
            let that = this;
            uni.chooseImage({
              count: 1,
              sizeType: ['original', 'compressed'],
              sourceType: ['album', 'camera'],
              success: function(res) {
                that.imgSrc = res.tempFilePaths[0];
              }
            });
          }
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200rpx;
		width: 200rpx;
		margin: 200rpx auto 50rpx auto;
	}

	.text-area {
		display: flex;
		justify-content: center;
	}

	.title {
		font-size: 36rpx;
		color: #8f8f94;
	}
</style>
