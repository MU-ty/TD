# Translated document

此仓库存放使用LT智能体翻译后的Linux内核中文文档

---

## 目录结构

```
TD-main/
├─ index.html
├─ accel/                # 加速器相关文档（已翻译的 rst 与对应 stats.json）
│  ├─ amdxdna/
│  │  ├─ amdnpu/
│  │  │  ├─ amdnpu_translated.rst
│  │  │  └─ amdnpu_translated.stats.json
│  │  └─ index/
│  │     └─ index_translated.rst
│  ├─ index/
│  │  ├─ index_translated.rst
│  │  └─ index_translated.stats.json
│  ├─ introduction/
│  │  ├─ introduction_translated.rst
│  │  └─ introduction_translated.stats.json
│  └─ qaic/
│     ├─ aic080/
│     │  ├─ aic080_translated.rst
│     │  └─ aic080_translated.stats.json
│     ├─ aic100/
│     │  ├─ aic100_translated.rst
│     │  └─ aic100_translated.stats.json
│     └─ qaic.index/
│        ├─ index_translated.rst
│        └─ index_translated.stats.json
├─ PCI/                  # 文档目录（已翻译的 rst 与对应 stats.json）
│  ├─ boot-interrupts/
│  ├─ controller/
│  │  ├─ index/
│  │  └─ rcar-pcie-firmware/
│  ├─ pci-error-recovery/
│  ├─ pcieaer-howto/
│  └─ tph/
└─ RCU/
	├─ checklist/
	├─ index/
	├─ listRCU/
	├─ lockdep/
	├─ lockdep-splat/
	├─ NMI-GCU/
	└─ UP/
└─ README.md
```

> 注：文件名中包含 `_translated.rst` 的为翻译后的文档；`*.stats.json` 为翻译对比摘要及评分数据。网页仅展示结构，不展示正文。

---

