---
layout: post
title: Swift Reusable Code Segment
---

可复用的swift代码段。为什么用英文做标题？为了突出“复用”
>文章的目的：记录经常使用的代码块方便复用
更新次数：将会不定期持续更新
内容来源：有些是从优秀开源代码中摘要的，有些是自己写的。在此不多做说明。
文章权限：欢迎收藏转载，但请勿用于商业用途

一定程度上说，编写代码是一件重复性很高的工作。当然我并不是说写出优质的代码是一件容易的事情，相反我想表达的是我们在大多数情况下会重复以前开发过的内容。所以代码的复用就显得尤为重要。说到这里我不得不分享一下，最近看到的一篇关于面向对象设计原则的文章。虽然是以Android为描述语言，但思路是相通的不耽误学习。

>面向对象的6个基本原则： 
* 1、单一职责原则：一个类中应该是一组相关性很高的函数、数据的封装。
*  2、开闭原则：简单的说就是，一个类只对扩展开放，对修改关闭。
*  3、里氏替换原则：所有引用基类的地方必须能透明地使用其子类的对象，简单的说就是父类能够使用的地方子类一定能够使用。 
* 4、接口隔离原则：客户端不应该依赖它不需要的接口。可以理解为一个方法中应该避免过多的接口依赖。
*  5、依赖倒置原则：模块间的依赖通过抽象发生，实现类之间不发生直接的依赖关系，其依赖关系是通过接口或抽象类产生的。既面向接口编程，或者面向抽象类编程。 
* 6、迪米特原则：也叫最少知识原则。意思是，一个对象应该对其他对象有最少的了解。可以理解为，一连串的相关处理事件应该抽离封装出来。对于调用这个封装函数的调用者，封装代码内部逻辑是透明的。 以上六点是java面向对象的原则。其他语言应该也是适用的。
如果感兴趣可以查看原文：[面向对象的六个基本原则](http://blog.csdn.net/bboyfeiyu/article/details/50103471)

好了偏题结束。正式"Resualbe"部分。

# 基本数据类型扩展
## 1、String 扩展
~~~ swift
extension String {

    ///用于计算字符串在高度。
    ///例如，当需要动态计算TableVIewCell高度的时候，需要根据某个Label的内容String的长度来调整cell的高度，让其能够全部显示内容。
    func stringHeightWith(fontSize:CGFloat,width:CGFloat)->CGFloat

    {
        let font = UIFont.systemFontOfSize(fontSize)
        let size = CGSizeMake(width,CGFloat.max)
        let paragraphStyle = NSMutableParagraphStyle()
        paragraphStyle.lineBreakMode = .ByWordWrapping;
        let  attributes = [NSFontAttributeName:font,
            NSParagraphStyleAttributeName:paragraphStyle.copy()]
        
        let text = self as NSString
        let rect = text.boundingRectWithSize(size, options:.UsesLineFragmentOrigin, attributes: attributes, context:nil)
        return rect.size.height
    }
    
    ///用于时间转换
    func dateStringFromTimestamp(timeStamp:NSString)->String
    {
        let ts = timeStamp.doubleValue
        let  formatter = NSDateFormatter ()
        formatter.dateFormat = "yyyy年MM月dd日 HH:MM:ss"
        let date = NSDate(timeIntervalSince1970 : ts)
         return  formatter.stringFromDate(date)
    }
    
}
~~~

## 2、UIView 扩展
~~~ swift
extension UIView  {
   
    ///用于视图的坐标
    func x()->CGFloat
    {
        return self.frame.origin.x
    }
    func right()-> CGFloat
    {
        return self.frame.origin.x + self.frame.size.width
    }
    func y()->CGFloat
    {
        return self.frame.origin.y
    }
    func bottom()->CGFloat
    {
        return self.frame.origin.y + self.frame.size.height
    }
    func width()->CGFloat
    {
        return self.frame.size.width
    }
    func height()-> CGFloat
    {
        return self.frame.size.height
    }
    
    func setX(x: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.origin.x = x
        self.frame = rect
    }
    
    func setRight(right: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.origin.x = right - rect.size.width
        self.frame = rect
    }
    
    func setY(y: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.origin.y = y
        self.frame = rect
    }
    
    func setBottom(bottom: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.origin.y = bottom - rect.size.height
        self.frame = rect
    }
    
    func setWidth(width: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.size.width = width
        self.frame = rect
    }
    
    func setHeight(height: CGFloat)
    {
        var rect:CGRect = self.frame
        rect.size.height = height
        self.frame = rect
    }
    /// 显示一个Alert
    class func showAlertView(title:String,message:String)
    {
        let alert = UIAlertView()
        alert.title = title
        alert.message = message
        alert.addButtonWithTitle("好")
        alert.show()

    }
   
}
~~~

# 代码片段

# 1、文件处理
~~~ swift
class FileUtility: NSObject {
   
    class func cachePath(fileName:String)->String
    {
        var arr =  NSSearchPathForDirectoriesInDomains(.CachesDirectory, .UserDomainMask, true)
        let path = arr[0] 
        return "\(path)/\(fileName)"
    }
    
    
    class func imageCacheToPath(path:String,image:NSData)->Bool
    {
       return image.writeToFile(path, atomically: true)
    }
    
    class func imageDataFromPath(path:String)->AnyObject
    {
        let exist = NSFileManager.defaultManager().fileExistsAtPath(path)
        if exist
        {
            //var urlStr = NSURL.fileURLWithPath(path)
            _ = NSData(contentsOfFile: path);
            //var img:UIImage? = UIImage(data:data!)
            //return img ?? NSNull()
            let img = UIImage(contentsOfFile: path)
            
            let url:NSURL? = NSURL.fileURLWithPath(path)
            let dd = NSFileManager.defaultManager().contentsAtPath(url!.path!)
            _ = UIImage(data:dd!)
            
            if img != nil {
                return img!
            } else {
                return NSNull()
            }
        }
        
        return NSNull()
    }
    
}
~~~

to be continue ...


