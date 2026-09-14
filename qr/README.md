# 收款码放这里

打赏区块（`/projects#sponsorship`）用的是这两张图：

| 文件 | 用途 |
| --- | --- |
| `qr/wx.png` | 微信收款码 |
| `qr/alipay.png` | 支付宝收款码 |

现在的两张是**占位图**（灰底写着"占位图 / PLACEHOLDER"），
把你自己的收款码截图**用同样的文件名覆盖**这两个文件就行，代码不用改：

```bash
cp ~/你的微信收款码.png qr/wx.png
cp ~/你的支付宝收款码.png qr/alipay.png
```

建议：正方形、边长 500~800px、扫码区域留点白边，PNG 或 JPG 都可以
（换成 jpg 记得同步改 `src/pages/projects/index.astro` 顶部的 import 后缀）。

改完本地跑 `npm run dev` 打开 http://localhost:4321/projects#sponsorship 确认能扫出来，
再 commit + push 让 Vercel 重新部署。
