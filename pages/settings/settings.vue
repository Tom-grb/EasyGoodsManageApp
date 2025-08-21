<template>
	<view class="container">
		<!-- 统计信息 -->
		<view class="stats-section">
			<view class="stats-card">
				<view class="stats-icon">
					<text class="iconfont icon-box"></text>
				</view>
				<view class="stats-info">
					<text class="stats-number">{{totalProducts}}</text>
					<text class="stats-label">商品总数</text>
				</view>
			</view>
			<view class="stats-card">
				<view class="stats-icon">
					<text class="iconfont icon-calendar"></text>
				</view>
				<view class="stats-info">
					<text class="stats-number">{{todayAdded}}</text>
					<text class="stats-label">今日新增</text>
				</view>
			</view>
		</view>

		<!-- 数据管理 -->
		<view class="data-section">
			<view class="section-title">数据管理</view>
			<view class="data-actions">
								<view class="action-item" @click="exportData">
					<view class="action-icon">
						<text class="iconfont icon-download"></text>
					</view>
					<view class="action-info">
						<text class="action-title">导出数据</text>
						<text class="action-desc">将商品数据导出为Excel文件</text>
					</view>
					<text class="iconfont icon-arrow-right"></text>
				</view>
				
				<view class="action-item" @click="importData">
					<view class="action-icon">
						<text class="iconfont icon-upload"></text>
					</view>
					<view class="action-info">
						<text class="action-title">导入数据</text>
						<text class="action-desc">从Excel文件导入商品数据</text>
					</view>
					<text class="iconfont icon-arrow-right"></text>
				</view>
				
				<view class="action-item" @click="clearAllData">
					<view class="action-icon danger">
						<text class="iconfont icon-trash"></text>
					</view>
					<view class="action-info">
						<text class="action-title">清空数据</text>
						<text class="action-desc">删除所有商品数据</text>
					</view>
					<text class="iconfont icon-arrow-right"></text>
				</view>
			</view>
		</view>

		<!-- 应用信息 -->
		<view class="app-section">
			<view class="section-title">应用信息</view>
			<view class="app-info">
				<view class="info-item">
					<text class="info-label">应用名称</text>
					<text class="info-value">商品管理</text>
				</view>
				<view class="info-item">
					<text class="info-label">版本号</text>
					<text class="info-value">v1.0.0</text>
				</view>
				<view class="info-item">
					<text class="info-label">数据存储</text>
					<text class="info-value">本地存储</text>
				</view>
			</view>
		</view>


	</view>
</template>

<script>
	export default {
		data() {
			return {
				totalProducts: 0,
				todayAdded: 0
			}
		},
		onLoad() {
			this.loadStats()
		},
		methods: {
			// 加载统计信息
			loadStats() {
				const products = uni.getStorageSync('products') || []
				this.totalProducts = products.length
				
				// 计算今日新增
				const today = new Date()
				today.setHours(0, 0, 0, 0)
				const todayTime = today.getTime()
				
				this.todayAdded = products.filter(product => {
					if (product.createTime) {
						const createDate = new Date(product.createTime)
						createDate.setHours(0, 0, 0, 0)
						return createDate.getTime() === todayTime
					}
					return false
				}).length
			},
			
			// 导出数据
			exportData() {
				const products = uni.getStorageSync('products') || []
				if (products.length === 0) {
					uni.showToast({
						title: '暂无数据可导出',
						icon: 'none'
					})
					return
				}
				
				// #ifdef H5
				// 在H5环境下导出Excel
				this.exportToExcel(products)
				// #endif
				
				// #ifdef APP-PLUS
				// 在APP环境下导出Excel
				this.exportToExcel(products)
				// #endif
			},
			
			// 导出为Excel格式
			exportToExcel(products) {
				// 创建CSV格式的数据（Excel可以打开）
				let csvContent = '\ufeff' // 添加BOM，确保中文正确显示
				
				// 添加表头
				csvContent += 'ID,商品名称,条形码,价格,备注,创建时间,查询时间\n'
				
				// 添加数据行
				products.forEach(product => {
					const id = product.id || ''
					const name = product.name || ''
					const barcode = product.barcode || ''
					const price = product.price || ''
					const remark = (product.remark || '').replace(/"/g, '""') // 转义双引号
					const createTime = product.createTime ? new Date(product.createTime).toLocaleString() : ''
					const queryTime = product.queryTime ? new Date(product.queryTime).toLocaleString() : ''
					
					// 用双引号包围字段，处理包含逗号的文本
					csvContent += `"${id}","${name}","${barcode}","${price}","${remark}","${createTime}","${queryTime}"\n`
				})
				
				const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
				const url = URL.createObjectURL(blob)
				const a = document.createElement('a')
				a.href = url
				a.download = `商品数据_${new Date().getTime()}.csv`
				document.body.appendChild(a)
				a.click()
				document.body.removeChild(a)
				URL.revokeObjectURL(url)
				
				uni.showToast({
					title: '导出成功',
					icon: 'success'
				})
			},
			
			// 导入数据
			importData() {
				console.log('开始导入流程')
				// 显示导入说明
				uni.showModal({
					title: '导入说明',
					content: '1. 支持Excel格式的数据文件(.xlsx, .xls)\n2. 文件应包含：商品名称、条形码、价格、备注\n3. 重复检查：先查条形码，无条形码则查商品名称\n4. 价格必须为数值类型，否则会提示错误行号\n5. 空行将被自动忽略\n6. 建议先导出备份现有数据',
					confirmText: '开始导入',
					cancelText: '取消',
					success: (res) => {
						if (res.confirm) {
							console.log('用户确认导入，开始文件选择')
							this.startFileSelection()
						} else {
							console.log('用户取消导入')
						}
					}
				})
			},
			
			// 开始文件选择
			startFileSelection() {
				console.log('创建文件选择器')
				// 创建隐藏的文件输入元素
				const input = document.createElement('input')
				input.type = 'file'
				input.accept = '.xlsx,.xls'
				input.style.display = 'none'
				
				input.onchange = (event) => {
					const file = event.target.files[0]
					if (file) {
						console.log('用户选择文件:', file.name, '大小:', file.size, '字节')
						this.processImportFile(file)
					} else {
						console.log('用户未选择文件')
					}
					// 清理DOM元素
					document.body.removeChild(input)
				}
				
				document.body.appendChild(input)
				input.click()
			},
			
			// 处理导入文件
			processImportFile(file) {
				console.log('开始处理文件:', file.name)
				
				// 检查文件类型
				if (!file.name.endsWith('.xlsx') && !file.name.endsWith('.xls')) {
					console.log('文件格式不支持:', file.name)
					uni.showToast({
						title: '请选择Excel文件(.xlsx或.xls)',
						icon: 'none'
					})
					return
				}
				
				// 检查文件大小（限制为10MB）
				if (file.size > 10 * 1024 * 1024) {
					console.log('文件过大:', file.size)
					uni.showToast({
						title: '文件过大，请选择小于10MB的文件',
						icon: 'none'
					})
					return
				}
				
				// 显示加载提示
				uni.showLoading({
					title: '正在解析文件...'
				})
				
				// 使用SheetJS库解析Excel文件
				this.parseExcelFile(file)
			},
			
			// 解析Excel文件
			parseExcelFile(file) {
				// 检查是否已加载SheetJS库
				if (typeof XLSX === 'undefined') {
					console.log('SheetJS库未加载，尝试动态加载')
					this.loadSheetJS().then(() => {
						this.parseExcelFile(file)
					}).catch(error => {
						console.error('加载SheetJS失败:', error)
						uni.hideLoading()
						uni.showToast({
							title: 'Excel解析库加载失败',
							icon: 'none'
						})
					})
					return
				}
				
				const reader = new FileReader()
				reader.onload = (e) => {
					try {
						console.log('文件读取完成，开始解析')
						const data = new Uint8Array(e.target.result)
						const workbook = XLSX.read(data, { type: 'array' })
						
						console.log('工作簿信息:', {
							工作表数量: workbook.SheetNames.length,
							工作表名称: workbook.SheetNames
						})
						
						// 获取第一个工作表
						const sheetName = workbook.SheetNames[0]
						const worksheet = workbook.Sheets[sheetName]
						
						console.log('使用工作表:', sheetName)
						
						// 将工作表转换为JSON
						const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 })
						
						console.log('解析到的数据行数:', jsonData.length)
						console.log('前3行数据:', jsonData.slice(0, 3))
						
						// 处理解析结果
						const result = this.processExcelData(jsonData)
						
						uni.hideLoading()
						
						if (result.success) {
							uni.setStorageSync('products', result.products)
							this.loadStats()
							
							console.log('导入成功，数据统计:', {
								总数据: result.products.length,
								新增: result.newCount,
								更新: result.updateCount,
								跳过: result.skipCount
							})
							
							uni.showToast({
								title: `导入成功！新增${result.newCount}条，更新${result.updateCount}条`,
								icon: 'success',
								duration: 3000
							})
						} else {
							console.log('导入失败，错误信息:', result.errors)
							// 显示错误信息
							uni.showModal({
								title: '导入错误',
								content: result.errors.join('\n'),
								showCancel: false,
								confirmText: '确定'
							})
						}
					} catch (error) {
						console.error('Excel解析错误:', error)
						uni.hideLoading()
						uni.showToast({
							title: '文件解析失败',
							icon: 'none'
						})
					}
				}
				
				reader.onerror = (error) => {
					console.error('文件读取错误:', error)
					uni.hideLoading()
					uni.showToast({
						title: '文件读取失败',
						icon: 'none'
					})
				}
				
				reader.readAsArrayBuffer(file)
			},
			
			// 加载SheetJS库
			loadSheetJS() {
				return new Promise((resolve, reject) => {
					if (typeof XLSX !== 'undefined') {
						resolve()
						return
					}
					
					const script = document.createElement('script')
					script.src = 'https://cdn.sheetjs.com/xlsx-0.19.3/package/dist/xlsx.full.min.js'
					script.onload = () => {
						console.log('SheetJS库加载成功')
						resolve()
					}
					script.onerror = () => {
						console.error('SheetJS库加载失败')
						reject(new Error('SheetJS库加载失败'))
					}
					document.head.appendChild(script)
				})
			},
			
			// 处理Excel数据
			processExcelData(jsonData) {
				console.log('开始处理Excel数据')
				
				const products = []
				const existingProducts = uni.getStorageSync('products') || []
				const errors = []
				let newCount = 0
				let updateCount = 0
				let skipCount = 0
				
				// 跳过表头行
				for (let i = 1; i < jsonData.length; i++) {
					const row = jsonData[i]
					console.log(`处理第${i + 1}行:`, row)
					
					// 检查是否为空行
					if (!row || row.length === 0 || row.every(cell => !cell || cell.toString().trim() === '')) {
						console.log(`第${i + 1}行为空行，跳过`)
						skipCount++
						continue
					}
					
					// 确保至少有3个字段
					if (row.length < 3) {
						const error = `第${i + 1}行：数据格式错误，至少需要商品名称、条形码、价格三个字段`
						console.log(error)
						errors.push(error)
						continue
					}
					
					const name = (row[0] || '').toString().trim()
					const barcode = (row[1] || '').toString().trim()
					const price = (row[2] || '').toString().trim()
					const remark = (row[3] || '').toString().trim()
					
					console.log(`第${i + 1}行数据:`, { name, barcode, price, remark })
					
					// 数据校验
					if (!name) {
						const error = `第${i + 1}行：商品名称不能为空`
						console.log(error)
						errors.push(error)
						continue
					}
					
					if (!price) {
						const error = `第${i + 1}行：价格不能为空`
						console.log(error)
						errors.push(error)
						continue
					}
					
					// 价格校验
					if (isNaN(parseFloat(price))) {
						const error = `第${i + 1}行：价格"${price}"不是有效的数值`
						console.log(error)
						errors.push(error)
						continue
					}
					
					// 重复检查
					let existingIndex = -1
					
					// 先检查条形码
					if (barcode) {
						existingIndex = existingProducts.findIndex(p => p.barcode && p.barcode === barcode)
						if (existingIndex !== -1) {
							console.log(`第${i + 1}行：发现重复条形码"${barcode}"，将更新现有商品`)
						}
					} else {
						// 无条形码则检查商品名称
						existingIndex = existingProducts.findIndex(p => p.name && p.name === name)
						if (existingIndex !== -1) {
							console.log(`第${i + 1}行：发现重复商品名称"${name}"，将更新现有商品`)
						}
					}
					
					const product = {
						id: this.generateId(),
						name: name,
						barcode: barcode,
						price: price,
						remark: remark,
						createTime: new Date().getTime(),
						queryTime: new Date().getTime()
					}
					
					if (existingIndex !== -1) {
						// 更新现有商品
						product.id = existingProducts[existingIndex].id
						product.createTime = existingProducts[existingIndex].createTime
						existingProducts[existingIndex] = product
						updateCount++
						console.log(`第${i + 1}行：更新现有商品，ID: ${product.id}`)
					} else {
						// 添加新商品
						products.push(product)
						newCount++
						console.log(`第${i + 1}行：添加新商品，ID: ${product.id}`)
					}
				}
				
				// 合并现有数据和新数据
				const allProducts = [...existingProducts, ...products]
				
				console.log('数据处理完成:', {
					总数据: allProducts.length,
					新增: newCount,
					更新: updateCount,
					跳过: skipCount,
					错误: errors.length
				})
				
				if (errors.length > 0) {
					return {
						success: false,
						errors: errors
					}
				}
				
				return {
					success: true,
					products: allProducts,
					newCount: newCount,
					updateCount: updateCount,
					skipCount: skipCount
				}
			},
			

			

			

			

			

			

			
			// 生成唯一ID
			generateId() {
				return Date.now().toString(36) + Math.random().toString(36).substr(2)
			},
			
			// 清空所有数据
			clearAllData() {
				uni.showModal({
					title: '确认清空',
					content: '此操作将删除所有商品数据，且无法恢复。确定要继续吗？',
					confirmText: '确定清空',
					confirmColor: '#dc3545',
					success: (res) => {
						if (res.confirm) {
							uni.removeStorageSync('products')
							uni.removeStorageSync('recentSearches')
							this.loadStats()
							
							uni.showToast({
								title: '数据已清空',
								icon: 'success'
							})
						}
					}
				})
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

	/* 统计信息 */
	.stats-section {
		display: flex;
		gap: 20rpx;
		margin-bottom: 30rpx;
	}

	.stats-card {
		flex: 1;
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		display: flex;
		align-items: center;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.stats-icon {
		width: 80rpx;
		height: 80rpx;
		border-radius: 40rpx;
		background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
		display: flex;
		align-items: center;
		justify-content: center;
		margin-right: 20rpx;
		color: #ffffff;
		font-size: 36rpx;
	}

	.stats-info {
		flex: 1;
	}

	.stats-number {
		font-size: 36rpx;
		font-weight: 700;
		color: #212529;
		display: block;
		margin-bottom: 5rpx;
	}

	.stats-label {
		font-size: 24rpx;
		color: #6c757d;
	}

	/* 数据管理 */
	.data-section {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		margin-bottom: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.section-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #212529;
		margin-bottom: 30rpx;
	}

	.data-actions {
		display: flex;
		flex-direction: column;
		gap: 20rpx;
	}

	.action-item {
		display: flex;
		align-items: center;
		padding: 30rpx;
		background: #f8f9fa;
		border-radius: 15rpx;
		border-left: 6rpx solid #007bff;
	}

	.action-icon {
		width: 60rpx;
		height: 60rpx;
		border-radius: 30rpx;
		background: #007bff;
		display: flex;
		align-items: center;
		justify-content: center;
		margin-right: 20rpx;
		color: #ffffff;
		font-size: 28rpx;
	}

	.action-icon.danger {
		background: #dc3545;
	}

	.action-info {
		flex: 1;
	}

	.action-title {
		font-size: 30rpx;
		font-weight: 500;
		color: #212529;
		display: block;
		margin-bottom: 5rpx;
	}

	.action-desc {
		font-size: 24rpx;
		color: #6c757d;
	}

	/* 应用信息 */
	.app-section {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.app-info {
		display: flex;
		flex-direction: column;
		gap: 20rpx;
	}

	.info-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 20rpx 0;
		border-bottom: 1rpx solid #e9ecef;
	}

	.info-item:last-child {
		border-bottom: none;
	}

	.info-label {
		font-size: 28rpx;
		color: #495057;
	}

	.info-value {
		font-size: 28rpx;
		color: #212529;
		font-weight: 500;
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

	.import-tips {
		margin-bottom: 30rpx;
	}

	.tips-title {
		font-size: 28rpx;
		font-weight: 600;
		color: #495057;
		display: block;
		margin-bottom: 15rpx;
	}

	.tips-text {
		font-size: 24rpx;
		color: #6c757d;
		display: block;
		margin-bottom: 8rpx;
		line-height: 1.5;
	}

	.file-input {
		border: 2rpx dashed #dee2e6;
		border-radius: 10rpx;
		padding: 40rpx;
		text-align: center;
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

	/* 图标字体 */
	.iconfont {
		font-family: "iconfont";
	}

	.icon-box::before { content: "📦"; }
	.icon-calendar::before { content: "📅"; }
	.icon-download::before { content: "⬇️"; }
	.icon-upload::before { content: "⬆️"; }
	.icon-trash::before { content: "🗑️"; }
	.icon-close::before { content: "✕"; }
	.icon-arrow-right::before { content: "→"; }
</style> 