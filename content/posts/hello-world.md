---
title: "你好,世界 - 我的第一篇博客"
date: 2026-07-27
draft: false
description: "博客正式上线,这篇文章展示 Markdown 的各种排版能力。"
tags: ["随笔", "开始"]
categories: ["公告"]
---

欢迎来到我的技术博客!这是第一篇文章,用来演示各种 Markdown 排版效果。

## 为什么开这个博客

俗话说:

> 好记性不如烂笔头。

写博客是整理思路最好的方式。把学到的东西用自己的话讲一遍,才算真正理解。

## 代码高亮

支持各种语言的语法高亮,右上角还有一键复制按钮:

```python
def fibonacci(n: int) -> list[int]:
    """生成前 n 个斐波那契数。"""
    seq = [0, 1]
    for i in range(2, n):
        seq.append(seq[i - 1] + seq[i - 2])
    return seq[:n]

print(fibonacci(10))
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

```bash
# 常用 Git 命令
git status
git add .
git commit -m "feat: add new post"
git push
```

## 列表与表格

无序列表:

- 写清楚问题是什么
- 记录尝试过的方案
- 总结最终结论

有序列表:

1. 提出问题
2. 分析原因
3. 给出方案
4. 验证结果

| 框架 | 语言 | 适用场景 |
|------|------|----------|
| Hugo | Go | 静态博客 |
| Hexo | Node.js | 静态博客 |
| Next.js | React | 全栈应用 |

## 行内元素

这里是一段包含 **加粗**、*斜体*、`行内代码` 和 [超链接](https://gohugo.io/) 的文字。

---

后续会持续更新技术笔记。下篇文章见 👋
