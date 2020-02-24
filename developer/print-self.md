# print self

打印自己的程序代码

参考网站一下网站, 自行编写.

代码集中在java,golang,python,JavaScript

[The Quine Page](http://www.nyx.net/~gthompso/quine.htm)


## golang
```golang
package main;import "fmt";func main(){s:="package main;import %q;func main(){s:=%q;fmt.Printf(s,%sfmt%s,s,string(rune(34)),string(rune(34)))}";fmt.Printf(s,"fmt",s,string(rune(34)),string(rune(34)))}
```

```golang
package main

import "fmt"
var arr = []string{"\"", "\\", "\n", "\t", ",", "%s", "{", "}", "for _, n := range index {", "package main", "import ", "fmt", "var arr = []string{", "func main() {", "build", "arr", "func build(index []int) (string, []interface{}) {", " ", "n", "t", "s, a := build(", "[]int{", "})", "9, 2, 2, 10, 0, 11, 0, 2, 12, 0, 1, 0, 0, 4, 17, 0, 1, 1, 0, 4, 17, 0, 1, 18, 0, 4, 17, 0, 1, 19, 0, 4, 17", "fmt.Printf(s, a...)", "s, a = buildSpace(4, len(arr)-1)", "var s string", "var r []interface{}", "s += ", "r = append(r, arr[n])", "return s, r", "func buildSpace(start, end int) (string, []interface{}) {", "var r []int", "for i := start; i < end; i++ {", "r = append(r, []int{0, i, 0, 4, 17}...)", "return build(r)", "[]int{0, len(arr) - 1, 0, 7, 2, 2})", "16, 2, 3, 26, 2, 3, 27, 2, 3, 8, 2, 3, 3, 28, 0, 5, 0, 2, 3, 3, 29, 2, 3, 7, 2, 3, 30, 2, 7, 2", "31, 2, 3, 32, 2, 3, 33, 2, 3, 3, 34, 2, 3, 7, 2, 3, 35, 2, 7", "s, a = build(", "13, 2, 3, 20, 21, 23, 22, 2, 3, 24, 2, 3, 25, 2, 3, 24, 2, 3, 39, 36, 2, 3, 24, 2, 3, 39, 21, 40, 22, 2, 3, 24, 2, 3, 39, 21, 37, 22, 2, 3, 24, 2, 3, 39, 21, 38, 22, 2, 3, 24, 2, 7, 2"}

func main() {
    s, a := build([]int{9, 2, 2, 10, 0, 11, 0, 2, 12, 0, 1, 0, 0, 4, 17, 0, 1, 1, 0, 4, 17, 0, 1, 18, 0, 4, 17, 0, 1, 19, 0, 4, 17})
    fmt.Printf(s, a...)
    s, a = buildSpace(4, len(arr)-1)
    fmt.Printf(s, a...)
    s, a = build([]int{0, len(arr) - 1, 0, 7, 2, 2})
    fmt.Printf(s, a...)
    s, a = build([]int{13, 2, 3, 20, 21, 23, 22, 2, 3, 24, 2, 3, 25, 2, 3, 24, 2, 3, 39, 36, 2, 3, 24, 2, 3, 39, 21, 40, 22, 2, 3, 24, 2, 3, 39, 21, 37, 22, 2, 3, 24, 2, 3, 39, 21, 38, 22, 2, 3, 24, 2, 7, 2})
    fmt.Printf(s, a...)
    s, a = build([]int{16, 2, 3, 26, 2, 3, 27, 2, 3, 8, 2, 3, 3, 28, 0, 5, 0, 2, 3, 3, 29, 2, 3, 7, 2, 3, 30, 2, 7, 2})
    fmt.Printf(s, a...)
    s, a = build([]int{31, 2, 3, 32, 2, 3, 33, 2, 3, 3, 34, 2, 3, 7, 2, 3, 35, 2, 7})
    fmt.Printf(s, a...)
}
func build(index []int) (string, []interface{}) {
    var s string
    var r []interface{}
    for _, n := range index {
        s += "%s"
        r = append(r, arr[n])
    }
    return s, r
}
func buildSpace(start, end int) (string, []interface{}) {
    var r []int
    for i := start; i < end; i++ {
        r = append(r, []int{0, i, 0, 4, 17}...)
    }
    return build(r)
}
```
