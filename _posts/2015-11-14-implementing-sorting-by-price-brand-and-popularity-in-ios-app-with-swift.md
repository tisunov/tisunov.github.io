---
layout: post
title: Implementing sorting by product price and brand in iOS app with Swift
categories: []
tags: []
published: True

---

I don't know how our clients managed to get by without sorting in our iOS app, but it's finally coming.

I want to allow users to sort products in UICollectionView by price, unit price, popularity (how often products are bought), brand name (group brand products together).

## Implement SortViewController

Let's begin by creating new view controller in storyboard with navigation bar to show title and table view underneath.

{:.center}
[![SortViewController](/assets/sorting/sort-view-controller.jpg){:width="450px"}](/assets/sorting/sort-view-controller.jpg)
*SortViewController in storyboard*

Table view will contain only 4 cells, and I want view controller to pop from the bottom so user can easily reach table view cells. That's why I set custom height. 

Setup constraints to stretch navigation bar horizontally and fill up rest of the view with table view. Don't forget to set Storyboard ID (as I just did) and set view controller _Custom Class_ to SortViewController which we'll create later.

Finally create table view prototype cell with _SortCell_ reuse identifier.

I will use enum to strongly type sort option currently in effect. Proceed by creating _SortBy_ enum with 4 cases:
 - Sort by price
 - Sort by unit price
 - Sort by popularity
 - Sort by brand

{% highlight swift %}
@objc enum SortBy: Int {
    case Price = 0
    case UnitPrice
    case Popularity
    case Brand
    
    init?(index: Int) {
        switch index {
        case 0: self = .Price
        case 1: self = .UnitPrice
        case 2: self = .Popularity
        case 3: self = .Brand
        default:
            return nil
        }
    }
    
    var description: String? {
        get {
            switch self {
            case .Price: return "Цене"
            case .UnitPrice: return "Цене за единицу"
            case .Popularity: return "Популярности"
            case .Brand: return "Брэнд"
            }
        }
    }
}
{% endhighlight %}

I added failable initializer to enum to convert cell index into _SortBy_ enum case. Then I added description property to get sort option title for table view cell. Most of the application is built with Objective C, I have to mark enum with _@objc_ it accessible from Objective C.

Let's create _SortViewController.swift_ and implement _UITableViewDataSource_:

{% highlight swift %}
class SortViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {
    private var selectedIndexPath: NSIndexPath?
    @IBOutlet var tableView: UITableView!

    // MARK: - UITableViewDataSource
    func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 4
    }
    
    func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCellWithIdentifier("SortCell", forIndexPath: indexPath)
        
        if let sortOption = SortBy(index: indexPath.row) {
            cell.textLabel?.text = sortOption.description
            if sortOption == SortBy.Popularity {
                cell.accessoryType = .Checkmark
                selectedIndexPath = indexPath
            }
        }
        
        return cell
    }
}
{% endhighlight %}

To show which sort option is in effect right now I use table view cell checkmark accessory.

Next, we need to tell anyone interested when user changed sort order, I chose to use protocol:
{% highlight swift %}

protocol SortViewControllerDelegate: class {
    func sortBy(option: SortBy)
}

{% endhighlight %}

Add delegate property to _SortViewController_:
{% highlight swift %}
weak var delegate: SortViewControllerDelegate?

{% endhighlight %}

and implement _UITableViewDelegate_ by calling delegate with selected sort option:

{% highlight swift %}
// MARK: UITableViewDelegate

func tableView(tableView: UITableView, didDeselectRowAtIndexPath indexPath: NSIndexPath) {
    let cell = tableView.cellForRowAtIndexPath(indexPath)
    cell?.accessoryType = .Checkmark

    if let sortOption = SortBy(index: indexPath.row) {
        delegate?.sortBy(sortOption)
    }

    if let path = selectedIndexPath {
        let cell = tableView.cellForRowAtIndexPath(path)
        cell?.accessoryType = .None
    }
    
    dismissViewControllerAnimated(true, completion: nil)
}

{% endhighlight %}


## Integrate SortViewController
Time to hook up _SortViewController_ with rest of the app. I want to enable product sorting in individual aisles. Each aisle is displayed in UICollectionView as a grid of products, with header on top. I have custom class to represent aisle header _GRProductCollectionHeaderView_ with a button which I'll use to show _SortViewController_.

I want to keep _GRShelfViewController_ focused, and instead of implementing additional protocols inside I will create Swift extension for sorting:

{% highlight swift %}

extension GRShelfViewController: SortViewControllerDelegate {
 
    // MARK: GRProductCollectionHeaderViewDelegate
    public override func displaySectionIndex(index: UInt) {
        let controller = UIStoryboard(name: "Store", bundle: nil).instantiateViewControllerWithIdentifier("SortViewController") as! SortViewController
        controller.delegate = self
        controller.transition = enableInteractiveTransitionFor(controller, behindViewScale: 1.0, alpha: 0.7)
        controller.view.bounds = CGRect(x: 0, y: 0, width: view.bounds.width, height: 200)
        
        dispatch_async(dispatch_get_main_queue(), {
            self.dismissViewControllerAnimated(true, completion:nil)
        });
    }

    // MARK: SortViewControllerDelegate
    func sortBy(option: SortBy) {
        
    }    
}
{% endhighlight %}

I found strange variable delay between call to `dismissViewControllerAnimated:completion` and actual dimissal. Delayed dismissal until next run loop and all worked fine.

I find it limiting to always reach for close button in top left or right corner of the view to dismiss view controller. Thankfully I found [ZFModalTransitionAnimator](https://github.com/zoonooz/ZFDragableModalTransition) which implements custom transition animation and allows user to drag view to dismiss. Create `ZFModalTransitionAnimator` inside `enableInteractiveTransitionFor` and pass it to SortViewController, so it can assigned it's UITableView so ZFModalTransitionAnimator will dismiss view controller on swipe down.

{% highlight objective-c %}

(ZFModalTransitionAnimator *)enableInteractiveTransitionFor:(UIViewController *)controller behindViewScale:(float)scale alpha:(float)alpha {
    ZFModalTransitionAnimator *transitionAnimator = [[ZFModalTransitionAnimator alloc] initWithModalViewController:controller];
    transitionAnimator.dragable = YES;
    transitionAnimator.direction = ZFModalTransitonDirectionBottom;
    transitionAnimator.bounces = NO;
    transitionAnimator.spring = NO;
    transitionAnimator.behindViewScale = scale;
    transitionAnimator.behindViewAlpha = alpha;

    // set transition delegate of modal view controller to our object
    controller.transitioningDelegate = transitionAnimator;
    controller.modalPresentationStyle = UIModalPresentationCustom;

    return transitionAnimator;
}

{% endhighlight %}

{:.center}
![Sort options](/assets/sorting/sort-options.gif)
*Sort Options Popup*

## Sorting by price

I think sorting belongs to aisle so I will implement it in my `GRShelf` class:

{% highlight objective-c %}
- (NSArray *)sortProducts:(NSArray *)products by:(NSInteger)option {
    NSArray *productsOrdered = nil;
    switch (option) {
        case SortByPrice:
            productsOrdered = [products sortedArrayUsingDescriptors:@[[NSSortDescriptor sortDescriptorWithKey:@"price" ascending:YES]]];
            break;
        default:
            break;
    }
    
    return productsOrdered;
}
{% endhighlight %}

I had to use raw enum value and NSInteger in Objective C code, because of circular dependency. The reason is I included `GRShelf.h` in Swift bridging header and have to include Swift header into `GRShelf.h` in order to use `SortBy` enum.

{% highlight swift %}

// MARK: SortViewControllerDelegate
func sortBy(option: SortBy) {
    products = shelf.sortProducts(products, by:option.rawValue)
}

{% endhighlight %}

## Animate UICollectionView cell reordering

After sorting products in the model I want to animate UICollectionView cell reordering. I enumerate all products in ordered array, find which row index given product had under old order and tell UICollectionView to `moveItemAtIndexPath:toIndexPath` from old index path to new index path.

{% highlight swift %}

// MARK: SortViewControllerDelegate
func sortBy(option: SortBy) {
    let productsOrdered = shelf.sortProducts(products, by:option.rawValue)
    
    collectionView?.performBatchUpdates({ () -> Void in
        for (newIndex, product) in productsOrdered.enumerate() {
            let oldIndex = self.products.indexOf({ $0 === product })
            let fromIndexPath = NSIndexPath(forRow: oldIndex!, inSection: 0)
            let toIndexPath = NSIndexPath(forRow: newIndex, inSection: 0)
            
            self.collectionView?.moveItemAtIndexPath(fromIndexPath, toIndexPath: toIndexPath)
        }
    }, completion: { finished in
        self.products = productsOrdered
    })
}
{% endhighlight %}

Sorting kinda works now, but since I use variable height for each row of 3 cells, I have to recalculate maximum height for each row of cells. I do calculations in `GRShelfViewController.calculateCellSizesForProductRows` method, call it after moving cells but before finishing batch updates:

{% highlight swift %}
collectionView?.performBatchUpdates({ () -> Void in
    ...

    self.products = productsOrdered
    self.calculateCellSizesForProductRows()
}, completion: nil)
{% endhighlight %}

## Sort by product unit price

I want to give people ability to compare similar product prices even if they have different weights, like candy 250g for $5 and 320g for $7, first one is cheaper .

So _unit_ is the minimum indivisible quantity of product which. I add new `GRProduct.fractionalPrice`:

{% highlight objective-c %}

- (float)fractionalPrice {
    if (self.size == 0) return self.price;
    
    float result = self.price;
    switch (self.unit) {
        case GRUnitPiece:
            result = self.price / self.size;
            break;
        case GRUnitLiter:
        case GRUnitKg:
            result = self.price / self.size / 1000;
            break;
    }

    return result;
}
{% endhighlight %}

And extend `sortProducts:by:` method with new case: 

{% highlight objective-c %}

case SortByUnitPrice:
    productsOrdered = [products sortedArrayUsingDescriptors:@[[NSSortDescriptor sortDescriptorWithKey:@"fractionalPrice" ascending:YES]]];
    break;

{% endhighlight %}

## Remember and restore aisle sort order

I want to keep sorting functionality close together until it will get in the way. Let's add computed property to _GRShelfViewController_ extension to store sort order:

{% highlight swift %}
@objc var sortOrder: SortBy {
    get {
        let key = "shelf.\(shelf.uid).order"
        
        return SortBy(rawValue: NSUserDefaults.standardUserDefaults().integerForKey(key))!
    }
    set {
        NSUserDefaults.standardUserDefaults().setInteger(newValue.rawValue, forKey: "shelf.\(shelf.uid).order")
    }
}
{% endhighlight %}

Even if there is no stored ordering for the aisle, `sortOrder` will return `SortBy.Popularity`, which exactly how server orders by default. I will skip sorting by Popularity during initial load of UICollectionView:

{% highlight objective-c %}
- (void)loadContent {
    ...
    
    if (self.sortOrder != SortByPopularity) {
        self.products = [self.shelf sortProducts:products by:self.sortOrder];
    }
    else {
        self.products = products;
    }
   
    ...
}
{% endhighlight %}

Let's see what we've got with production DB:

{:.center}
![Groser Sorting](/assets/sorting/final.gif)
*Sorting*

Thanks for reading! As you may have noticed that's just stream of consciousness as I develop new features for our app. 

I feel like there are a lot of tutorials, which cover only abstract features of the language or Cocoa frameworks. I wanted to show how all of this applicable to real world development where you switch roles from backend to front end, fix bugs and debug weird behaviour.
