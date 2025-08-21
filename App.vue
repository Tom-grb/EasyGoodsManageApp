<script>
	export default {
		onLaunch: function() {
			console.log('App Launch')
		},
		onShow: function() {
			console.log('App Show')
		},
		onHide: function() {
			console.log('App Hide')
		}
	}
</script>

<style>
	/* 全局样式 */
	page {
		background: #f8f9fa;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
	}
	
	/* 通用样式 */
	.container {
		min-height: 100vh;
		background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
	}
	
	/* 按钮样式 */
	.btn {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		padding: 20rpx 40rpx;
		border-radius: 50rpx;
		font-size: 28rpx;
		font-weight: 500;
		text-decoration: none;
		border: none;
		cursor: pointer;
		transition: all 0.3s ease;
	}
	
	.btn-primary {
		background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
		color: #ffffff;
	}
	
	.btn-secondary {
		background: linear-gradient(135deg, #6c757d 0%, #495057 100%);
		color: #ffffff;
	}
	
	.btn-danger {
		background: linear-gradient(135deg, #dc3545 0%, #c82333 100%);
		color: #ffffff;
	}
	
	/* 卡片样式 */
	.card {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
		margin-bottom: 20rpx;
	}
	
	/* 输入框样式 */
	.input {
		width: 100%;
		height: 80rpx;
		border: 2rpx solid #e9ecef;
		border-radius: 10rpx;
		padding: 0 20rpx;
		font-size: 28rpx;
		background: #f8f9fa;
		transition: border-color 0.3s ease;
	}
	
	.input:focus {
		border-color: #007bff;
		outline: none;
	}
	
	/* 文本样式 */
	.text-primary {
		color: #007bff;
	}
	
	.text-secondary {
		color: #6c757d;
	}
	
	.text-success {
		color: #28a745;
	}
	
	.text-danger {
		color: #dc3545;
	}
	
	.text-muted {
		color: #adb5bd;
	}
	
	/* 间距样式 */
	.mt-10 { margin-top: 10rpx; }
	.mt-20 { margin-top: 20rpx; }
	.mt-30 { margin-top: 30rpx; }
	.mb-10 { margin-bottom: 10rpx; }
	.mb-20 { margin-bottom: 20rpx; }
	.mb-30 { margin-bottom: 30rpx; }
	.ml-10 { margin-left: 10rpx; }
	.ml-20 { margin-left: 20rpx; }
	.mr-10 { margin-right: 10rpx; }
	.mr-20 { margin-right: 20rpx; }
	
	.pt-10 { padding-top: 10rpx; }
	.pt-20 { padding-top: 20rpx; }
	.pt-30 { padding-top: 30rpx; }
	.pb-10 { padding-bottom: 10rpx; }
	.pb-20 { padding-bottom: 20rpx; }
	.pb-30 { padding-bottom: 30rpx; }
	.pl-10 { padding-left: 10rpx; }
	.pl-20 { padding-left: 20rpx; }
	.pr-10 { padding-right: 10rpx; }
	.pr-20 { padding-right: 20rpx; }
	
	/* 布局样式 */
	.flex {
		display: flex;
	}
	
	.flex-column {
		flex-direction: column;
	}
	
	.flex-center {
		align-items: center;
		justify-content: center;
	}
	
	.flex-between {
		justify-content: space-between;
	}
	
	.flex-1 {
		flex: 1;
	}
	
	/* 文本对齐 */
	.text-center {
		text-align: center;
	}
	
	.text-left {
		text-align: left;
	}
	
	.text-right {
		text-align: right;
	}
	
	/* 字体大小 */
	.text-sm {
		font-size: 24rpx;
	}
	
	.text-base {
		font-size: 28rpx;
	}
	
	.text-lg {
		font-size: 32rpx;
	}
	
	.text-xl {
		font-size: 36rpx;
	}
	
	/* 字体粗细 */
	.font-normal {
		font-weight: 400;
	}
	
	.font-medium {
		font-weight: 500;
	}
	
	.font-semibold {
		font-weight: 600;
	}
	
	.font-bold {
		font-weight: 700;
	}
</style>
