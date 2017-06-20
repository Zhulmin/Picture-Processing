## 图片处理知识笔记



#### iOS从磁盘加载一张图片，使用UIImageVIew显示在屏幕上，需要经过以下步骤：
1.从磁盘拷贝数据到内核缓冲区
2.从内核缓冲区复制数据到用户空间
3.生成UIImageView，把图像数据赋值给UIImageView
4.如果图像数据为未解码的PNG/JPG，解码为位图数据
5.CATransaction捕获到UIImageView layer树的变化
6.主线程Runloop提交CATransaction，开始进行图像渲染
  6.1 如果数据没有字节对齐，Core Animation会再拷贝一份数据，进行字节对齐。
  6.2 GPU处理位图数据，进行渲染。
 

  
* Xcode会对png图片进行解码优化, jpeg的解压算法更复杂
  
- imageNamed:与imageWithContentsOfFile:
这两个方法, 都是将数据读入内存而不进行解码，只有当图片将要显示之前才会被解码.不同点是imageNamed:方法会在第一次解码显示之后将解码后的位图进行全局缓存，只有在程序退入后台或者接收到内存警告时才会将位图释放。这也是为什么在第一次滑动tableVIew的时候会卡，之后再反复滑动就不会卡顿的原因。
调用imageNamed方法, 系统会将解码后的图片进行缓存, 调用imageWithContentsOfFile方法, 当图片被释放时, 缓存数据会消失。所以imageNamed方法加载图片会占用更多的内存。
  
* 手动解码(一)
解码和渲染的过程系统默认是一定要在主线程执行的。 
通过上下文绘制跳过SDWebImageDecoder
```Objective-C
  dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
    NSInteger index = indexPath.row;
    NSString *imagePath = self.dataArr[index];
    UIImage *image = [UIImage imageWithContentsOfFile:imagePath];
    //redraw image using device context
    UIGraphicsBeginImageContextWithOptions(imageView.bounds.size, NO, 0);
    [image drawInRect:imageView.bounds];
    image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    //set image on main thread, but only if index still matches up
    dispatch_async(dispatch_get_main_queue(), ^{
        if (index == cell.tag) {
            imageView.image = image;
        }
    });
});
```
  
  
* 手动解码(二)
1.CGImageSourceCreateWithData(data) 创建 ImageSource。
2.CGImageSourceCreateImageAtIndex(source) 创建一个未解码的 CGImage。
3.CGImageGetDataProvider(image) 获取这个图片的数据源。
4.CGDataProviderCopyData(provider) 从数据源获取直接解码的数据。
ImageIO 解码发生在最后一步，这样获得的数据是没有经过颜色类型转换的原生数据（比如灰度图像）。  
  
  

- 缓存和异步解码只是缓解CPU压力的方法之一，除此之外还有很多地方可以优化CPU和GPU资源，比如之前提到的对于圆角和阴影的处理。

- 使用CATiledLayer加载大图, 也可以优化图片显示。可以看到当滑动屏幕的时候图片的显示会呈现碎片式的淡入淡出效果。(参考地图类app)

- 字节对齐
那什么是字节对齐呢，按我的理解，为了性能，底层渲染图像时不是一个像素一个像素渲染，而是一块一块渲染，数据是一块块地取，就可能遇到这一块连续的内存数据里结尾的数据不是图像的内容，是内存里其他的数据，可能越界读取导致一些奇怪的东西混入，所以在渲染之前CoreAnimation要把数据拷贝一份进行处理，确保每一块都是图像数据，对于不足一块的数据置空。大致图示：(pixel是图像像素数据，data是内存里其他数据)
  
- 怎么能避免缓存呢  
1. 手动调用 CGImageSourceCreateWithData() 来创建图片，并把 ShouldCache 和 ShouldCacheImmediately 关掉。这么做会导致每次图片显示到屏幕时，解码方法都会被调用，造成很大的 CPU 占用。
2. 把图片用 CGContextDrawImage() 绘制到画布上，然后把画布的数据取出来当作图片。这也是常见的网络图片库的做法。

- 怎样像浏览器那样边下载边显示图片
图片本身有 3 种常见的编码方式：
第一种是 baseline，即逐行扫描。默认情况下，JPEG、PNG、GIF 都是这种保存方式。
第二种是 interlaced，即隔行扫描。PNG 和 GIF 在保存时可以选择这种格式。
第三种是 progressive，即渐进式。JPEG 在保存时可以选择这种方式。
在下载图片时，首先用 CGImageSourceCreateIncremental(NULL) 创建一个空的图片源，随后在获得新数据时调用
CGImageSourceUpdateData(data, false) 来更新图片源，最后在用 CGImageSourceCreateImageAtIndex() 创建图片来显示。  
  
  
  
  
参考的文章:
[关于iOS中图像显示的一些优化处理](http://www.jianshu.com/p/e19fcaf29c77)  
[iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
[iOS 处理图片的一些小 Tip](http://blog.ibireme.com/2015/11/02/ios_image_tips/)



其它资料:
[谈谈 iOS 中图片的解压缩](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)





