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

		<!-- API配置 -->
		<view class="api-section">
			<view class="section-title">API配置</view>
			<view class="api-config">
				<view class="config-item">
					<text class="config-label">AppID</text>
					<input 
						class="config-input" 
						v-model="appId" 
						placeholder="请输入AppID"
						@blur="saveApiConfig"
					/>
				</view>
				<view class="config-item">
					<text class="config-label">AppSecret</text>
					<input 
						class="config-input" 
						v-model="appSecret" 
						placeholder="请输入AppSecret"
						password
						@blur="saveApiConfig"
					/>
				</view>
				<view class="config-tips">
					<text class="tips-text">用于扫码获取商品名称，配置后扫码时将自动查询在线数据库获取商品名称</text>
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
				todayAdded: 0,
				appId: '',
				appSecret: ''
			}
		},
		onLoad() {
			this.loadStats()
			this.loadApiConfig()
		},
		methods: {
			// 加载API配置
			loadApiConfig() {
				this.appId = uni.getStorageSync('appId') || ''
				this.appSecret = uni.getStorageSync('appSecret') || ''
			},
			
			// 保存API配置
			saveApiConfig() {
				// 二次确认
				uni.showModal({
					title: '确认修改',
					content: '确定要修改API配置吗？',
					confirmText: '确定',
					cancelText: '取消',
					success: (res) => {
						if (res.confirm) {
							uni.setStorageSync('appId', this.appId)
							uni.setStorageSync('appSecret', this.appSecret)
							uni.showToast({
								title: '配置已保存',
								icon: 'success'
							})
						} else {
							// 取消时恢复原值
							this.loadApiConfig()
						}
					}
				})
			},
			
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
				this.exportToExcel(products)
				// #endif
				
				// #ifdef APP-PLUS
				this.exportToAppFile(products)
				// #endif
				
				// #ifdef MP-WEIXIN
				this.exportToClipboard(products)
				// #endif
			},
			
			// H5环境导出
			exportToExcel(products) {
				// 创建CSV格式的数据
				let csvContent = '\ufeff' // 添加BOM，确保中文正确显示
				
				// 添加表头
				csvContent += '条形码,商品名称,商品价格,备注\n'
				
				// 添加数据行
				products.forEach(product => {
					const barcode = product.barcode || ''
					const name = product.name || ''
					const price = product.price || ''
					const remark = (product.remark || '').replace(/"/g, '""') // 转义双引号
					
					// 处理条形码，确保长数字不被缩写
					const formattedBarcode = barcode ? `="${barcode}"` : ''
					
					// 用双引号包围字段，处理包含逗号的文本
					csvContent += `${formattedBarcode},"${name}","${price}","${remark}"\n`
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
			
			// APP环境导出
			exportToAppFile(products) {
				console.log('开始APP环境导出，数据条数:', products.length)
				
				// 显示加载提示
				uni.showLoading({
					title: '正在导出...'
				})
				
				// 创建CSV格式的数据
				let csvContent = '\ufeff' // 添加BOM，确保中文正确显示
				
				// 添加表头
				csvContent += '条形码,商品名称,商品价格,备注\n'
				
				// 添加数据行
				products.forEach(product => {
					const barcode = product.barcode || ''
					const name = product.name || ''
					const price = product.price || ''
					const remark = (product.remark || '').replace(/"/g, '""') // 转义双引号
					
					// 处理条形码，确保长数字不被缩写
					const formattedBarcode = barcode ? `="${barcode}"` : ''
					
					// 用双引号包围字段，处理包含逗号的文本
					csvContent += `${formattedBarcode},"${name}","${price}","${remark}"\n`
				})
				
				const fileName = `商品数据_${new Date().getTime()}.csv`
				
				// 写入文件
				plus.io.requestFileSystem(plus.io.PRIVATE_DOC, (fs) => {
					fs.root.getFile(fileName, { create: true }, (fileEntry) => {
						fileEntry.createWriter((writer) => {
							writer.write(csvContent)
							writer.onwrite = () => {
								console.log('文件写入成功')
								uni.hideLoading()
								
								// 获取文件的绝对路径
								try {
									// 方法1：使用plus.io.convertLocalFileSystemURL
									const relativePath = `_doc/${fileName}`
									const absolutePath = plus.io.convertLocalFileSystemURL(relativePath)
									console.log('文件绝对路径:', absolutePath)
									this.showExportSuccess(absolutePath)
								} catch (error) {
									console.error('获取路径失败:', error)
									try {
										// 方法2：使用toLocalURL
										if (typeof fileEntry.toLocalURL === 'function') {
											fileEntry.toLocalURL((absolutePath) => {
												console.log('文件绝对路径:', absolutePath)
												this.showExportSuccess(absolutePath)
											})
										} else {
											// 方法3：使用fullPath
											const absolutePath = fileEntry.fullPath || fileEntry.nativeURL
											console.log('文件绝对路径:', absolutePath)
											this.showExportSuccess(absolutePath)
										}
									} catch (error2) {
										console.error('获取路径失败:', error2)
										// 方法4：使用默认路径
										const defaultPath = `应用文档目录/${fileName}`
										this.showExportSuccess(defaultPath)
									}
								}
							}
							writer.onerror = (e) => {
								console.error('文件写入失败:', e)
								uni.hideLoading()
								uni.showToast({
									title: '导出失败',
									icon: 'none'
								})
							}
						})
					})
				})
			},
			
			// 微信小程序环境导出
			exportToClipboard(products) {
				// 创建CSV格式的数据
				let csvContent = '条形码,商品名称,商品价格,备注\n'
				
				// 添加数据行
				products.forEach(product => {
					const barcode = product.barcode || ''
					const name = product.name || ''
					const price = product.price || ''
					const remark = product.remark || ''
					
					// 处理条形码，确保长数字不被缩写
					const formattedBarcode = barcode ? `="${barcode}"` : ''
					
					csvContent += `${formattedBarcode},${name},${price},${remark}\n`
				})
				
				// 复制到剪贴板
				uni.setClipboardData({
					data: csvContent,
					success: () => {
						uni.showModal({
							title: '导出成功',
							content: '数据已复制到剪贴板，请粘贴到Excel中保存',
							showCancel: false
						})
					}
				})
			},
			
			// 导入数据
			importData() {
				console.log('开始导入流程')
				// 显示导入说明
				uni.showModal({
					title: '导入说明',
					content: '1. 支持CSV格式的数据文件\n2. 文件应包含：条形码（选填）、商品名称（必填）、商品价格（必填，数值）、备注（选填）\n3. 重复检查：先查条形码，无条形码重复，则查商品名称，无重复才插入，否则替换\n4. 商品名称和价格不能为空，价格必须为有效数值\n5. 空行将被自动忽略\n6. 建议先导出备份现有数据\n7.从应用文档目录选择文件',
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
				
				// #ifdef H5
				// H5环境使用DOM API
				if (typeof document !== 'undefined') {
					const input = document.createElement('input')
					input.type = 'file'
					input.accept = '.csv'
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
				} else {
					uni.showToast({
						title: 'H5环境不支持文件选择',
						icon: 'none'
					})
				}
				// #endif
				
				// #ifdef APP-PLUS
				// APP环境直接使用应用文档目录选择
				this.scanAppDirectory()
				// #endif
				
				// #ifdef MP-WEIXIN
				// 微信小程序环境
				uni.showToast({
					title: '微信小程序不支持文件导入',
					icon: 'none'
				})
				// #endif
			},
			

			
			// 扫描APP目录
			scanAppDirectory() {
				// #ifdef APP-PLUS
				plus.io.requestFileSystem(plus.io.PRIVATE_DOC, (fs) => {
					fs.root.createReader().readEntries((entries) => {
						const csvFiles = entries.filter(entry => 
							entry.isFile && entry.name.toLowerCase().endsWith('.csv')
						)
						
						if (csvFiles.length === 0) {
							uni.showModal({
								title: '未找到CSV文件',
								content: '应用文档目录下没有找到CSV文件，请先放入CSV文件',
								confirmText: '确定',
								showCancel: false
							})
							return
						}
						
						// 显示文件选择列表
						const fileNames = csvFiles.map(file => file.name)
						uni.showActionSheet({
							itemList: fileNames,
							success: (res) => {
								const selectedFile = csvFiles[res.tapIndex]
								this.processAppFile(selectedFile)
							}
						})
					})
				})
				// #endif
			},
			
			// 处理APP文件
			processAppFile(fileEntry) {
				// #ifdef APP-PLUS
				fileEntry.file((file) => {
					const reader = new plus.io.FileReader()
					reader.onload = (e) => {
						const content = e.target.result
						this.parseCSVContent(content)
					}
					reader.onerror = (e) => {
						console.error('文件读取失败:', e)
						uni.showToast({
							title: '文件读取失败',
							icon: 'none'
						})
					}
					reader.readAsText(file, 'utf-8')
				})
				// #endif
			},
			
			// 处理导入文件
			processImportFile(file) {
				console.log('开始处理文件:', file.name)
				
				// 检查文件类型
				if (!file.name.endsWith('.csv')) {
					console.log('文件格式不支持:', file.name)
					uni.showToast({
						title: '请选择CSV文件(.csv)',
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
				
				// 读取CSV文件
				const reader = new FileReader()
				reader.onload = (e) => {
					const content = e.target.result
					uni.hideLoading()
					this.parseCSVContent(content)
				}
				reader.onerror = (e) => {
					console.error('文件读取失败:', e)
					uni.hideLoading()
					uni.showToast({
						title: '文件读取失败',
						icon: 'none'
					})
				}
				reader.readAsText(file, 'utf-8')
			},
			

			

			


			

			

			

			

			

			

			
			// 解析CSV内容
			parseCSVContent(content) {
				console.log('开始解析CSV内容')
				
				try {
					// 移除BOM
					if (content.charCodeAt(0) === 0xFEFF) {
						content = content.substring(1)
					}
					
					// 按行分割
					const lines = content.split('\n').filter(line => line.trim() !== '')
					console.log('CSV行数:', lines.length)
					
					if (lines.length < 2) {
						uni.showToast({
							title: '文件格式错误，至少需要表头和数据行',
							icon: 'none'
						})
						return
					}
					
					// 解析表头
					const header = this.parseCSVLine(lines[0])
					console.log('表头:', header)
					
					// 验证表头格式
					const expectedHeaders = ['条形码', '商品名称', '商品价格', '备注']
					const headerMatch = expectedHeaders.every((expected, index) => 
						header[index] && header[index].trim() === expected
					)
					
					if (!headerMatch) {
						uni.showModal({
							title: '表头格式错误',
							content: `期望的表头格式：${expectedHeaders.join(', ')}\n\n字段说明：\n- 条形码：选填\n- 商品名称：必填\n- 商品价格：必填，数值\n- 备注：选填\n\n实际表头：${header.join(', ')}`,
							showCancel: false
						})
						return
					}
					
					// 处理数据行
					const result = this.processCSVData(lines.slice(1))
					
					if (result.success) {
						console.log('导入成功:', {
							新增: result.newCount,
							更新: result.updateCount,
							跳过: result.skipCount
						})
						
						// 显示导入结果弹窗
						uni.showModal({
							title: '导入成功',
							content: `导入完成！\n\n新增：${result.newCount}条\n更新：${result.updateCount}条\n跳过：${result.skipCount}条\n\n总计处理：${result.newCount + result.updateCount + result.skipCount}条数据`,
							confirmText: '确定',
							showCancel: false
						})
						
						// 刷新统计信息
						this.loadStats()
					} else {
						console.log('导入失败，错误信息:', result.errors)
						uni.showModal({
							title: '导入错误',
							content: result.errors.join('\n'),
							showCancel: false,
							confirmText: '确定'
						})
					}
					
				} catch (error) {
					console.error('CSV解析错误:', error)
					uni.showToast({
						title: '文件解析失败',
						icon: 'none'
					})
				}
			},
			
			// 解析CSV行
			parseCSVLine(line) {
				const result = []
				let current = ''
				let inQuotes = false
				
				for (let i = 0; i < line.length; i++) {
					const char = line[i]
					
					if (char === '"') {
						if (inQuotes && line[i + 1] === '"') {
							// 转义的双引号
							current += '"'
							i++
						} else {
							// 开始或结束引号
							inQuotes = !inQuotes
						}
					} else if (char === ',' && !inQuotes) {
						// 字段分隔符
						result.push(current.trim())
						current = ''
					} else {
						current += char
					}
				}
				
				// 添加最后一个字段
				result.push(current.trim())
				return result
			},
			
			// 处理CSV数据
			processCSVData(dataLines) {
				console.log('开始处理CSV数据')
				
				const products = []
				const existingProducts = uni.getStorageSync('products') || []
				const errors = []
				let newCount = 0
				let updateCount = 0
				let skipCount = 0
				
				// 处理每一行数据
				for (let i = 0; i < dataLines.length; i++) {
					const line = dataLines[i]
					const rowNumber = i + 2 // 第2行开始是数据
					console.log(`处理第${rowNumber}行:`, line)
					
					// 检查是否为空行
					if (!line || line.trim() === '') {
						console.log(`第${rowNumber}行为空行，跳过`)
						skipCount++
						continue
					}
					
					// 解析行数据
					const fields = this.parseCSVLine(line)
					console.log(`第${rowNumber}行字段:`, fields)
					
					// 确保至少有2个字段（商品名称和价格是必填的）
					if (fields.length < 2) {
						const error = `第${rowNumber}行：数据格式错误，至少需要商品名称和价格两个字段`
						console.log(error)
						errors.push(error)
						continue
					}
					
					const barcode = fields[0] || ''
					const name = fields[1] || ''
					const price = fields[2] || ''
					const remark = fields[3] || ''
					
					console.log(`第${rowNumber}行数据:`, { barcode, name, price, remark })
					
					// 验证数据
					if (!name || name.trim() === '') {
						const error = `第${rowNumber}行：商品名称不能为空（必填字段）`
						console.log(error)
						errors.push(error)
						continue
					}
					
					if (!price || price.trim() === '') {
						const error = `第${rowNumber}行：商品价格不能为空（必填字段）`
						console.log(error)
						errors.push(error)
						continue
					}
					
					// 验证价格是否为有效数字
					const priceNum = parseFloat(price)
					if (isNaN(priceNum) || priceNum < 0) {
						const error = `第${rowNumber}行：商品价格必须是有效的正数，当前值："${price}"`
						console.log(error)
						errors.push(error)
						continue
					}
					
					// 检查重复
					let existingIndex = -1
					if (barcode && barcode.trim() !== '') {
						// 先按条形码查找（条形码是选填的，但如果有值则优先按条形码查找）
						existingIndex = existingProducts.findIndex(p => p.barcode && p.barcode === barcode)
						if (existingIndex !== -1) {
							console.log(`第${rowNumber}行：发现重复条形码"${barcode}"，将更新现有商品`)
						}
					}
					
					if (existingIndex === -1) {
						// 再按商品名称查找
						existingIndex = existingProducts.findIndex(p => p.name && p.name === name)
						if (existingIndex !== -1) {
							console.log(`第${rowNumber}行：发现重复商品名称"${name}"，将更新现有商品`)
						}
					}
					
					// 创建或更新商品
					const product = {
						id: this.generateId(),
						barcode: barcode.trim(), // 条形码（选填）
						name: name.trim(), // 商品名称（必填）
						price: priceNum.toString(), // 商品价格（必填，数值）
						remark: remark.trim(), // 备注（选填）
						createTime: new Date().getTime(),
						queryTime: new Date().getTime()
					}
					
					if (existingIndex !== -1) {
						// 更新现有商品
						const existingProduct = existingProducts[existingIndex]
						product.id = existingProduct.id
						product.createTime = existingProduct.createTime
						existingProducts[existingIndex] = product
						updateCount++
						console.log(`第${rowNumber}行：更新商品`, product)
					} else {
						// 添加新商品
						existingProducts.push(product)
						newCount++
						console.log(`第${rowNumber}行：新增商品`, product)
					}
				}
				
				// 保存数据
				if (errors.length === 0) {
					uni.setStorageSync('products', existingProducts)
					console.log('数据保存成功')
					return {
						success: true,
						newCount,
						updateCount,
						skipCount
					}
				} else {
					console.log('数据验证失败，错误数:', errors.length)
					return {
						success: false,
						errors
					}
				}
			},
			
			// 显示导出成功信息
			showExportSuccess(filePath) {
				console.log('显示导出成功，文件路径:', filePath)
				
				// 显示绝对路径
				uni.showModal({
					title: '导出成功',
					content: `文件已保存到:\n${filePath}\n\n文件路径已复制到剪贴板`,
					confirmText: '确定',
					showCancel: false,
					success: () => {
						// 复制文件路径到剪贴板
						uni.setClipboardData({
							data: filePath,
							success: () => {
								uni.showToast({
									title: '路径已复制',
									icon: 'success'
								})
							},
							fail: (err) => {
								console.error('复制到剪贴板失败:', err)
								uni.showToast({
									title: '路径获取成功',
									icon: 'success'
								})
							}
						})
					}
				})
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

	/* API配置 */
	.api-section {
		background: #ffffff;
		border-radius: 20rpx;
		padding: 30rpx;
		margin-bottom: 30rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.08);
	}

	.api-config {
		display: flex;
		flex-direction: column;
		gap: 20rpx;
	}

	.config-item {
		display: flex;
		flex-direction: column;
		gap: 10rpx;
	}

	.config-label {
		font-size: 28rpx;
		color: #495057;
		font-weight: 500;
	}

	.config-input {
		height: 80rpx;
		border: 2rpx solid #e9ecef;
		border-radius: 10rpx;
		padding: 0 20rpx;
		font-size: 28rpx;
		color: #212529;
		background: #f8f9fa;
	}

	.config-input:focus {
		border-color: #007bff;
		background: #ffffff;
	}

	.config-tips {
		margin-top: 10rpx;
	}

	.tips-text {
		font-size: 24rpx;
		color: #6c757d;
		line-height: 1.5;
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