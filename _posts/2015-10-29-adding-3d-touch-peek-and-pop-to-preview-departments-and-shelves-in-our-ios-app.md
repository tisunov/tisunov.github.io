---
layout: single
title: Adding 3D Touch Peek and Pop to preview departments and shelves in our iOS app
categories: []
tags: []
published: True

---

I want to make our [Groser app](https://itunes.apple.com/en/app/groser/id915288141) awesome by adding 3D Touch Peek and Pop to it on new iPhones. Let's start by adding preview feature for store department when user applies force on "More" button.

{:.center}
![Store](/assets/3d-touch/store-1.png){:height="500px"}
![Department](/assets/3d-touch/department-1.png){:height="500px"}
*We want to preview department on the second screen when user uses 3D Touch Peek on the first screen*

I found a [tutorial](http://www.the-nerd.be/2015/10/06/3d-touch-peek-and-pop-tutorial/) in Swift and will follow it to integrate 3D Touch in my app.

First, I create Swift extension for _GRStoreViewController_ Objective C view controller class, import `Groser-Swift.h` into `GRStoreViewController.h`, and register for previewing department on button press. Our "More" button is inside CollectionView header, so I register inside implementation of `collectionView:viewForSupplementaryElementOfKind:atIndexPath:`. Before registering we check controller's _traitCollection.forceTouchCapability_ for _UIForceTouchCapabilityAvailable_.

**GRStoreViewController.m**
{% highlight objective-c %}
#import "Groser-Swift.h"

- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath
{
    GRProductCollectionHeaderView *headerView = [collectionView dequeueReusableSupplementaryViewOfKind:UICollectionElementKindSectionHeader withReuseIdentifier:@"SectionHeader" forIndexPath:indexPath];

    if (kind == UICollectionElementKindSectionHeader) {
      if (self.traitCollection.forceTouchCapability == UIForceTouchCapabilityAvailable) {
          [self registerForPreviewingWithDelegate:self sourceView:headerView.moreButton];
      }
    }

    return headerView;
}
{% endhighlight %}

**StoreViewControllerExtension.swift**
{% highlight swift %}

extension GRStoreViewController : UIViewControllerPreviewingDelegate {
    public func previewingContext(previewingContext: UIViewControllerPreviewing, viewControllerForLocation location: CGPoint) -> UIViewController? {
        return nil
    }
    
    public func previewingContext(previewingContext: UIViewControllerPreviewing, commitViewController viewControllerToCommit: UIViewController) {
        
    }   
}

{% endhighlight %}

Next, we'll implement _UIViewControllerPreviewingDelegate_ method starting with `previewingContext:viewControllerForLocation`. To preview department we need to know which one to show, we'll use `location` argument to find indexPath of the section which was touched. Each section index maps directly to department index in store. _UICollectionView_ can only find _indexPath_ from point for collection cells, and we know that there are cells under collection section header, so we offset touch point **y** by **50px** and get _indexPath_ for the cell under section header. Initialize _DepartmentsPagerController_ with store and current department.

{% highlight swift %}

public func previewingContext(previewingContext: UIViewControllerPreviewing, viewControllerForLocation location: CGPoint) -> UIViewController? {
    
    let cellLocation = CGPoint(x: location.x, y: location.y + 50)
    guard let indexPath = collectionView?.indexPathForItemAtPoint(cellLocation) else { return nil }
    
    let store = GRGroser.shared().currentStore
    let department = store.departments[indexPath.section] as! GRDepartment
    let controller = DepartmentsPagerController.init(store: store, department: department)
    
    return controller
}

{% endhighlight %}

Configure previewing controller _preferredContentSize_ and set _previewingContext.sourceRect_ to collection view header frame, so everything but header will be blured when previewing department. Let' iOS decide the width, and set the height to 200px less than Store view height.

{% highlight swift %}
controller.preferredContentSize = CGSize(width: 0.0, height: view.frame.height - 200.0)

// Get header view frame
if #available(iOS 9.0, *) {
    let headerView = collectionView?.supplementaryViewForElementKind(UICollectionElementKindSectionHeader, atIndexPath: NSIndexPath.init(index: indexPath.section))
    
    previewingContext.sourceRect = headerView!.frame
}

{% endhighlight %}

## Popping into Department

Implement `previewingContext:commitViewController` to push previewed view controller into _UINavigationController_ stack.

{% highlight swift %}
public func previewingContext(previewingContext: UIViewControllerPreviewing, commitViewController viewControllerToCommit: UIViewController) {
    if #available(iOS 8.0, *) {
        showViewController(viewControllerToCommit, sender: self)
    }    
}   
{% endhighlight %}

That's it! Let's check how it looks on device. Shit, I upgraded to iOS 9.1 and now my Xcode 7.0.1 can't build app for iOS 9.1. Almost 45 minutes later after downloading and installing Xcode 7.1...

{:.center}
![Almost works](/assets/3d-touch/almost.png){:height="500px"}
*3D Touch Peek*

It works, but only on the first collection view section. It was not the smartest move on my part to register preview delegate for each section header when only 2 at most will be visible on screen. I decided to register preview delegate for the _view_ and find which section header was force touched by checking which section header frame contains touch location.

**GRStoreViewController.m**
{% highlight objective-c %}

- (void)viewDidLoad {
    [super viewDidLoad];

    if (self.traitCollection.forceTouchCapability == UIForceTouchCapabilityAvailable) {
        [self registerForPreviewingWithDelegate:self sourceView:self.view];
    }
}

{% endhighlight %}

**StoreViewControllerExtension.swift**
{% highlight swift %}

extension GRStoreViewController : UIViewControllerPreviewingDelegate {
    public func previewingContext(previewingContext: UIViewControllerPreviewing, viewControllerForLocation location: CGPoint) -> UIViewController? {
        
        var controller: UIViewController?
        
        // We want to detect force touch on collection header
        guard let layoutAttributes = collectionView!.collectionViewLayout.layoutAttributesForElementsInRect(collectionView!.bounds) else { return nil }
        
        for attributes in layoutAttributes {

            let point = collectionView!.convertPoint(location, fromView: collectionView!.superview)
            
            if attributes.representedElementKind == UICollectionElementKindSectionHeader && CGRectContainsPoint(attributes.frame, point) {
                if #available(iOS 9.0, *) {
                    let headerView = collectionView!.supplementaryViewForElementKind(UICollectionElementKindSectionHeader, atIndexPath: attributes.indexPath) as! GRProductCollectionHeaderView

                    // Ignore if location inside Store section header and above department header
                    if headerView.isMemberOfClass(GRStoreHeader) && CGRectContainsPoint(CGRect(x:attributes.frame.origin.x, y:attributes.frame.origin.y, width:attributes.frame.width, height:attributes.frame.height - 80.0), point) {
                        break
                    }
                    
                    previewingContext.sourceRect = collectionView!.convertRect(attributes.frame, toView: collectionView!.superview)
                    controller = previewDepartmentWithIndex(Int(headerView.sectionIndex))
                }
                break
            }
        }
        
        return controller
    }
    
    public func previewingContext(previewingContext: UIViewControllerPreviewing, commitViewController viewControllerToCommit: UIViewController) {
        if #available(iOS 8.0, *) {
            showViewController(viewControllerToCommit, sender: self)
        }
    }
    
    func previewDepartmentWithIndex(index: Int) -> UIViewController {
        let store = GRGroser.shared().currentStore
        let department = store.departments[index] as! GRDepartment
        
        let controller = DepartmentsPagerController.init(store: store, department: department)
        controller.preferredContentSize = CGSize(width: 0.0, height: 400.0)
        
        return controller
    }
}

{% endhighlight %}

{:.center}
![Department Peek](/assets/3d-touch/department-peek.png){:height="500px"}
![Shelf Peek](/assets/3d-touch/shelf-peek.png){:height="500px"}
*3D Touch Peek & Pop for Departments and Shelves inside Departments*


Adding Peek & Pop was really easy, most of the time was spent on figuring out where user tapped in _UICollectionView_. I thought about adding Peek & Pop for individual product cells, not so sure why it would be useful though. Checking available delivery windows and delivery zone definitely will be useful, so I'll add them too.