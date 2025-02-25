
## Format

```bash
gofmt
```

## 避免多余的else

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```
`err`被重新赋值


## for 循环

`range` 语句可以管理循环。

```go
for key, value := range oldMap {
    newMap[key] = value
}
```


##  命名结果参数


##
closure

[[go Closure 闭包]]
