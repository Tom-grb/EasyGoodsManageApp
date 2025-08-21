<template>
	<view class="container">
		<!-- 搜索区域 -->
		<view class="search-section">
			<view class="search-input-wrapper">
				<view class="search-icon">
					<text class="iconfont icon-search"></text>
				</view>
				<input 
					class="search-input" 
					v-model="searchKeyword" 
					placeholder="输入商品名称或条形码搜索..."
					@input="onInput"
					@confirm="onSearch"
					confirm-type="search"
					focus
				/>
				<view v-if="searchKeyword" class="clear-btn" @click="clearSearch">
					<text class="iconfont icon-close"></text>
				</view>
			</view>

		</view>

		<!-- 搜索结果 -->
		<view class="search-results">
			<view v-if="searchKeyword && searchTimer" class="searching">
				<view class="searching-icon">
					<text class="iconfont icon-search"></text>
				</view>
				<text class="searching-text">正在搜索...</text>
			</view>
			
			<view v-else-if="searchKeyword && searchResults.length === 0" class="no-results">
				<view class="no-results-icon">
					<text class="iconfont icon-search"></text>
				</view>
				<text class="no-results-text">未找到相关商品</text>
				<text class="no-results-hint">尝试使用不同的关键词搜索</text>
			</view>
			
			<view v-else-if="!searchKeyword" class="recent-searches">
				<view class="section-header">
					<text class="section-title">最近搜索</text>
					<view class="clear-history" @click="clearHistory">
						<text class="clear-text">清空</text>
					</view>
				</view>
				<view v-if="recentSearches.length === 0" class="empty-history">
					<text class="empty-text">暂无搜索记录</text>
				</view>
				<view v-else class="search-history">
					<view 
						v-for="(item, index) in recentSearches" 
						:key="index"
						class="history-item"
					>
						<view class="history-content" @click="searchByHistory(item)">
							<text class="history-text">{{item.length > 4 ? item.substring(0, 4) + '...' : item}}</text>
						</view>
						<view class="history-delete" @click="deleteHistoryItem(index)">
							x
						</view>
					</view>
				</view>
			</view>
			
			<view v-else class="results-list">
				<view class="section-header">
					<text class="section-title">搜索结果</text>
					<text class="results-count">({{searchResults.length}})</text>
				</view>
				<view class="product-items">
					<view 
						v-for="(product, index) in searchResults" 
						:key="index"
						class="product-item"
						@click="showProductDetail(product)"
					>
						<view class="product-info">
							<text class="product-name">{{product.name}}</text>
							<text class="product-code" v-if="product.barcode">{{product.barcode}}</text>
							<text class="product-code no-barcode" v-else>无条形码</text>
							<text v-if="product.remark" class="product-remark">{{product.remark}}</text>
						</view>
						<view class="product-price">
							<text class="price-symbol">¥</text>
							<text class="price-value">{{product.price}}</text>
						</view>
					</view>
				</view>
			</view>
		</view>

		<!-- 商品详情弹窗 -->
		<view v-if="showDetail" class="modal-overlay" @click="closeDetail">
			<view class="modal-content" @click.stop>
				<view class="modal-header">
					<text class="modal-title">商品详情</text>
					<view class="modal-close" @click="closeDetail">
						<text class="iconfont icon-close"></text>
					</view>
				</view>
				<view class="modal-body">
					<view class="detail-item">
						<text class="detail-label">条形码 <text class="optional-text">(可选)</text></text>
						<view class="input-with-scan">
							<input class="detail-input" v-model="currentProduct.barcode" placeholder="请输入条形码" />
							<view class="scan-btn" @click="scanBarcodeForInput">
								<image class="scan-icon" src="/static/scanIcon.svg" mode="aspectFit"></image>
							</view>
						</view>
					</view>
					<view class="detail-item">
						<text class="detail-label">商品名称 <text class="required-text">*</text></text>
						<input class="detail-input" v-model="currentProduct.name" placeholder="请输入商品名称" />
					</view>
					<view class="detail-item">
						<text class="detail-label">商品价格 <text class="required-text">*</text></text>
						<input class="detail-input" v-model="currentProduct.price" type="number" placeholder="请输入价格" />
					</view>
					<view class="detail-item">
						<text class="detail-label">备注 <text class="optional-text">(可选)</text></text>
						<input class="detail-input" v-model="currentProduct.remark" placeholder="请输入备注信息" />
					</view>
				</view>
				<view class="modal-footer">
					<view class="modal-btn danger" @click="deleteProduct">删除</view>
					<view class="modal-btn primary" @click="saveProduct">保存</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				searchKeyword: '',
				searchResults: [],
				recentSearches: [],
				showDetail: false,
				currentProduct: {},
				// 防抖相关
				searchTimer: null
			}
		},
		onLoad() {
			this.loadRecentSearches()
		},
		onUnload() {
			// 页面销毁时清除定时器
			if (this.searchTimer) {
				clearTimeout(this.searchTimer)
				this.searchTimer = null
			}
		},
		methods: {
			// 加载最近搜索记录
			loadRecentSearches() {
				let recentSearches = uni.getStorageSync('recentSearches') || []
				
				// 清理重复记录（不区分大小写）
				const uniqueSearches = []
				const seen = new Set()
				
				for (const item of recentSearches) {
					const lowerItem = item.toLowerCase()
					if (!seen.has(lowerItem)) {
						seen.add(lowerItem)
						uniqueSearches.push(item)
					}
				}
				
				// 如果清理后有变化，更新存储
				if (uniqueSearches.length !== recentSearches.length) {
					uni.setStorageSync('recentSearches', uniqueSearches)
					console.log('已清理重复搜索记录')
				}
				
				this.recentSearches = uniqueSearches
			},
			
			// 保存搜索历史
			saveSearchHistory(keyword) {
				if (!keyword || !keyword.trim()) return
				
				const trimmedKeyword = keyword.trim()
				let recentSearches = uni.getStorageSync('recentSearches') || []
				
				// 移除已存在的相同关键词（不区分大小写）
				recentSearches = recentSearches.filter(item => 
					item.toLowerCase() !== trimmedKeyword.toLowerCase()
				)
				
				// 添加到开头
				recentSearches.unshift(trimmedKeyword)
				
				// 限制历史记录数量为20个
				if (recentSearches.length > 20) {
					recentSearches = recentSearches.slice(0, 20)
				}
				
				// 保存到本地存储
				uni.setStorageSync('recentSearches', recentSearches)
				
				// 更新当前页面的历史记录
				this.recentSearches = recentSearches
				
				console.log('搜索历史已保存:', trimmedKeyword)
				console.log('当前历史记录:', recentSearches)
			},
			
			// 输入处理（仅搜索，不保存历史）
			onInput() {
				// 清除之前的定时器
				if (this.searchTimer) {
					clearTimeout(this.searchTimer)
				}
				
				// 如果搜索关键词为空，立即清空结果
				if (!this.searchKeyword.trim()) {
					this.searchResults = []
					this.searchTimer = null
					return
				}
				
				// 设置0.2秒防抖
				this.searchTimer = setTimeout(() => {
					this.performSearch(false) // 不保存历史记录
					this.searchTimer = null
				}, 200)
			},
			
			// 搜索功能（点击搜索按钮或回车键）
			onSearch() {
				if (!this.searchKeyword.trim()) {
					uni.showToast({
						title: '请输入搜索关键词',
						icon: 'none'
					})
					return
				}
				
				// 直接执行搜索并保存历史
				this.performSearch(true)
			},
			
			// 执行实际搜索
			performSearch(saveHistory = false) {
				const products = uni.getStorageSync('products') || []
				const keyword = this.searchKeyword.toLowerCase()
				
				console.log('搜索关键词:', keyword)
				console.log('商品总数:', products.length)
				console.log('是否保存历史:', saveHistory)
				
				this.searchResults = products.filter(product => 
					product.name.toLowerCase().includes(keyword) ||
					(product.barcode && product.barcode.includes(keyword)) ||
					(product.remark && product.remark.toLowerCase().includes(keyword))
				)
				
				console.log('搜索结果数量:', this.searchResults.length)
				console.log('搜索结果:', this.searchResults)
				
				// 只有在明确要求时才保存搜索历史
				if (saveHistory) {
					this.saveSearchHistory(keyword)
				}
			},
			
			// 清空搜索
			clearSearch() {
				// 清除定时器
				if (this.searchTimer) {
					clearTimeout(this.searchTimer)
					this.searchTimer = null
				}
				
				this.searchKeyword = ''
				this.searchResults = []
			},
			

			
			// 删除单个历史记录
			deleteHistoryItem(index) {
				if (index >= 0 && index < this.recentSearches.length) {
					const deletedItem = this.recentSearches[index]
					this.recentSearches.splice(index, 1)
					uni.setStorageSync('recentSearches', this.recentSearches)
					
					console.log('已删除搜索记录:', deletedItem)
					
					uni.showToast({
						title: '已删除',
						icon: 'success'
					})
				}
			},
			
			// 清空搜索历史
			clearHistory() {
				uni.showModal({
					title: '确认清空',
					content: '确定要清空搜索历史吗？',
					success: (res) => {
						if (res.confirm) {
							this.recentSearches = []
							uni.setStorageSync('recentSearches', [])
							
							uni.showToast({
								title: '历史记录已清空',
								icon: 'success'
							})
						}
					}
				})
			},
			
			// 点击历史记录搜索
			searchByHistory(keyword) {
				this.searchKeyword = keyword
				// 直接执行搜索并保存历史
				this.performSearch(true)
			},
			
			// 显示商品详情
			showProductDetail(product) {
				this.currentProduct = { ...product }
				this.showDetail = true
				// 更新查询时间
				this.updateQueryTime(product.id)
			},
			
			// 关闭详情弹窗
			closeDetail() {
				this.showDetail = false
				this.currentProduct = {}
			},
			
			// 保存商品
			saveProduct() {
				if (!this.currentProduct.name || !this.currentProduct.price) {
					uni.showToast({
						title: '请填写商品名称和价格',
						icon: 'none'
					})
					return
				}
				
				let products = uni.getStorageSync('products') || []
				const index = products.findIndex(p => p.id === this.currentProduct.id)
				
				if (index !== -1) {
					products[index] = { ...this.currentProduct }
				}
				
				uni.setStorageSync('products', products)
				this.performSearch(false) // 重新搜索以更新结果，但不保存历史
				this.closeDetail()
				
				uni.showToast({
					title: '保存成功',
					icon: 'success'
				})
			},
			
			// 更新查询时间
			updateQueryTime(productId) {
				let products = uni.getStorageSync('products') || []
				const index = products.findIndex(p => p.id === productId)
				if (index !== -1) {
					products[index].queryTime = new Date().getTime()
					uni.setStorageSync('products', products)
				}
			},
			
			// 删除商品
			deleteProduct() {
				// 保存当前商品信息，因为关闭弹窗后会清空
				const productToDelete = { ...this.currentProduct }
				
				// 先关闭详情弹窗，避免层级冲突
				this.closeDetail()
				
				// 延迟执行删除确认，确保弹窗关闭完成
				setTimeout(() => {
					uni.showModal({
						title: '确认删除',
						content: '确定要删除这个商品吗？',
						success: (res) => {
							if (res.confirm) {
								let products = uni.getStorageSync('products') || []
								
								// 确保有ID才进行删除
								if (productToDelete && productToDelete.id) {
									products = products.filter(p => p.id !== productToDelete.id)
									uni.setStorageSync('products', products)
									this.performSearch(false) // 重新搜索以更新结果，但不保存历史
									
									uni.showToast({
										title: '删除成功',
										icon: 'success'
									})
								} else {
									uni.showToast({
										title: '删除失败：商品ID不存在',
										icon: 'none'
									})
								}
							}
						}
					})
				}, 100)
			},
			
			// 从API获取商品信息
			async getProductFromAPI(barcode) {
				try {
					// 获取API配置
					const appId = uni.getStorageSync('appId') || ''
					const appSecret = uni.getStorageSync('appSecret') || ''
					
					// 检查是否配置了API
					if (!appId || !appSecret) {
						console.log('API配置未完成，跳过API查询')
						return { name: '', price: '', remark: '' }
					}
					
					console.log('开始查询API，条形码:', barcode)
					
					// 显示加载提示
					uni.showLoading({
						title: '正在查询商品信息...'
					})
					
					// 调用API
					const res = await uni.request({
						url: 'https://www.mxnzp.com/api/barcode/goods/details',
						method: 'GET',
						data: {
							barcode: barcode,
							app_id: appId,
							app_secret: appSecret
						}
					});
					
					uni.hideLoading()
					
					console.log('API响应:', res)
					
					// 检查响应
					if (res.statusCode === 200 && res.data) {
						const data = res.data
						
						// 检查API返回的数据结构
						if (data.code === 1 && data.data) {
							const productData = data.data
							console.log('API返回商品信息:', productData)
							
							return {
								name: productData.name || productData.goodsName || '',
								price: '',
								remark: ''
							}
						} else {
							console.log('API返回错误:', data.msg || '未知错误')
							return { name: '', price: '', remark: '' }
						}
					} else {
						console.log('API请求失败，状态码:', res.statusCode)
						return { name: '', price: '', remark: '' }
					}
					
				} catch (error) {
					console.error('API调用异常:', error)
					uni.hideLoading()
					return { name: '', price: '', remark: '' }
				}
			},
			
			// 为输入框扫描条形码
			scanBarcodeForInput() {
				// #ifdef APP-PLUS
				uni.scanCode({
					success: async (res) => {
						console.log('扫码结果:', res)
						const barcode = res.result
						
						// 填充到当前商品的条形码输入框
						this.currentProduct.barcode = barcode
						
						// 尝试从API获取商品名称
						const apiProduct = await this.getProductFromAPI(barcode)
						console.log('API返回商品信息:', apiProduct)
						if (apiProduct.name) {
							this.currentProduct.name = apiProduct.name
							uni.showToast({
								title: '已获取商品名称',
								icon: 'success'
							})
						} else {
							uni.showToast({
								title: '条形码已填充',
								icon: 'success'
							})
						}
					},
					fail: (err) => {
						console.log('扫码失败:', err)
						uni.showToast({
							title: '扫码失败',
							icon: 'none'
						})
					}
				})
				// #endif
				
				// #ifndef APP-PLUS
				uni.showToast({
					title: 'APP环境才支持扫码功能',
					icon: 'none'
				})
				// #endif
			}
		}
	}
</script>

<style>
	.container {
		min-height: 100vh;
		background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
		padding: 20rpx;
	}

	/* 搜索区域 */
	.search-section {
		display: flex;
		align-items: center;
		margin-bottom: 30rpx;
	}

	.search-input-wrapper {
		flex: 1;
		height: 80rpx;
		background: #ffffff;
		border-radius: 40rpx;
		display: flex;
		align-items: center;
		padding: 0 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
		margin-right: 20rpx;
	}

	.search-icon {
		margin-right: 20rpx;
		color: #6c757d;
	}

	.search-input {
		flex: 1;
		height: 100%;
		font-size: 28rpx;
		color: #212529;
	}

	.clear-btn {
		width: 40rpx;
		height: 40rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #adb5bd;
	}
	


	.settings-btn {
		width: 80rpx;
		height: 80rpx;
		background: #ffffff;
		border-radius: 40rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
		color: #6c757d;
		margin-right: 10rpx;
	}
	
	.test-btn {
		width: 80rpx;
		height: 80rpx;
		background: #007bff;
		border-radius: 40rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
		color: #ffffff;
		font-size: 20rpx;
	}

	/* 搜索结果区域 */
	.search-results {
		flex: 1;
	}

	.section-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 20rpx;
		padding: 0 10rpx;
	}

	.section-title {
		font-size: 28rpx;
		font-weight: 600;
		color: #495057;
	}

	.results-count {
		font-size: 24rpx;
		color: #6c757d;
	}

	.clear-history {
		padding: 10rpx 20rpx;
	}

	.clear-text {
		font-size: 24rpx;
		color: #007bff;
	}

	/* 无搜索结果 */
	.no-results {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 100rpx 0;
	}

	.no-results-icon {
		font-size: 80rpx;
		color: #dee2e6;
		margin-bottom: 20rpx;
	}

	.no-results-text {
		font-size: 28rpx;
		color: #6c757d;
		margin-bottom: 10rpx;
	}

	.no-results-hint {
		font-size: 24rpx;
		color: #adb5bd;
	}

	/* 搜索中状态 */
	.searching {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 100rpx 0;
	}

	.searching-icon {
		font-size: 80rpx;
		color: #007bff;
		margin-bottom: 20rpx;
		animation: searchPulse 1s infinite;
	}

	.searching-text {
		font-size: 28rpx;
		color: #007bff;
		animation: textPulse 1.5s infinite;
	}

	@keyframes searchPulse {
		0% { 
			opacity: 1; 
			transform: scale(1);
		}
		50% { 
			opacity: 0.7; 
			transform: scale(1.1);
		}
		100% { 
			opacity: 1; 
			transform: scale(1);
		}
	}

	@keyframes textPulse {
		0% { opacity: 1; }
		50% { opacity: 0.6; }
		100% { opacity: 1; }
	}

	/* 最近搜索 */
	.recent-searches {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.empty-history {
		display: flex;
		justify-content: center;
		padding: 60rpx 0;
	}

	.empty-text {
		font-size: 28rpx;
		color: #adb5bd;
	}

	.search-history {
		display: flex;
		flex-wrap: wrap;
		gap: 15rpx;
	}

	.history-item {
		display: flex;
		align-items: center;
		padding: 15rpx 20rpx;
		background: #f8f9fa;
		border-radius: 25rpx;
		border: 1rpx solid #e9ecef;
		max-width: 200rpx;
	}

	.history-content {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.history-text {
		font-size: 24rpx;
		color: #495057;
		text-align: center;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.history-delete {
		width: 40rpx;
		height: 40rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #9b9697;
		border-radius: 50%;
		transition: background-color 0.3s ease;
		margin-left: 10rpx;
	}

	.history-delete:active {
		background-color: rgba(83, 81, 81, 0.1);
	}

	/* 搜索结果列表 */
	.results-list {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.product-items {
		display: flex;
		flex-direction: column;
		gap: 20rpx;
	}

	.product-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 30rpx;
		background: #f8f9fa;
		border-radius: 15rpx;
		border-left: 6rpx solid #007bff;
	}

	.product-info {
		flex: 1;
	}

	.product-name {
		font-size: 30rpx;
		font-weight: 500;
		color: #212529;
		display: block;
		margin-bottom: 8rpx;
	}

	.product-code {
		font-size: 24rpx;
		color: #6c757d;
		display: block;
		margin-bottom: 5rpx;
	}
	
	.product-code.no-barcode {
		color: #adb5bd;
		font-style: italic;
	}

	.product-remark {
		font-size: 22rpx;
		color: #adb5bd;
		display: block;
	}

	.product-price {
		display: flex;
		align-items: baseline;
	}

	.price-symbol {
		font-size: 24rpx;
		color: #dc3545;
		margin-right: 4rpx;
	}

	.price-value {
		font-size: 32rpx;
		font-weight: 600;
		color: #dc3545;
	}

	/* 弹窗样式 */
	.modal-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.5);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
	}

	.modal-content {
		width: 90%;
		max-width: 600rpx;
		background: #ffffff;
		border-radius: 20rpx;
		overflow: hidden;
		box-shadow: 0 20rpx 60rpx rgba(0, 0, 0, 0.3);
	}

	.modal-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 30rpx;
		border-bottom: 1rpx solid #e9ecef;
	}

	.modal-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #212529;
	}

	.modal-close {
		width: 60rpx;
		height: 60rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #6c757d;
	}

	.modal-body {
		padding: 30rpx;
	}

	.detail-item {
		margin-bottom: 30rpx;
	}

	.detail-label {
		font-size: 28rpx;
		color: #495057;
		margin-bottom: 15rpx;
		display: block;
	}
	
	.required-text {
		color: #dc3545;
		font-weight: bold;
	}
	
	.optional-text {
		color: #6c757d;
		font-size: 24rpx;
		font-weight: normal;
	}

	.detail-input {
		height: 80rpx;
		border: 2rpx solid #e9ecef;
		border-radius: 10rpx;
		padding: 0 20rpx;
		font-size: 28rpx;
		background: #f8f9fa;
	}

	.detail-textarea {
		width: 100%;
		height: 120rpx;
		border: 2rpx solid #e9ecef;
		border-radius: 10rpx;
		padding: 20rpx;
		font-size: 28rpx;
		background: #f8f9fa;
	}

	.modal-footer {
		display: flex;
		gap: 20rpx;
		padding: 30rpx;
		border-top: 1rpx solid #e9ecef;
	}

	.modal-btn {
		flex: 1;
		height: 80rpx;
		border-radius: 40rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 28rpx;
		font-weight: 500;
	}

	.modal-btn.primary {
		background: #007bff;
		color: #ffffff;
	}

	.modal-btn.danger {
		background: #dc3545;
		color: #ffffff;
	}

	/* 图标字体 */
	.iconfont {
		font-family: "iconfont";
	}

	.icon-search::before { content: "🔍"; }
	.icon-settings::before { content: "⚙️"; }
	.icon-close::before { content: "✕"; }
	.icon-arrow-right::before { content: "→"; }
	
	/* 带扫描按钮的输入框 */
	.input-with-scan {
		display: flex;
		align-items: center;
		gap: 15rpx;
	}
	
	.input-with-scan .detail-input {
		flex: 1;
	}
	
	.scan-btn {
		width: 60rpx;
		height: 60rpx;
		background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
		border-radius: 30rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		box-shadow: 0 4rpx 12rpx rgba(0, 123, 255, 0.3);
		transition: all 0.3s ease;
	}
	
	.scan-btn:active {
		transform: scale(0.95);
		box-shadow: 0 2rpx 8rpx rgba(0, 123, 255, 0.4);
	}
	
	.scan-icon {
		width: 32rpx;
		height: 32rpx;
		filter: brightness(0) invert(1); /* 将SVG图标转换为白色 */
	}
</style> 