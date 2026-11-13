# zq-file-kit

通用文件「上传 + 下载」工具包，可整体拷贝到其它 uni-app x 项目复用，兼容 Vapor 模式。

## 目录结构

```
zq-file-kit/
├── index.uts          # 统一导出入口
├── fileUpload.uts     # 文件上传（content:// 转沙盒 + 原始文件名）
├── fileDownload.uts   # 浏览器下载 / 下载后预览
└── README.md          # 本说明
```

## 引入方式

把整个 `zq-file-kit/` 文件夹拷贝到目标项目的 `src/`（或与 `pages/` 同级）下，
然后按你的路径别名引入：

```uts
import { uploadFile, openInBrowser, downloadAndOpen } from '@/zq-file-kit'
```

> 路径别名 `@` 需指向项目根目录；若你的别名不同，改 import 里的前缀即可。

---

## 一、文件上传 `uploadFile`

解决两个实际问题：
1. Android 10+ 分区存储下 `uni.chooseFile` 返回的是 `content://` URI，
   部分机型（尤其 MIUI 从云盘/微信/系统文件提供器选文件）直接透传给
   `uni.uploadFile` 会触发底层 Cursor 异常（`Couldn't read row 0, col -1`）。
   本工具自动把 `content://` 复制到沙盒再上传。
2. 复制时保留**原始文件名**（时间戳只放在子目录上隔离，不污染文件名），
   后端收到的 multipart `filename` 与客户选的一致。

### 用法

```uts
import { uploadFile } from '@/zq-file-kit'

uni.chooseFile({
	count: 1,
	type: 'all',
	success: (result) => {
		if (result.tempFiles.length == 0) return
		const f = result.tempFiles[0]
		uploadFile({
			filePath: f.path,          // content:// 或普通路径都行
			fileName: f.name,          // 原始文件名（保留原名用）
			url: 'https://your.host/api/upload',
			header: { 'Sign': '你的签名' },   // 签名/鉴权，按需
			fieldName: 'file',          // 默认 file
		}).then((res) => {
			console.log('上传成功，文件路径：', res.filePath)
		}).catch((err) => {
			console.error('上传失败：', err.message)
		})
	}
})
```

### 完整参数 `UploadOptions`

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `filePath` | string | 是 | 本地文件路径，支持 `content://` 或普通绝对路径 |
| `url` | string | 是 | 上传接口完整地址（含协议域名） |
| `fileName` | string? | 否 | 原始文件名，用于沙盒命名与后端 filename |
| `fieldName` | string? | 否 | multipart 表单字段名，默认 `file` |
| `header` | UTSJSONObject? | 否 | 额外请求头（签名 Sign、token 等） |
| `formData` | UTSJSONObject? | 否 | 额外表单参数 |
| `timeout` | number? | 否 | 超时毫秒，默认 120000 |
| `allowedExtensions` | string[]? | 否 | 扩展名白名单（不含点、小写），默认 `pdf/jpg/jpeg/png/doc/docx/zip/rar` |
| `isSuccess` | (body)=>boolean? | 否 | 业务成功判定，默认 `code == 1` |
| `extractPath` | (body)=>string? | 否 | 从响应取文件路径，默认依次尝试 `full_path/file/path/url` |

### 返回值 `UploadResult`

```
{ filePath: string, response: UTSJSONObject }
```

- `filePath`：后端返回的文件路径（用于回填表单/回显）
- `response`：后端返回的完整 JSON 对象（已解析）

### 适配你自己的后端

不需要改 `fileUpload.uts`，用参数就能适配：

```uts
uploadFile({
	filePath, fileName, url: 'https://xxx/upload',
	isSuccess: (body) => body.getNumber('status') == 200,   // 你的成功码
	extractPath: (body) => body.getString('data').toString(), // 你的路径字段
	allowedExtensions: ['jpg', 'png', 'pdf', 'doc'],          // 你的白名单
})
```

---

## 二、浏览器下载 `openInBrowser`

跳转系统浏览器打开链接，压缩包（zip/rar）会触发浏览器下载，绕开 App 沙盒限制。

### 依赖

**需要 DCloud 官方插件 `uts-openSchema`**（在 HBuilderX 插件市场搜索并导入，
会装到 `uni_modules/uts-openSchema`）。未安装时，App 端会提示「请升级到支持浏览器打开的版本」。

### 用法

```uts
import { openInBrowser } from '@/zq-file-kit'

openInBrowser('https://your.host/upload/xxx.zip')
```

> 浏览器打开链接的行为取决于文件类型：zip/rar 触发下载，图片/PDF 直接预览。

---

## 三、下载后预览 `downloadAndOpen`

下载到本地后用系统文档预览器打开（App 端会调起「打开方式」选择应用）。

```uts
import { downloadAndOpen } from '@/zq-file-kit'

downloadAndOpen('https://your.host/upload/xxx.pdf', 'pdf')
```

---

## 附：常用文件类型判断（供 UI 展示）

```uts
function fileExtension(name: string): string {
	const clean = name.split('?')[0]
	const i = clean.lastIndexOf('.')
	return i >= 0 ? clean.substring(i + 1).toLowerCase() : ''
}
// 图片：png/jpg/jpeg/gif/webp/bmp
// 压缩包：zip/rar/7z/tar/gz/bz2
```

---

## 注意事项

1. **仅 App 端完整支持**：`content://` 转沙盒、浏览器下载是 App 行为；小程序/Web 端行为不同，需自行判断。
2. **浏览器下载依赖插件**：`openInBrowser` 需要 `uts-openSchema`；iOS 还需自定义基座。
3. **临时副本不主动删除**：上传复制到沙盒的临时文件放在 `CACHE_PATH`，由系统缓存清理回收，无需手动删。
4. **上传 url/header 完全参数化**：本工具不依赖项目的签名逻辑（如 `createSign`），签名请通过 `header` 参数传入。
