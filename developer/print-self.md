# print self

打印自己的程序代码

参考网站一下网站, 自行编写.

代码集中在java,golang,python,JavaScript

[The Quine Page](http://www.nyx.net/~gthompso/quine.htm)


## golang
```go
package main;import "fmt";func main(){s:="package main;import %q;func main(){s:=%q;fmt.Printf(s,%sfmt%s,s,string(rune(34)),string(rune(34)))}";fmt.Printf(s,"fmt",s,string(rune(34)),string(rune(34)))}
```
