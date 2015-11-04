---
layout: post
title: Adding instructional coach marks to iOS app with Swift
categories: []
tags: []
published: True

---

I generally don't like onboarding screens, which no one actually reads, and they just stand in the way of using the app. User already found, downloaded and launched your app, why would make 4 screen of obstacles to your app you spent so much time developing?

What I like is the idea of teaching the user about non obvious features of the app to make him productive in context of application. And user should be able to clearly see the _Skip_ button to just explore on his one.

With that in mind I chose [Instructions](https://github.com/ephread/Instructions) Swift library that shows cute coach marks and has in my view handy architecture inside.

{:.center}
![Instructions](/assets/coach-marks/Instructions.jpg){:height="400px"}
*Instructions*


## Adapting Instructions to support iOS 7.0
We have about 12% of users on iOS 7 and since _Instructions_ was built with iOS 8 in mind, it means 12% users won't be able to enjoy new features, and we will probably lose their business. I decided to refator _Instructions_ to support iOS 7 and upwards.

Main obstactle for iOS 7 support is usage of _UIBlurEffectStyle_ and _UIVisualEffectView_ to blur background while showing coach mark. 

We'll use darkened background instead of blur on iOS 7 and blur on iOS 8 and up. I declared attributes specifying _UIBlurEffectStyle_ and _UIVisualEffectView_ as _AnyObject?_, on assignment will convert enum value to NSNumber and will cast them to enum depending on iOS version. Swift has very handy iOS version checking _@available(iOS 8.0, *)_ which I'll use to mark parts of code as unavailable for iOS 7.

{% highlight swift %}
internal class OverlayView: UIView {

    var blurEffectStyle: AnyObject? { // UIBlurEffectStyle
        didSet {
            if #available(iOS 8, *) {
                let blurStyle = self.blurEffectStyle as? NSNumber
                let oldBlurStyle = oldValue as? NSNumber
                if blurStyle != oldBlurStyle {
                    self.destroyBlurView()
                    self.createBlurView()
                }
            }
        }
    }

    private func createBlurView() {
        if self.blurEffectStyle == nil { return }
        
        if #available(iOS 8.0, *) {
            let blurEffect = UIBlurEffect(style: UIBlurEffectStyle(rawValue: (self.blurEffectStyle as! NSNumber).integerValue)!)
            
            let blurView = UIVisualEffectView(effect:blurEffect)
            blurView.translatesAutoresizingMaskIntoConstraints = false
            
            self.blurEffectView = blurView
            self.addSubview(blurView)
        }
    }

}
{% endhighlight %}

Next, _Instructions_ also uses _UIImage.imageNamed:inBundle:compatibleWithTraitCollection:_ initilizer available only on iOS 8 and up.

**CoachMarkBodyDefaultView.swift**
{% highlight swift %}

private let backgroundImage = UIImage(named: "background", inBundle: NSBundle(forClass: CoachMarkBodyDefaultView.self), compatibleWithTraitCollection: nil)

private let highlightedBackgroundImage = UIImage(named: "background-highlighted", inBundle: NSBundle(forClass: CoachMarkBodyDefaultView.self), compatibleWithTraitCollection: nil)

{% endhighlight %}

I refactored by moving initialization to default initializer and loading UIImage from file path for images in current bundle:

**CoachMarkBodyDefaultView.swift**
{% highlight swift %}
override public init (frame: CGRect) {
    backgroundImage = UIImage(named: "background")
    
    backgroundImageView = UIImageView(image: backgroundImage)
    
    highlightedBackgroundImage = UIImage(named: "background-highlighted")
    
    super.init(frame: frame)

    self.setupInnerViewHierarchy()
}

{% endhighlight %}

I did the same for _CoachMarkArrowDefaultView.swift_:
{% highlight swift %}
public init(orientation: CoachMarkArrowOrientation) {
    if orientation == .Top {
        super.init(image: UIImage(named: "arrow-top"), highlightedImage: UIImage(named: "arrow-top-highlighted"))
    } else {
        super.init(image: UIImage(named: "arrow-bottom"), highlightedImage: UIImage(named: "arrow-bottom-highlighted"))
    }
}

{% endhighlight %}

Second problem is _Instructions_ is a **Swift dynamic library supported only after iOS 8**. I tried to make static, but then Cocoapods complained that I can't use Swift static library.

I have to copy _Instructions_ code and assets into my project and probably manually merge future library versions until we drop iOS 7.0 support.

## Adding Store controller coach marks

I want to show non standard UI component used for quick navigation to departments and aisles. I decided to follow so called _Protocol Oriented Development_ and instead of adding new code to existing view controllers create Swift extensions. 

First, I created generic UIViewController extension to add coach marks capability to every UIViewController descendant. It stores _CoachMarksController_ instance in associated object, setups CoachMarksController, and Skip control to skip instructions all together:

**UIViewControllerInstructions.swift**
{% highlight swift %}

extension UIViewController {
    private struct AssociatedKeys {
        static var CoachMarksControllerName = "CoachMarksController"
    }
    
    private var coachMarksController: CoachMarksController?{
        get{
            return objc_getAssociatedObject(self, &AssociatedKeys.CoachMarksControllerName) as? CoachMarksController
        }
        set{
            if let newValue = newValue{
                willChangeValueForKey(AssociatedKeys.CoachMarksControllerName)
                objc_setAssociatedObject(self, &AssociatedKeys.CoachMarksControllerName, newValue as CoachMarksController, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
                didChangeValueForKey(AssociatedKeys.CoachMarksControllerName)
            }
        }
    }
    
    func setupCoachMarks() {
        coachMarksController = CoachMarksController()
        coachMarksController?.allowOverlayTap = true
        coachMarksController?.datasource = self as? CoachMarksControllerDataSource
        coachMarksController?.overlayBackgroundColor = UIColor(red: 0.5254, green: 0.5253, blue: 0.5253, alpha: 0.6)
        
        let skipView = CoachMarkSkipDefaultView()
        skipView.setTitle("Пропустить", forState: .Normal)
        skipView.setTitleColor(UIColor.whiteColor(), forState: .Normal)
        skipView.backgroundColor = UIColor ( red: 0.4266, green: 0.4758, blue: 0.4803, alpha: 0.9 )
        
        coachMarksController?.skipView = skipView
    }
    
    func startCoaching() {
        coachMarksController?.startOn(self)
    }
    
    public func coachMarksController(coachMarksController: CoachMarksController, constraintsForSkipView skipView: UIView, inParentView parentView: UIView) -> [NSLayoutConstraint]? {
        
        var constraints: [NSLayoutConstraint] = []
        
        // Stretch horizontally
        constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("H:|[skipView]|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: ["skipView": skipView]))
        
        // Pin to bottom
        constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("V:[skipView(==50)]|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: ["skipView": skipView]))
        
        return constraints
    }
}
{% endhighlight %}

Next, I want to highlight portions of the UI on iPhone 6s/6s Plus that user can 3D Touch to preview departments and products without leaving main store screen:

**StoreViewControllerInstructions.swift**
{% highlight swift %}
extension GRStoreViewController: CoachMarksControllerDataSource {
    
    // MARK: - Protocol Conformance | CoachMarksControllerDataSource
    public func numberOfCoachMarksForCoachMarksController(coachMarksController: CoachMarksController) -> Int {
        return hasForceTouch() ? 3 : 1
    }
    
    public func coachMarksController(coachMarksController: CoachMarksController, coachMarksForIndex index: Int) -> CoachMark {
        
        // This will create cutout path matching perfectly the given view.
        // No padding!
        let flatBezierPathBlock = { (frame: CGRect) -> UIBezierPath in
            return UIBezierPath(rect: frame)
        }
        
        var coachMark : CoachMark
        
        switch(index) {
        case 0:
            coachMark = coachMarksController.coachMarkForView(departmentsBar, pointOfInterest: self.departmentsBar?.center, bezierPathBlock: flatBezierPathBlock)
        case 1:
            coachMark = coachMarksController.coachMarkForView(storeHeaderView?.coachMarkTarget, pointOfInterest: self.storeHeaderView?.coachMarkTarget.center, bezierPathBlock: flatBezierPathBlock)
        case 2:
            let cell = firstCell()
            coachMark = cell != nil ? coachMarksController.coachMarkForView(cell, pointOfInterest: cell!.center, bezierPathBlock: flatBezierPathBlock) : coachMarksController.coachMarkForView()
            
        default:
            coachMark = coachMarksController.coachMarkForView()
        }
        
        coachMark.gapBetweenCoachMarkAndCutoutPath = 6.0
        
        return coachMark
    }
    
    func firstCell() -> UIView? {
        guard let layoutAttributes = collectionView!.collectionViewLayout.layoutAttributesForElementsInRect(collectionView!.bounds) else { return nil }
        
        // Find first cell in collection view
        if let i = layoutAttributes.indexOf({$0.representedElementKind == nil}) {
            let attr = layoutAttributes[i]
            return collectionView?.cellForItemAtIndexPath(attr.indexPath)
        }
        
        return nil
    }
    
    func hasForceTouch() -> Bool {
        if #available(iOS 9.0, *) {
            return traitCollection.forceTouchCapability == .Available
        }
        else {
            return false
        }
    }
    
    public func coachMarksController(coachMarksController: CoachMarksController, coachMarkViewsForIndex index: Int, coachMark: CoachMark) -> (bodyView: CoachMarkBodyView, arrowView: CoachMarkArrowView?) {
        
        let coachViews = coachMarksController.defaultCoachViewsWithArrow(true, arrowOrientation: coachMark.arrowOrientation)
        
        switch(index) {
        case 0:
            coachViews.bodyView.hintLabel.text = "Это меню быстрого перехода к отделам или рядам"
            coachViews.bodyView.nextLabel.text = "Оk!"
        case 1:
            coachViews.bodyView.hintLabel.text = "С силой надавите на заголовок отдела, чтобы быстро посмотреть отдел"
            coachViews.bodyView.nextLabel.text = "Оk!"
        case 2:
            coachViews.bodyView.hintLabel.text = "С силой надавите на продукт, чтобы быстро его посмотреть"
            coachViews.bodyView.nextLabel.text = "Оk!"
            
        default: break
        }
        
        return (bodyView: coachViews.bodyView, arrowView: coachViews.arrowView)
    }
}
{% endhighlight %}

Finally, I call _setupCoachNarks_ in _viewDidLoad_ and _startCoaching_ after store infromation has been downloaded from server and collection view was populated:

**GRStoreViewController.m**
{% highlight objective-c %}

- (void)viewDidLoad {
    [super viewDidLoad];

    [self setupCoachMarks];
}

- (void)loadContent {
    @weakify(self)
    [[GRGroser shared] loadStores].then(^(NSArray *stores) {
        @strongify(self)

        [self startCoaching];
    });
}

{% endhighlight %}

Here is how it looks with everything integrated:

{:.center}
![Store Menu Instruction](/assets/coach-marks/store-menu.jpg){:height="400px"}
![Store Department Instruction](/assets/coach-marks/store-department.jpg){:height="400px"}
![Store Product Instruction](/assets/coach-marks/store-product.jpg){:height="400px"}
*Store Instructions*

## Department coach marks

Recently I ditched drop down menu for navigation in departments and aisles and now use custom UIPageControl with scrolling list of all departments to allow user to swipe between controllers. I want to highlight that:

**DepartmentsPagerInstructions.swift**
{% highlight swift %}
{% endhighlight %}

I show coach mark as soon as view appears, but _viewDidAppear_ gets called during 3D Touch Peek and gets really weird. I added property to disable coach mark during 3D Touch Peek.

{:.center}
![Department Instructions](/assets/coach-marks/department.jpg){:height="400px"}
*Department Instructions*

## Deleting and adding notes to order items

Some people can't figure out how to delete order item from their cart. I'll show instruction one time after they visit their cart with at least 1 order item.

**CartControllerInstructions.swift**
{% highlight swift %}
extension GRCartViewController: CoachMarksControllerDataSource {
    // MARK: - Protocol Conformance | CoachMarksControllerDataSource
    public func numberOfCoachMarksForCoachMarksController(coachMarksController: CoachMarksController) -> Int {
        return 2
    }
    
    public func coachMarksController(coachMarksController: CoachMarksController, coachMarksForIndex index: Int) -> CoachMark {
        
        // This will create cutout path matching perfectly the given view.
        // No padding!
        let flatBezierPathBlock = { (frame: CGRect) -> UIBezierPath in
            return UIBezierPath(rect: frame)
        }
        
        var coachMark: CoachMark
        let cell = tableViewCellFor(index)!
        
        switch(index) {
        case 0:
            coachMark = coachMarksController.coachMarkForView(cell, pointOfInterest: CGPoint(x: cell.frame.width - 50.0, y: cell.center.y), bezierPathBlock: flatBezierPathBlock)
        case 1:
            coachMark = coachMarksController.coachMarkForView(cell, pointOfInterest: CGPoint(x: 50.0, y: cell.center.y), bezierPathBlock: flatBezierPathBlock)
        default:
            coachMark = coachMarksController.coachMarkForView()
        }
        
        coachMark.gapBetweenCoachMarkAndCutoutPath = 6.0
        
        return coachMark
    }
    
    func tableViewCellFor(coachMarkIndex: Int) -> UIView? {
        let indexPath = tableView?.numberOfRowsInSection(0) > 1 ? NSIndexPath(forRow: coachMarkIndex, inSection: 0) : NSIndexPath(forRow: 0, inSection: 0)
        
        return tableView?.cellForRowAtIndexPath(indexPath)
    }
    
    public func coachMarksController(coachMarksController: CoachMarksController, coachMarkViewsForIndex index: Int, coachMark: CoachMark) -> (bodyView: CoachMarkBodyView, arrowView: CoachMarkArrowView?) {
        
        let coachViews = coachMarksController.defaultCoachViewsWithArrow(true, arrowOrientation: coachMark.arrowOrientation)
        
        switch(index) {
        case 0:
            coachViews.bodyView.hintLabel.text = "Потяните товар справа налево, чтобы удалить"
            coachViews.bodyView.nextLabel.text = "Оk!"
        case 1:
            coachViews.bodyView.hintLabel.text = "Потяните товар слева направо, чтобы добавить инструкцию к нему"
            coachViews.bodyView.nextLabel.text = "Оk!"
            
        default: break
        }
        
        return (bodyView: coachViews.bodyView, arrowView: coachViews.arrowView)
    }
}
{% endhighlight %}

I'm using custom coach mark _CoachMarkBodyView_ implementation with colorful button to be consistent with the rest of UI.

{:.center}
![Cart Delete Instruction](/assets/coach-marks/cart-delete.jpg){:height="400px"}
![Cart Note Instruction](/assets/coach-marks/cart-note.jpg){:height="400px"}
![Search Emoji Instruction](/assets/coach-marks/search-emoji.jpg){:height="400px"}
*Cart & Search Instructions*

## Don't annoy user

Let's be nice to our users and don't annoy them with repetitive instructions. If user skips instructions or seen at least one coach mark for the given view controller, we will not show instructions next time.

I created _InstructionScreens_ enum and default initializer to map view controllers to enum values:

**UserInstructions.swift**
{% highlight swift %}

enum InstructionScreens {
    case Store
    case Department
    case Cart
    case Search
    
    init?(controller: UIViewController) {
        switch controller {
        case is GRStoreViewController: self = .Store
        case is DepartmentsPagerController: self = .Department
        case is GRCartViewController: self = .Cart
        case is GRProductSearchViewController: self = .Search
        default:
            return nil
        }
    }
}

{% endhighlight %}

Then, I write and read _NSUserDefaults_ to track which screen to coach user about or not:

**UserInstructions.swift**
{% highlight swift %}

extension GRUser {
    func finishInstructionsFor(screen: InstructionScreens?) {
        if let instructionScreen = screen {
            NSUserDefaults.standardUserDefaults().setBool(true, forKey: "user.instructions.\(instructionScreen)")
        }
    }
    
    func shouldCoachFor(screen: InstructionScreens?) -> Bool {
        if let instructionScreen = screen {
            return !NSUserDefaults.standardUserDefaults().boolForKey("user.instructions.\(instructionScreen)")
        }
        else {
            return false
        }
    }
}

{% endhighlight %}

Last, in _startCoaching_ I check if current view controller coaching was completed and save completion in _didFinishShowingFromCoachMarksController_, don't forget to set _coachMarksViewController.delegate_:

**UIViewControllerInstructions.swift**
{% highlight swift %}

extension UIViewController: CoachMarksControllerDelegate {
    func setupCoachMarks() {
        coachMarksController?.delegate = self
    }

    func startCoaching() {
        if GRUser.shared().finishInstructionsFor(InstructionScreens(controller: self)) {
            coachMarksController?.startOn(self)
        }
    }

    // MARK: - Protocol Conformance | CoachMarksControllerDelegate
    public func didFinishShowingFromCoachMarksController(coachMarksController: CoachMarksController) {
        GRUser.shared().finishInstructionsFor(InstructionScreens(controller: self))
    }
}

{% endhighlight %}

I'm genuinely impressed with Swift enums, they make code so expressive and readable!
