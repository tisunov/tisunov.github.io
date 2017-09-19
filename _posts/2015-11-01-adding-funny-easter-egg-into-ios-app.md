---
layout: single
title: Adding funny Easter egg into iOS app
categories: []
tags: []
published: True

---

I often scan [GitHub Trending repositories](https://github.com/trending) looking for some cool new tools and controls for iOS and Web apps. I stumbled on [UViewXXYBoom](https://github.com/xxycode/UIViewXXYBoom) and got an idea to have fun and add easter egg into my own grocery delivery app.

_UIViewXXYBoom_ is simply a UIView extension so it will be easy to integrate to explode collection view cells.

{:.center}
![UIViewXXYBoom](/assets/easter-egg/XXYBoom.gif)
_UIViewXXYBoom_

## Easter Egg

When person navigates to Fruit Aisle in our app and shakes the phone, random fruits will start exploding one by one.

The rules will be simple, first 3 people who find the Easter egg and send us a screen shot will get free delivery and hopefully have fun.

{:.center}
![Fruit Aisle](/assets/easter-egg/fruit-aisle.jpg){:height="500px"}
_Fruit Aisle_

## Detect shake and explode random visible collection view cells

I start by adding Swift bridging header and detecting shake gesture in view controller:

**GRShelfViewController.m**
{% highlight objective-c %}

#import "Groser-Swift.h"

- (void)motionEnded:(UIEventSubtype)motion withEvent:(UIEvent *)event {
    if (motion == UIEventSubtypeMotionShake) {
        [self startExplosions];
    }
}

{% endhighlight %}

Then I disable double explosion by keeping track if explosions are in progress and if we are in Fruit Aisle by checking aisle name. I'll reset __isExploding_ when explosions are completed:

{% highlight objective-c %}

- (void)startExplosions {
    if (_isExploding || ![self.shelf.name isEqualToString:@"Фрукты"]) return;
    _isExploding = YES;
}

{% endhighlight %}

Code below goes inside **startExplosions** method. 

Next I want to get all completely visible cells, so we won't explode cells partially behind navigation and tab bars:

{% highlight objective-c %}
// Get completely visible cells
NSArray *visibleCells = [self.collectionView.visibleCells select:^BOOL(UIView *cell) {
    CGRect cellRect = [self.collectionView convertRect:cell.frame toView:self.collectionView.superview];
    
    return CGRectContainsRect(self.collectionView.frame, cellRect);
}];
{% endhighlight %}

To make explosions a bit unpredictable I will randomly generate unique set of cell indexes:

{% highlight objective-c %}
NSMutableSet *uniqueCellIndexes = [NSMutableSet set];
for (int i = 0; i < visibleCells.count; i++) {
    NSInteger cellIndex = arc4random_uniform((u_int32_t)visibleCells.count);
    [uniqueCellIndexes addObject:@(cellIndex)];
}
{% endhighlight %}

To explode cells one by one I adapted demo code from _UIViewXXYBoom_ by creating Swift UIView extension with method _delayTask_ to delay individual cell explosion by a number of seconds which equals to cell's index. In the end each explosion will be delayed by 1 second from the previous one.

{% highlight objective-c %}

[uniqueCellIndexes eachWithIndex:^(NSNumber *cellIndex, NSUInteger delaySeconds) {
    UIView *cell = visibleCells[cellIndex.intValue];
    [cell delayTask:delaySeconds task:^{
        [cell boom];
    }];
}];

{% endhighlight %}

After exploding all selected cells make a dramatic pause for number of exploded cells plus 2 seconds, restore cell's visibility and reset __isExploding_ property to again allow explosions on shake.

{% highlight objective-c %}
// After all explosions, delay for count + 2 seconds
dispatch_time_t when = (uniqueCellIndexes.count + 2) * 1000 * USEC_PER_SEC;

@weakify(self)
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, when), dispatch_get_main_queue(), ^{
    @strongify(self)
    
    // Restore cells visibility
    [UIView animateWithDuration:0.35 animations:^{
        for (UIView *cell in visibleCells) {
            cell.alpha = 1.0;
        }
    } completion:^(BOOL finished) {
        self.isExploding = NO;
    }];
});
{% endhighlight %}

Here is how it looks:

{:.center}
![Fruit Explosions](/assets/easter-egg/fruits-boom.gif){:height="566px"}
_Fruit Explosions_
