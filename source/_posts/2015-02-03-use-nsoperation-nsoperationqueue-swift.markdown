---
layout: post
title: "在Swift中使用NSOperation 和NSOperationQueue"
date: 2015-02-03 17:41
comments: true
categories: Swift GCD
---

> 译自[NSOperation and NSOperationQueue Tutorial in Swift](http://www.raywenderlich.com/76341/use-nsoperation-nsoperationqueue-swift)。

每个人都有这样沮丧的经历：在iOS或Mac应用中，点一个按钮或输入一些文字，突然 —— 界面失去响应。

在Mac上，用户不得不盯着沙漏或转圈圈的图标直到重新响应。而在iOS应用中，用户期望程序能立刻响应他们的点触。响应性不佳的程序让人感到笨拙缓慢，通常会受到差评。

保持程序响应性是一件说起来容易做起来难的事。一旦你的应用需要执行很多任务，事情很快变得复杂起来。当在主线程中执行繁重的任务时，没办法保证UI响应。

可怜的开发者该怎么办？解决办法就是通过并发将工作量从主线程中转移出去。并发意味着程序同时执行多个工作流（或线程）—— 从而在执行任务的同时保持UI响应。

在iOS中实现并发的一种方式是使用 **NSOperation** 和 **NSOperationQueue**。在这篇教程中，你会学到怎样使用它们！你从一个完全不使用并发的应用起步，所以它看起来很慢。接下来你会重构这个应用，加入并发 —— 但愿 —— 会提供更好的响应性。

<!-- more -->

## 起步

示例项目的功能就是用tableview来展示经过滤镜处理的图片。图片会从网络下载，然后添加滤镜，最后展示在tableview中。

下面是应用的示意图：

![](http://cdn5.raywenderlich.com/wp-content/uploads/2012/08/NSOperation_model_stalled.png)

初始模型
***

### 第一次尝试

下载[示例工程](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/ClassicPhotos-Starter6_1_1.zip)。

> **注意**：所有图片来源于[stock.xchng](http://sxc.hu/)。一些图片有意的拼错名字，用来测试下载失败的情况。

运行工程，（最终）你会看到应用展示一列图片。试着滚动列表。很痛苦，不是吗？

![](http://cdn1.raywenderlich.com/wp-content/uploads/2014/07/classicphotos-stalled-screenshot-213x320.png)

所有的动作都发生在 **ListViewController.swift** 中，并且大多数都在`tableView(_:cellForRowAtIndexPath:)`里。

检查一下这个方法，注意到有两件事情很繁重：

1. **从网上加载图片数据**。即使这是很简单的工作，应用仍然要等待下载完成才可以继续向下执行。
2. **用Core Image为图片添加滤镜**。方法为图片添加sepia滤镜。如果想了解更多Core Image滤镜，查看[Beginning Core Image in Swift](http://www.raywenderlich.com/?p=76285)。

除此之外，你还在第一次请求时从网络加载图片列表：

```
 lazy var photos = NSDictionary(contentsOfURL:dataSourceURL)
```
 
所有这些工作都是在主线程中执行的。因为主线程需要响应用户操作，所以让主线程忙于网络加载或是渲染图片会扼杀应用的响应能力。你可以通过Xcode的gauges视图快速查看到这种影响。在程序运行时，展开 **Debug navigator** (Command-6)然后选择 **CPU** 。

![Gauges View](http://cdn1.raywenderlich.com/wp-content/uploads/2014/07/gauges.png)

你可以看到线程1（主线程）中的尖峰。想了解更多细节，可以使用[Instruments](http://www.raywenderlich.com/?p=23037)。

是时候思考一下如何提升用户体验了！

### 任务（Tasks，线程（Threads）和进程（Processes）

在深入学习这篇教程之前，需要首先熟悉一些技术概念：

- **任务**：一件需要完成的工作。
- **线程**：操作系统所提供的机制，使得一个单独的应用程序可以同时执行多个指令。
- **进程**：一大块可执行代码，可以由多个线程组成。

> **注意**：在iOS和OS X中，线程功能由POSIX线程API（或pthreads）提供。这是很底层的操作，稍有不慎便会引发错误；并且线程的错误很难发现。

> Foundation框架包含一个称作NSThread的类，它更容易使用，但是用它来管理多个线程仍然很令人头疼。NSOperation 和 NSOperationQueue是更高层的类，使用它们操作多线程会更加简单。

![Process, Thread, Task](http://cdn1.raywenderlich.com/wp-content/uploads/2012/08/Process_Thread_Task.png)

如你所见，进程可以包含多个线程，每个线程可以同时执行多个任务。

在上图中，线程2在读文件，与此同时线程1在执行UI操作。这和你在iOS中组织代码的方式相同 —— 主线程执行UI相关的工作，同时从属线程执行慢速或需要长时间运行的任务，比如读文件，网络请求等。

### NSOperation vs. Grand Central Dispatch(GCD)

你可能听说过GCD。简单说来，GCD由语言特性，运行时(runtime)库，系统优化组成，它提供了系统级的改进来支持多核硬件在iOS和OS X下的并发。如想了解更多GCD的内容，可以读我们的[教程](http://www.raywenderlich.com/?p=4295)。

快速比较一下两者，帮助你选择何时使用GCD或NSOperation：

- GCD以一种轻量级的方式来表示并发的工作。你不会规划（schedule）这些工作，而是由系统来为你规划。在任务间添加依赖会很头疼。取消或挂起任务需要额外的工作量！:]
- **NSOperation** 较之GCD增加了一些额外的开销，但是你可以在多个任务间添加依赖，并且可以重用，取消或是挂起它们。

这篇教程会使用 **NSOperation** 是因为你需要关心tableview的表现以及电量消耗，在用户滚动屏幕时，你要能够取消那些已经划出屏幕的图片的任务。即使操作是在后台进程中，但是如果有大量工作在后台队列等待执行，应用的表现仍然会难以忍受。

## 重定义的App模型

是时候重新定义先前的非线程模型了！如果你仔细观察先前的模型，你会发现有三个可改善的地方，它们导致了线程陷入困境。通过分离并放置到单独的线程，主线程得到解放并保证UI响应性。

![](http://cdn3.raywenderlich.com/wp-content/uploads/2012/08/NSOperation_model_improved.png)

改良模型
***

为了突破程序的瓶颈，你需要一个线程响应用户交互，一个线程下载数据和图片，以及一个线程为图片添加滤镜。在新模型中，应用从主线程开始并加载一个空的tableview。同时，应用程序启动第二个线程下载数据。

一旦数据下载完成，你会重新加载tableview。这必须在主线程完成，因为涉及到UI。此时，tableview知道有多少行，并且知道每个图片的URL，但是它还没有实际的图片！如果此时你立即开始下载所有图片，那会非常低效，因为你不需要一次性下载所有图片！

怎样改进？

更好的模型是仅仅下载那些当前屏幕中可见的图片。所以你的首要任务是询问tableview哪些行可见，然后再下载它们。同时，在图片下载完成前不能为它添加滤镜。因此，只有在未添加滤镜的图片等待处理的情况下才应该执行添加滤镜的代码。

为了使程序看起来响应更灵敏，图片在下载后应该立刻呈现。然后开始为图片添加滤镜，最后更新UI来展示滤镜图片。下图展示了处理的流程：

![](http://cdn4.raywenderlich.com/wp-content/uploads/2012/09/NSOperation_workflow.png)

为了达成目标，你需要追踪图片当前是否正在下载，是否下载完成，或者滤镜是否添加完成。你同样需要追踪操作的状态，以及它是下载操作还是滤镜操作，这样在用户滚动时你才能取消，暂停或继续这些操作。

好的！你已经准备好写代码了！:]

打开工程，添加新的 **Swift文件** 并命名为 **PhotoOperations.swift**。添加如下代码：

```
import UIKit
 
// This enum contains all the possible states a photo record can be in
enum PhotoRecordState {
  case New, Downloaded, Filtered, Failed
}
 
class PhotoRecord {
  let name:String
  let url:NSURL
  var state = PhotoRecordState.New
  var image = UIImage(named: "Placeholder")
 
  init(name:String, url:NSURL) {
    self.name = name
    self.url = url
  }
}
```

> **注意**：确保在文件顶部 **import UIKit**。默认情况下，Xcode只会在Swift文件中 import Foundation。

这个简单的类表示app中展示的图片，其中包含当前状态，新建记录默认是 **.New**。图片默认显示为占位图片。

为了追踪每个操作的状态，你需要一个单独的类。添加如下定义到 **PhotoOperations.swift**：

```
class PendingOperations {
  lazy var downloadsInProgress = [NSIndexPath:NSOperation]()
  lazy var downloadQueue:NSOperationQueue = {
    var queue = NSOperationQueue()
    queue.name = "Download queue"
    queue.maxConcurrentOperationCount = 1
    return queue
    }()
 
  lazy var filtrationsInProgress = [NSIndexPath:NSOperation]()
  lazy var filtrationQueue:NSOperationQueue = {
    var queue = NSOperationQueue()
    queue.name = "Image Filtration queue"
    queue.maxConcurrentOperationCount = 1
    return queue
    }()
}
```

这个类包含两个字典(dictionaries)来追踪进行中的和等待中的下载和滤镜操作，以及两种操作队列。

所有值都是惰性的，意味着它们只有在第一次使用时才会被初始化。这可以优化应用的表现。

创建一个 `NSOperationQueue` 非常简单。为队列命名很有帮助，因为名字会在Instruments或调试器中出现。`maxConcurrentOperationCount`设成1是出于这篇教程考虑，这能够让你看到任务一个接一个完成。你可以移掉这行代码，让队列自己决定可以同时处理多少操作 —— 可以进一步提升性能。

队列怎样决定同时运行多少任务？好问题！:] 这取决于硬件。默认情况下，`NSOperationQueue`会在后台做一些运算以决定什么设置是最适合当前平台的，然后会启动最大可能数量的线程。

思考下面的例子。假设系统处于空闲状态，有大量资源可用，所以队列可以启动8个线程。而下一次运行程序时，系统可能忙于其他消耗资源的任务，因此队列只启动了两个线程。因为你设置了最大并发操作数，当前的应用同一时刻只能执行一个操作。

> **注意**：你可能好奇为什么要追踪所有进行中和等待中的任务。既然队列有一个`operations`方法返回任务数组，为什么不用它呢？在这个项目中，这么做不是很高效。因为你需要追踪哪个任务对应到tableview中的哪一行，这会导致每次都遍历数组。而将操作保存在字典中，以index path为键（key），查询就会很高效且迅速。

是时候关注下载和滤镜任务了。添加下面的代码到 **PhotoOperations.swift**：

```
class ImageDownloader: NSOperation {
  //1
  let photoRecord: PhotoRecord
 
  //2
  init(photoRecord: PhotoRecord) {
    self.photoRecord = photoRecord
  }
 
  //3
  override func main() {
    //4
    if self.cancelled {
      return
    }
    //5
    let imageData = NSData(contentsOfURL:self.photoRecord.url)
 
    //6
    if self.cancelled {
      return
    }
 
    //7
    if imageData?.length > 0 {
      self.photoRecord.image = UIImage(data:imageData!)
      self.photoRecord.state = .Downloaded
    }
    else
    {
      self.photoRecord.state = .Failed
      self.photoRecord.image = UIImage(named: "Failed")
    }
  }
}
```

`NSOperation`是一个抽象类，它被设计成基类让子类继承。每个子类表示一种特定的 **任务**。

注释：

1. 添加对`PhotoRecord`的常量引用。
2. 创建一个指定构造函数来传入photo record。
3. 在子类中重载`NSOperation`的`main`方法来执行实际的任务。
4. 在开始执行前检查撤消状态。任务在试图执行繁重的工作前应该检查它是否已经被撤消。
5. 下载图片。
6. 再一次检查撤销状态。
7. 如果有数据，创建一个图片对象并加入记录，然后更改状态。如果没有数据，将记录标记为失败并设置失败图片。

接下来，创建另一个滤镜任务！添加如下代码到 **PhotoOperations.swift** 底部：

```
class ImageFiltration: NSOperation {
  let photoRecord: PhotoRecord
 
  init(photoRecord: PhotoRecord) {
    self.photoRecord = photoRecord
  }
 
  override func main () {
    if self.cancelled {
      return
    }
 
    if self.photoRecord.state != .Downloaded {
      return
    }
 
    if let filteredImage = self.applySepiaFilter(self.photoRecord.image!) {
      self.photoRecord.image = filteredImage
      self.photoRecord.state = .Filtered
    }
  }
}
```

看起来和下载任务的代码很相似，除了你对图片应用滤镜（还没实现的方法，编译器会报错）而非下载。

添加缺失的滤镜方法到 `ImageFiltration`类：

```
func applySepiaFilter(image:UIImage) -> UIImage? {
  let inputImage = CIImage(data:UIImagePNGRepresentation(image))
 
  if self.cancelled {
    return nil
  }
  let context = CIContext(options:nil)
  let filter = CIFilter(name:"CISepiaTone")
  filter.setValue(inputImage, forKey: kCIInputImageKey)
  filter.setValue(0.8, forKey: "inputIntensity")
  let outputImage = filter.outputImage
 
  if self.cancelled {
    return nil
  }
 
  let outImage = context.createCGImage(outputImage, fromRect: outputImage.extent())
  let returnImage = UIImage(CGImage: outImage)
  return returnImage
}
```

图片滤镜和在`ListViewController`中的实现相同。它被移到这里，从而单独在后台执行。再一次，你应该经常检查撤销状态；应该在开销大的方法调用前后都执行检查。一旦滤镜完成，设置photo record实例。

很好！现在你拥有了处理后台任务用到的所有工具。是时候改进view controller了。

切换至 **ListViewController.swift** 并删除 `lazy var photos`属性声明。替换为如下声明：

```
var photos = [PhotoRecord]()
let pendingOperations = PendingOperations()
```

创建`PhotoRecord`数组来记录图片，并用`PendingOperations`来管理任务。

在类中添加一个新方法来下载列表中的图片：

```
func fetchPhotoDetails() {
  let request = NSURLRequest(URL:dataSourceURL!)
  UIApplication.sharedApplication().networkActivityIndicatorVisible = true
 
  NSURLConnection.sendAsynchronousRequest(request, queue: NSOperationQueue.mainQueue()) {response,data,error in
    if data != nil {
      let datasourceDictionary = NSPropertyListSerialization.propertyListFromData(data,
        mutabilityOption: .Immutable, format: nil, errorDescription: nil) as NSDictionary
 
      for(key : AnyObject,value : AnyObject) in datasourceDictionary {
        let name = key as? String
        let url = NSURL(string:value as? String ?? "")
        if name != nil && url != nil {
          let photoRecord = PhotoRecord(name:name!, url:url!)
          self.photos.append(photoRecord)
        }
      }
 
      self.tableView.reloadData()
    }
 
    if error != nil {
      let alert = UIAlertView(title:"Oops!",message:error.localizedDescription, delegate:nil, cancelButtonTitle:"OK")
      alert.show()
    }
    UIApplication.sharedApplication().networkActivityIndicatorVisible = false
  }
}
```

这个方法执行异步网络请求，当完成后，会在主队列中执行完成闭包。当下载完成后，属性列表数据会存入`NSDictionary`并进一步转化成`PhotoRecord`数组。这里没有直接用到`NSOperation`，而是使用`NSOperationQueue.mainQueue()`来获取主队列。

在`viewDidLoad`最后调用新方法：
```
fetchPhotoDetails()
```

接下来，找到`tableView(_:cellForRowAtIndexPath:)`并替换成下面的实现：

```
override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
  let cell = tableView.dequeueReusableCellWithIdentifier("CellIdentifier", forIndexPath: indexPath) as UITableViewCell
 
  //1
  if cell.accessoryView == nil {
    let indicator = UIActivityIndicatorView(activityIndicatorStyle: .Gray)
    cell.accessoryView = indicator
  }
  let indicator = cell.accessoryView as UIActivityIndicatorView
 
  //2
  let photoDetails = photos[indexPath.row]
 
  //3
  cell.textLabel?.text = photoDetails.name
  cell.imageView?.image = photoDetails.image
 
  //4
  switch (photoDetails.state){
  case .Filtered:
    indicator.stopAnimating()
  case .Failed:
    indicator.stopAnimating()
    cell.textLabel?.text = "Failed to load"
  case .New, .Downloaded:
    indicator.startAnimating()
    self.startOperationsForPhotoRecord(photoDetails,indexPath:indexPath)
  }
 
  return cell
}
```

来看一下注释：

1. 为了提示用户，将cell的accessory view设置为`UIActivityIndicatorView`。
2. `photos`包含`PhotoRecord`实例。获取当前行所对应的图片记录。
3. cell的文本（差不多）总是相同的，图片会在`PhotoRecord`中处理，所以可以在这里使用它们而不用关心图片的状态。
4. 检查图片。设置适当的activity indicator 和文本，然后开始执行任务（暂未实现）。

可以移除`applySepiaFilter`，因为这里不需要再调用它了。添加下面的方法来开始任务：

```
func startOperationsForPhotoRecord(photoDetails: PhotoRecord, indexPath: NSIndexPath){
  switch (photoDetails.state) {
  case .New:
    startDownloadForRecord(photoDetails, indexPath: indexPath)
  case .Downloaded:
    startFiltrationForRecord(photoDetails, indexPath: indexPath)
  default:
    NSLog("do nothing")
  }
}
```

这里传入一个`PhotoRecord`实例和它的index path。根据图片状态，下载或添加滤镜。

> **注意**：下载和滤镜是分别实现的，因为有可能在图片下载时，用户将图片划动出屏幕了，这时不会再添加滤镜。当下一次用户划到当前行时，就不需要再下载图片了；只需要添加滤镜！高效的做法！:]

现在你需要实现上面所调用的方法。别忘了你有一个`PendingOperations`类来跟踪任务；现在可以使用它了！添加下面的代码：

```
func startDownloadForRecord(photoDetails: PhotoRecord, indexPath: NSIndexPath){
  //1
  if let downloadOperation = pendingOperations.downloadsInProgress[indexPath] {
    return
  }
 
  //2
  let downloader = ImageDownloader(photoRecord: photoDetails)
  //3
  downloader.completionBlock = {
    if downloader.cancelled {
      return
    }
    dispatch_async(dispatch_get_main_queue(), {
      self.pendingOperations.downloadsInProgress.removeValueForKey(indexPath)
      self.tableView.reloadRowsAtIndexPaths([indexPath], withRowAnimation: .Fade)
    })
  }
  //4
  pendingOperations.downloadsInProgress[indexPath] = downloader
  //5
  pendingOperations.downloadQueue.addOperation(downloader)
}
 
func startFiltrationForRecord(photoDetails: PhotoRecord, indexPath: NSIndexPath){
  if let filterOperation = pendingOperations.filtrationsInProgress[indexPath]{
    return
  }
 
  let filterer = ImageFiltration(photoRecord: photoDetails)
  filterer.completionBlock = {
    if filterer.cancelled {
      return
    }
    dispatch_async(dispatch_get_main_queue(), {
      self.pendingOperations.filtrationsInProgress.removeValueForKey(indexPath)
      self.tableView.reloadRowsAtIndexPaths([indexPath], withRowAnimation: .Fade)
      })
  }
  pendingOperations.filtrationsInProgress[indexPath] = filterer
  pendingOperations.filtrationQueue.addOperation(filterer)
}
```

好！确保你明白上面代码的工作原理：

1. 首先，在`downloadInProgress`中检查对应`indexPath`的任务是否存在。如果存在，则不需要处理。
2. 如果不存在，通过指定构造器（designated initializer）创建`ImageDownloader`实例。
3. 加入完成闭包，它会在任务完成后执行。在这里通知你的程序一件任务已经完成最合适不过。需要注意的是，即使任务取消，闭包也会执行，因此在做任何工作前你必须检查任务是否被取消。你同样无法保证完成闭包在哪个线程中调用，所以需要用GCD在主线程中重新加载tableview。
4. 将任务加入`downloadsInProgress`来跟踪它。
5. 将任务加入下载队列。这里是任务实际执行的地方 —— 一旦你将任务加入队列，会由队列替你规划任务。

给图片添加滤镜的方法遵循相同的模式，只不过它用`ImageFiltration`和`filtrationsInProgress`来跟踪任务。作为练习，你可以尝试去除重复代码 :]

你做到了！你的项目完成了。运行应用来查看改进效果！当你滚动tableview时，应用不会再卡顿，当图片可见时才开始下载和加载滤镜。

![](http://cdn1.raywenderlich.com/wp-content/uploads/2014/07/iOS-Simulator-Screen-Shot-8-Jul-2014-20.57.53-333x500.png)

是不是很cool？你可以发现一点努力便能使你的程序大大改进 —— 并给用户带来更多乐趣！

## 调优

你已经学到很多了！你的小项目相比初始版本改进了许多。然而，仍然有一些细节需要关注。你想成为伟大的程序员，而不仅仅是好的程序员！

你也许注意到当你滚动tableview时，那些划出屏幕的cell仍然在下载或加载滤镜。如果你快速滚动屏幕，程序会忙于下载图片和为图片加载滤镜，即使图片已经不可见。理想中，程序应该取消那些不可见图片的任务，而优先处理当前展示的cell。

你之前已经加入了取消任务的代码 —— 现在应该使用它们了！:]

打开 **ListViewController.swift**。找到`tableView(_:cellForRowAtIndexPath:)`，将`startOperationsForPhotoRecord`包进if语句：

```
if (!tableView.dragging && !tableView.decelerating) {
  self.startOperationsForPhotoRecord(photoDetails, indexPath: indexPath)
}
```

你告诉tableview *只有停止滚动后* 才开始执行任务。这些实际上是`UIScrollView`的属性，因为`UITableView`是`UIScrollView`的子类，所以自动继承了这些属性。

接下来，实现`UIScrollView`代理方法：

```
override func scrollViewWillBeginDragging(scrollView: UIScrollView) {
  //1
  suspendAllOperations()
}
 
override func scrollViewDidEndDragging(scrollView: UIScrollView, willDecelerate decelerate: Bool) {
  // 2
  if !decelerate {
    loadImagesForOnscreenCells()
    resumeAllOperations()
  }
}
 
override func scrollViewDidEndDecelerating(scrollView: UIScrollView) {
  // 3
  loadImagesForOnscreenCells()
  resumeAllOperations()
}
```

快速浏览以上代码：

1. 一旦用户开始滚动屏幕，你将挂起所有任务并留意用户想要看哪些行。稍后会实现`suspendAllOperations`。
2. 如果减速（decelerate）是`false`，表示用户停止拖拽tableview。此时你要继续执行之前挂起的任务，撤销不在屏幕中的cell的任务并开始在屏幕中的cell的任务。稍后实现`loadImagesForOnscreenCells`和`resumeAllOperations`。
3. 这个代理方法告诉你tableview停止滚动，执行#2中相同的操作。

现在，在 **ListViewController.swift** 中实现方法：

```
func suspendAllOperations () {
  pendingOperations.downloadQueue.suspended = true
  pendingOperations.filtrationQueue.suspended = true
}
 
func resumeAllOperations () {
  pendingOperations.downloadQueue.suspended = false
  pendingOperations.filtrationQueue.suspended = false
}
 
func loadImagesForOnscreenCells () {
  //1
  if let pathsArray = tableView.indexPathsForVisibleRows() {
    //2
    let allPendingOperations = NSMutableSet(array:pendingOperations.downloadsInProgress.keys.array)
    allPendingOperations.addObjectsFromArray(pendingOperations.filtrationsInProgress.keys.array)
 
    //3
    let toBeCancelled = allPendingOperations.mutableCopy() as NSMutableSet
    let visiblePaths = NSSet(array: pathsArray)
    toBeCancelled.minusSet(visiblePaths)
 
    //4
    let toBeStarted = visiblePaths.mutableCopy() as NSMutableSet
    toBeStarted.minusSet(allPendingOperations)
 
    // 5
    for indexPath in toBeCancelled {
      let indexPath = indexPath as NSIndexPath
      if let pendingDownload = pendingOperations.downloadsInProgress[indexPath] {
        pendingDownload.cancel()
      }
      pendingOperations.downloadsInProgress.removeValueForKey(indexPath)
      if let pendingFiltration = pendingOperations.filtrationsInProgress[indexPath] {
        pendingFiltration.cancel()
      }
      pendingOperations.filtrationsInProgress.removeValueForKey(indexPath)
    }
 
    // 6
    for indexPath in toBeStarted {
      let indexPath = indexPath as NSIndexPath
      let recordToProcess = self.photos[indexPath.row]
      startOperationsForPhotoRecord(recordToProcess, indexPath: indexPath)
    }
  }
}
```

`suspendAllOperations`和`resumeAllOperations`实现都很简单。`NSOperationQueues`可以被挂起，只需将`suspended`属性设为`true`。这会挂起队列中所有任务 —— 你不能挂起某个单独的任务。

`loadImagesForOnscreenCells`有一点复杂。它的工作原理如下：

1. 开始将tableview可见行的index path放入数组中。
2. 通过组合所有下载队列和滤镜队列中的任务来创建一个包含所有等待任务的集合。
3. 构建一个需要撤销的任务的集合。从所有任务中除掉可见行的index path，剩下的就是屏幕外的行所代表的任务。
4. 创建一个需要执行的任务的集合。从所有可见index path的集合中除去那些已经在等待队列中的。
5. 遍历需要撤销的任务，撤消它们，然后从`PendingOperations`中去掉它们。
6. 遍历需要开始的任务，调用`startOperationsForPhotoRecord`。

运行程序，你会得到一个响应性更佳，资源管理更好的程序。给自己一点掌声！

![](http://cdn1.raywenderlich.com/wp-content/uploads/2014/09/improved-700x350.png)

注意到当你停止滚动tableview时，可见行上的图片才开始处理。

## 下一步

[完成项目](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/ClassicPhotos-Final_070114.zip)。

如果你完成了这个项目并且花时间理解了它，祝贺你！你可以认为你是一个更有价值的iOS开发者了！大多软件开发公司都会因为有一两个懂得这些的人而觉得幸运。

但是当心 —— 就像深度嵌套（deeply-nested）的闭包，毫无节制的使用多线程会让你的项目难以理解和维护。线程会引入难以察觉的bug，直到你的网络很慢或代码运行在更快（或更慢）的设备上，或是运行在不同数量的核上，bug才会显现。仔细测试，并且总是使用Instruments（或你自己的观察）来验证引入线程确实做出了改进。

这里还有一个没涉及到的特性 —— 依赖。你可以让一项任务依赖于另一项或多项任务。那么这项任务直到它所依赖的任务完成后才开始。例如：

```
// MyDownloadOperation is a subclass of NSOperation
let downloadOperation = MyDownloadOperation()
// MyFilterOperation  is a subclass of NSOperation
let filterOperation = MyFilterOperation()
 
filterOperation.addDependency(downloadOperation)
```

去除依赖：
```
filterOperation.removeDependency(downloadOperation)
```

可以使用依赖简化这个项目的代码吗？自己尝试一下 :] 需要注意的是，如果一项任务所依赖的任务被 *撤销* 了，这项任务也会开始执行，就像它所依赖的任务完成了一样。你要牢记这一点。

译者：[loveltyoic](http://loveltyoic.com)