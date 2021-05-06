# 工具脚本

简单目录生成
```
echo "" > README.md ; for i in `find . -name '*.md' | sort -n`; do  echo "* [$i]($i)" >> README.md; done
```
