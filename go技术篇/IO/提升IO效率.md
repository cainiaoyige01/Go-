### 提升IO的效率的原理：

​	减少IO的操作次数！怎么减少IO的操作次数？比如说我们可以把一些少的内容先缓冲到buffer中去，累积到缓冲快满的时候，一次把缓冲的内容全写到磁盘中去了，就可以减少了IO的操作次数了！通俗的说就是把==多分小文件收集到一个大的纸箱中去==，然后一次性把纸箱中文件全写到磁盘中去！

![IOcache](D:\Go文档\go技术篇\IO\IOcache.jpg)

```go
//定义一串要写的东西 写到磁盘中去
var log = "WriteString is like Write, but writes the contents of string s rather than a slice of bytes."

// WriteDirect 直接写入文件
func WriteDirect(filePath string) {
	//os 是操作系统的提供的包 可以对文件、目录、执行命令、信号中断、进程、系统状态等进行操作
	//参数1.文件名称 2.打开方式（os.O_TRUN 清空文件中内容再写） 3.文件权限
    //file就是一个文件句柄（相当于一把钥匙，可以对文件进行打开和关闭以及对文件进行读写等操作！）
	file, err := os.OpenFile(filePath, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Println(err)
	}
	//文件必须要关闭链接的
	defer file.Close()
	//写入文件中去
	for i := 0; i < 10000; i++ {
		file.WriteString(log)
	}
}
func main() {
	begin := time.Now()
	WriteDirect1("high_performance_go/b.txt")
	fmt.Printf("%dms\n", time.Since(begin).Milliseconds()) //耗时54ms
}

```

优化IO操作：

```go
/*
	提升io的操作的效率 原理是把小文件内容先写进缓冲中，当达到缓冲快慢的时候
	再写进文件中去，就可以减少了io的操作了，从而提升了效率
*/
// BufferFileWriter 结构体
type BufferFileWriter struct {
	//要有文件句柄
	file *os.File
	//缓冲的切片
	cache []byte
	//移动cacheIndex来判断缓冲是否满了
	cacheIndex int
}

// NewBufferFileWriter 获取初始化buffer对象
func NewBufferFileWriter(file *os.File, cacheSize int) *BufferFileWriter {
	return &BufferFileWriter{
		file:       file,
		cache:      make([]byte, cacheSize),
		cacheIndex: 0,
	}
}
// WriteBuffer 写入缓冲的方法 这个方法是属于BufferFileWrite的
func (w *BufferFileWriter) WriteBuffer(cont []byte) {
	//总共有三种可能 1.要写如文件内容大于缓冲空间buffer 直接写入 2.是缓冲buffer剩余空间不够 先flush再写入 3.一种缓冲空间够大的直接写入buffer 
	if len(cont) > len(w.cache) {//要写的内容大于缓冲的空间
		//先把原来缓冲中的内容写到磁盘中去 然后把内容直接写入磁盘中去
		w.Flush()
		w.file.Write(cont)
	} else {
		if len(cont)+w.cacheIndex >= len(w.cache) {//缓冲中的剩余空间不够了
			//先把缓冲中内容写入磁盘 再把要写的内容 写入缓冲中去
			w.Flush()
		}
		copy(w.cache[w.cacheIndex:], cont)
		w.cacheIndex += len(cont)//这里记得移动缓冲的空间的index
	}

}

// Flush 把缓冲中内容写入到磁盘中去
func (w *BufferFileWriter) Flush() {
	w.file.Write(w.cache[0:w.cacheIndex])
	w.cacheIndex = 0
}
// WriteBufferString 参数传经来的是string
func (w *BufferFileWriter) WriteBufferString(cont string) {
	w.WriteBuffer([]byte(cont))
}
// WriteDirect1 使用缓冲来写入磁盘中去
func WriteDirect1(filePath string) {
	//os 是操作系统的提供的包 可以对文件、目录、执行命令、信号中断、进程、系统状态等进行操作
	//参数1.文件名称 2.打开方式 3.文件权限
	file, err := os.OpenFile(filePath, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Println(err)
	}
	writer := NewBufferFileWriter(file, 4096)
	//文件必须要关闭链接的
	defer file.Close()
	//把剩余缓冲的内容都写进磁盘中去
	defer writer.Flush()
	//写入文件中去
	for i := 0; i < 10000; i++ {
		//使用自己自定义的方法
		writer.WriteBufferString(log)
	}
}
func main() {
	begin := time.Now()
	WriteDirect1("E:\\workspace\\practice\\high_performance_go\\b.txt")
	fmt.Printf("%dms\n", time.Since(begin).Milliseconds())//5ms 
    //整整优化了50ms之多！
}

```

