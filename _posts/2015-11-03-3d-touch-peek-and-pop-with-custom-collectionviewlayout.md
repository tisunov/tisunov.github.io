---
layout: single
title: 3D Touch Peek and Pop with custom CollectionViewLayout
categories: []
tags: []
published: True

---

I followed this [tutorial](http://www.the-nerd.be/2015/10/06/3d-touch-peek-and-pop-tutorial/) while learning how to add 3D Touch Peek & Pop to my app. And it works fine if you have standard collection view layout.

In case you use custom collection layout, you'll see inconsistencies between detection of 3D Touch on collection cells, and cell.frame assigned to _previewingContext.sourceRect_ will be incorrect.

What I found, is that you can get layout attributes for every element in collection view bounds, iterate over it, do point and rect conversion between coordinate systems and get correct frames for detection of 3D Touched cell.

How to detect which cell was 3D Touched in collection view with custom layout:
{% highlight swift %}

public func previewingContext(previewingContext: UIViewControllerPreviewing, viewControllerForLocation location: CGPoint) -> UIViewController? {
    
    var controller: UIViewController?
    
    guard let layoutAttributes = collectionView!.collectionViewLayout.layoutAttributesForElementsInRect(collectionView!.bounds) else { return nil }
    
    for attributes in layoutAttributes {
        let point = collectionView!.convertPoint(location, fromView: collectionView!.superview)
        
        if attributes.representedElementKind == nil && CGRectContainsPoint(attributes.frame, point) {
            if #available(iOS 9.0, *) {
                previewingContext.sourceRect = collectionView!.convertRect(attributes.frame, toView: collectionView!.superview)
                
                controller = UIStoryboard(name: "Main", bundle: nil).instantiateViewControllerWithIdentifier("ProductNavViewController")
                
                setupProductController(controller as? UINavigationController, indexPath: attributes.indexPath)
            }
            break
        }
    }
    
    return controller
}

{% endhighlight %}

I convert location passed by the system to collection view coordinate space. Then I find which cell's rect contains passed location and use that rect after converting it to app window coordinate space in assignment to _previewingContext.sourceRect_.

Hope it will save you 30-40 minutes :-)