<template>
	<view class="container">
		<!-- 顶部搜索区域 -->
		<view class="search-section">
			<view class="search-bar" @click="goToSearch">
				<view class="search-icon">
					<text class="iconfont icon-search"></text>
				</view>
				<text class="search-placeholder">搜索商品...</text>
			</view>
			<view class="settings-btn" @click="goToSettings">
				<text class="iconfont icon-settings"></text>
			</view>
		</view>

		<!-- 商品列表 -->
		<view class="product-list">
			<view class="list-header">
				<text class="list-title">最近商品</text>
			</view>
			
			<view v-if="recentProducts.length === 0" class="empty-state">
				<view class="empty-icon">
					<text class="iconfont icon-box"></text>
				</view>
				<text class="empty-text">暂无商品数据</text>
				<text class="empty-hint">点击下方按钮添加商品</text>
			</view>
			
			<view v-else class="product-items">
				<view 
					v-for="product in recentProducts" 
					:key="product.id"
					class="product-item"
					:class="{ 'selected': selectedProducts.includes(product.id) }"
					@click="handleProductClick(product)"
					@longpress="handleProductLongPress(product)"
				>
					<view class="product-info">
						<text class="product-name">{{product.name}}</text>
						<text class="product-code" v-if="product.barcode">{{product.barcode}}</text>
						<text class="product-code no-barcode" v-else>无条形码</text>
					</view>
					<view class="product-price">
						<text class="price-symbol">¥</text>
						<text class="price-value">{{product.price}}</text>
					</view>
				</view>
				
				<!-- 加载更多按钮 -->
				<view v-if="hasMore" class="load-more-btn" @click="loadMore">
					<text class="load-more-text">加载更多</text>
				</view>
				
				<!-- 没有更多数据提示 -->
				<view v-else-if="recentProducts.length > 0" class="no-more-hint">
					<text class="no-more-text">没有更多数据了</text>
				</view>
			</view>
		</view>

		<!-- 底部操作按钮 -->
		<view class="bottom-actions" v-if="!isBatchMode">
			<view class="action-btn primary" @click="showAddProduct">
				<text class="iconfont icon-plus"></text>
				<text class="btn-text">新增商品</text>
			</view>
			<view class="action-btn secondary" @click="scanBarcode">
				<text class="iconfont icon-scan"></text>
				<text class="btn-text">扫码录入</text>
			</view>
		</view>
		
		<!-- 批量删除操作按钮 -->
		<view class="bottom-actions" v-if="isBatchMode">
			<view class="action-btn secondary" @click="cancelBatchMode">
				<text class="iconfont icon-close"></text>
				<text class="btn-text">取消</text>
			</view>
			<view class="action-btn danger" @click="batchDelete" :class="{ 'disabled': selectedProducts.length === 0 }">
				<text class="iconfont icon-delete"></text>
				<text class="btn-text">删除({{selectedProducts.length}})</text>
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
							<view class="scan-btn" @click="scanBarcodeForInput('current')">
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

		<!-- 新增商品弹窗 -->
		<view v-if="showAdd" class="modal-overlay" @click="closeAdd">
			<view class="modal-content" @click.stop>
				<view class="modal-header">
					<text class="modal-title">新增商品</text>
					<view class="modal-close" @click="closeAdd">
						<text class="iconfont icon-close"></text>
					</view>
				</view>
				<view class="modal-body">
					<view class="detail-item">
						<text class="detail-label">条形码 <text class="optional-text">(可选)</text></text>
						<view class="input-with-scan">
							<input class="detail-input" v-model="newProduct.barcode" placeholder="请输入条形码" />
							<view class="scan-btn" @click="scanBarcodeForInput('new')">
								<image class="scan-icon" src="/static/scanIcon.svg" mode="aspectFit"></image>
							</view>
						</view>
					</view>
					<view class="detail-item">
						<text class="detail-label">商品名称 <text class="required-text">*</text></text>
						<input class="detail-input" v-model="newProduct.name" placeholder="请输入商品名称" />
					</view>
					<view class="detail-item">
						<text class="detail-label">商品价格 <text class="required-text">*</text></text>
						<input class="detail-input" v-model="newProduct.price" type="number" placeholder="请输入价格" />
					</view>
					<view class="detail-item">
						<text class="detail-label">备注 <text class="optional-text">(可选)</text></text>
						<input class="detail-input" v-model="newProduct.remark" placeholder="请输入备注信息" />
					</view>
				</view>
				<view class="modal-footer">
					<view class="modal-btn secondary" @click="closeAdd">取消</view>
					<view class="modal-btn primary" @click="addProduct">确定</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				recentProducts: [],
				showDetail: false,
				showAdd: false,
				currentProduct: {},
				newProduct: {
					name: '',
					barcode: '',
					price: '',
					remark: '',
					queryTime: new Date().getTime()
				},
				// 批量删除相关
				isBatchMode: false,
				selectedProducts: [],
				// 分页相关
				pageSize: 10,
				currentPage: 1,
				hasMore: true,
				allProducts: []
			}
		},
		onLoad() {
			this.loadProducts()
		},
		onShow() {
			// 每次显示页面时重新加载数据
			this.refreshData()
		},
		methods: {
			// 生成唯一ID
			generateId() {
				return Date.now().toString(36) + Math.random().toString(36).substr(2)
			},
			
			// 加载商品数据
			loadProducts() {
				let products = uni.getStorageSync('products') || []
				
				// 数据迁移：为没有ID的商品添加ID
				let hasChanges = false
				products = products.map(product => {
					if (!product.id) {
						hasChanges = true
						return {
							...product,
							id: this.generateId()
						}
					}
					return product
				})
				
				if (hasChanges) {
					uni.setStorageSync('products', products)
				}
				
				// 按查询时间排序，最早的在前面
				const sortedProducts = products.sort((a, b) => {
					const timeA = a.queryTime || a.createTime || 0
					const timeB = b.queryTime || b.createTime || 0
					return timeA - timeB
				})
				
				this.allProducts = sortedProducts
				this.loadMoreProducts()
			},
			
			// 刷新数据
			refreshData() {
				this.currentPage = 1
				this.hasMore = true
				this.recentProducts = []
				console.log('refreshData')
				this.loadProducts()
			},
			
			// 加载更多商品
			loadMoreProducts() {
				const startIndex = (this.currentPage - 1) * this.pageSize
				const endIndex = startIndex + this.pageSize
				const newProducts = this.allProducts.slice(startIndex, endIndex)
				
				if (this.currentPage === 1) {
					this.recentProducts = newProducts
				} else {
					this.recentProducts = [...this.recentProducts, ...newProducts]
				}
				
				this.hasMore = endIndex < this.allProducts.length
			},
			
			// 下拉刷新
			onPullDownRefresh() {
				this.refreshData()
				setTimeout(() => {
					uni.stopPullDownRefresh()
				}, 500)
			},
			
			// 加载更多按钮点击事件
			loadMore() {
				if (this.hasMore && !this.isBatchMode) {
					this.currentPage++
					this.loadMoreProducts()
				}
			},
			
			// 跳转到搜索页面
			goToSearch() {
				uni.navigateTo({
					url: '/pages/search/search'
				})
			},
			
			// 跳转到设置页面
			goToSettings() {
				uni.navigateTo({
					url: '/pages/settings/settings'
				})
			},
			
			// 处理商品点击
			handleProductClick(product) {
				if (this.isBatchMode) {
					// 批量模式下，点击选择/取消选择
					this.toggleProductSelection(product.id)
				} else {
					// 正常模式下，显示详情
					this.showProductDetail(product)
				}
			},
			
			// 处理商品长按
			handleProductLongPress(product) {
				if (!this.isBatchMode) {
					this.enterBatchMode()
					this.toggleProductSelection(product.id)
				}
			},
			
			// 进入批量模式
			enterBatchMode() {
				this.isBatchMode = true
				this.selectedProducts = []
			},
			
			// 取消批量模式
			cancelBatchMode() {
				this.isBatchMode = false
				this.selectedProducts = []
			},
			
			// 切换商品选择状态
			toggleProductSelection(productId) {
				const index = this.selectedProducts.indexOf(productId)
				if (index > -1) {
					this.selectedProducts.splice(index, 1)
				} else {
					this.selectedProducts.push(productId)
				}
			},
			
			// 批量删除
			batchDelete() {
				if (this.selectedProducts.length === 0) {
					uni.showToast({
						title: '请选择要删除的商品',
						icon: 'none'
					})
					return
				}
				
				// 保存选中的商品ID列表
				const selectedIds = [...this.selectedProducts]
				
				uni.showModal({
					title: '确认删除',
					content: `确定要删除选中的 ${selectedIds.length} 个商品吗？`,
					success: (res) => {
						if (res.confirm) {
							let products = uni.getStorageSync('products') || []
							products = products.filter(p => !selectedIds.includes(p.id))
							uni.setStorageSync('products', products)
							this.loadProducts()
							this.cancelBatchMode()
							
							uni.showToast({
								title: '删除成功',
								icon: 'success'
							})
						}
					}
				})
			},
			
			// 显示商品详情
			showProductDetail(product) {
				this.currentProduct = { ...product }
				this.showDetail = true
				// 更新查询时间
				this.updateQueryTime(product.id)
				
				// 调试信息：检查商品是否有ID
				console.log('显示商品详情:', this.currentProduct)
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
				} else {
					// 如果没有找到匹配的ID，添加为新商品
					products.unshift({
						...this.currentProduct,
						id: this.generateId(),
						createTime: new Date().getTime(),
						queryTime: new Date().getTime()
					})
				}
				
				uni.setStorageSync('products', products)
				this.loadProducts()
				this.closeDetail()
				
				uni.showToast({
					title: '保存成功',
					icon: 'success'
				})
			},
			
			// 删除商品
			deleteProduct() {
				// 保存当前商品信息，因为关闭弹窗后会清空
				const productToDelete = { ...this.currentProduct }
				
				console.log('准备删除商品:', productToDelete)
				
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
								
								console.log('删除前商品数量:', products.length)
								console.log('要删除的商品ID:', productToDelete.id)
								
								// 确保有ID才进行删除
								if (productToDelete && productToDelete.id) {
									products = products.filter(p => p.id !== productToDelete.id)
									console.log('删除后商品数量:', products.length)
									
									uni.setStorageSync('products', products)
									this.loadProducts()
									
									uni.showToast({
										title: '删除成功',
										icon: 'success'
									})
								} else {
									console.error('商品ID不存在:', productToDelete)
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
			
			// 显示新增商品弹窗
			showAddProduct() {
				// 如果条形码已经设置（来自扫码），则保留；否则清空
				const currentBarcode = this.newProduct.barcode || ''
				this.newProduct = {
					name: '',
					barcode: currentBarcode,
					price: '',
					remark: '',
					queryTime: new Date().getTime()
				}
				this.showAdd = true
			},
			
			// 关闭新增弹窗
			closeAdd() {
				this.showAdd = false
			},
			
			// 添加商品
			addProduct() {
				if (!this.newProduct.name || !this.newProduct.price) {
					uni.showToast({
						title: '请填写商品名称和价格',
						icon: 'none'
					})
					return
				}
				
				let products = uni.getStorageSync('products') || []
				// 只有当条形码存在时才检查重复
				if (this.newProduct.barcode) {
					const exists = products.find(p => p.barcode === this.newProduct.barcode)
					if (exists) {
						uni.showToast({
							title: '条形码已存在',
							icon: 'none'
						})
						return
					}
				}
				
				products.unshift({
					...this.newProduct,
					id: this.generateId(),
					createTime: new Date().getTime(),
					queryTime: this.newProduct.queryTime
				})
				
				uni.setStorageSync('products', products)
				this.loadProducts()
				this.closeAdd()
				
				// 清除新增商品的所有信息，包括条形码
				this.newProduct = {
					name: '',
					barcode: '',
					price: '',
					remark: '',
					queryTime: new Date().getTime()
				}
				
				uni.showToast({
					title: '添加成功',
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
			
			// 扫码录入
			scanBarcode() {
				// #ifdef APP-PLUS
				uni.scanCode({
					success: async (res) => {
						const barcode = res.result
						console.log('扫码结果:', barcode)
						
						let products = uni.getStorageSync('products') || []
						const product = products.find(p => p.barcode === barcode)
						
						if (product) {
							console.log('找到商品:', product)
							this.showProductDetail(product)
						} else {
							console.log('未找到商品，尝试从API获取信息，条形码:', barcode)
							
							// 尝试从API获取商品信息
							const apiProduct = await this.getProductFromAPI(barcode)
							
							// 设置新增商品信息
							this.newProduct = {
								name: apiProduct.name || '',
								barcode: barcode,
								price: apiProduct.price || '',
								remark: apiProduct.remark || '',
								queryTime: new Date().getTime()
							}
							this.showAdd = true
							
							// 提示用户
							if (apiProduct.name) {
								uni.showToast({
									title: '已获取商品信息',
									icon: 'success',
									duration: 2000
								})
							} else {
								uni.showToast({
									title: '未找到商品，请手动添加',
									icon: 'none',
									duration: 2000
								})
							}
						}
					},
					fail: (err) => {
						console.error('扫码失败:', err)
						uni.showToast({
							title: '扫码失败',
							icon: 'none'
						})
					}
				})
				// #endif
				
				// #ifdef H5
				uni.showToast({
					title: 'H5环境不支持扫码',
					icon: 'none'
				})
				// #endif
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
			scanBarcodeForInput(type) {
				// #ifdef APP-PLUS
				uni.scanCode({
					success: async (res) => {
						console.log('扫码结果:', res)
						const barcode = res.result
						
						// 根据类型填充到对应的输入框
						if (type === 'current') {
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
						} else if (type === 'new') {
							this.newProduct.barcode = barcode
							
							// 尝试从API获取商品名称
							const apiProduct = await this.getProductFromAPI(barcode)
							console.log('API返回商品信息:', apiProduct)
							if (apiProduct.name) {
								this.newProduct.name = apiProduct.name
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
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
		padding: 20rpx;
		display: flex;
		flex-direction: column;
		overflow: hidden;
		box-sizing: border-box;
		z-index: 1;
	}

	/* 搜索区域 */
	.search-section {
		display: flex;
		align-items: center;
		margin-bottom: 30rpx;
		flex-shrink: 0;
	}

	.search-bar {
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

	.search-placeholder {
		color: #adb5bd;
		font-size: 28rpx;
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
	}

	/* 商品列表 */
	.product-list {
		flex: 1;
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		min-height: 0;
	}

	.list-header {
		display: flex;
		align-items: center;
		margin-bottom: 30rpx;
		flex-shrink: 0;
	}

	.list-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #212529;
	}

	.list-count {
		font-size: 24rpx;
		color: #6c757d;
		margin-left: 10rpx;
	}

	.empty-state {
		flex: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: 80rpx 0;
		min-height: 0;
	}

	.empty-icon {
		font-size: 80rpx;
		color: #dee2e6;
		margin-bottom: 20rpx;
	}

	.empty-text {
		font-size: 28rpx;
		color: #6c757d;
		margin-bottom: 10rpx;
	}

	.empty-hint {
		font-size: 24rpx;
		color: #adb5bd;
	}

	.product-items {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 20rpx;
		overflow-y: auto;
		-webkit-overflow-scrolling: touch;
		min-height: 0;
	}

	/* 加载更多按钮 */
	.load-more-btn {
		display: flex;
		justify-content: center;
		align-items: center;
		padding: 30rpx;
		background: #f8f9fa;
		border-radius: 15rpx;
		border: 2rpx solid #e9ecef;
		margin-top: 20rpx;
		transition: all 0.3s ease;
	}

	.load-more-btn:active {
		background: #e9ecef;
		transform: scale(0.98);
	}

	.load-more-text {
		font-size: 28rpx;
		color: #007bff;
		font-weight: 500;
	}

	/* 没有更多数据提示 */
	.no-more-hint {
		display: flex;
		justify-content: center;
		align-items: center;
		padding: 30rpx;
		margin-top: 20rpx;
	}

	.no-more-text {
		font-size: 24rpx;
		color: #adb5bd;
	}

	.product-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 30rpx;
		background: #f8f9fa;
		border-radius: 15rpx;
		border-left: 6rpx solid #007bff;
		transition: all 0.3s ease;
	}
	
	.product-item.selected {
		background: #e3f2fd;
		border-left-color: #2196f3;
		transform: scale(0.98);
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
	}
	
	.product-code.no-barcode {
		color: #adb5bd;
		font-style: italic;
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

	/* 底部操作按钮 */
	.bottom-actions {
		display: flex;
		gap: 20rpx;
		margin-top: auto;
		flex-shrink: 0;
	}

	.action-btn {
		flex: 1;
		height: 100rpx;
		border-radius: 50rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 28rpx;
		font-weight: 500;
		box-shadow: 0 6rpx 25rpx rgba(0, 0, 0, 0.15);
	}

	.action-btn.primary {
		background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
		color: #ffffff;
	}

	.action-btn.secondary {
		background: linear-gradient(135deg, #6c757d 0%, #495057 100%);
		color: #ffffff;
	}
	
	.action-btn.danger {
		background: linear-gradient(135deg, #dc3545 0%, #c82333 100%);
		color: #ffffff;
	}
	
	.action-btn.disabled {
		background: linear-gradient(135deg, #adb5bd 0%, #6c757d 100%);
		color: #ffffff;
		opacity: 0.6;
	}

	.btn-text {
		margin-left: 10rpx;
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
	
	/* 确保uni.showModal在最上层 */
	.uni-modal {
		z-index: 9999 !important;
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

	.detail-input {
		height: 80rpx;
		border: 2rpx solid #e9ecef;
		border-radius: 10rpx;
		padding: 0 20rpx;
		font-size: 28rpx;
		background: #f8f9fa;
	}

	.detail-textarea {
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

	.modal-btn.secondary {
		background: #6c757d;
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
	.icon-plus::before { content: "➕"; }
	.icon-scan::before { content: "📷"; }
	.icon-close::before { content: "✕"; }
	.icon-box::before { content: "📦"; }
	.icon-delete::before { content: "🗑️"; }
</style>
