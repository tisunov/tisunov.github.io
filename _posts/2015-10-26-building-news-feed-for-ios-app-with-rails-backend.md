---
layout: post
title: Building News Feed for iOS app with Rails backend
categories: []
tags: []
published: True

---

## Motivation

Let's implement news feed for my [Groser](https://itunes.apple.com/en/app/groser/id915288141) iOS app, it's a front end to our same day grocery delivery service in Moscow. When shoppers pick order in the store they update product prices and create new products. This activity will generate events which our clients can view inside iOS app, discover what's new and find out about price changes. 

The idea is to prompt users to return to the app and act on news feed events by ordering groceries. 

## Create static feed

To start we’ll create **Api::NewsFeedController** in Rails app:

{% highlight console %}
rails g controller Api/V1/NewsFeed --no-helper --no-assets --no-controller-specs --no-view-specs --no-views
{% endhighlight %}

I prefer to build iOS UI first and work on returning real news feed events. So we return JSON of static news items and move on to start working on iOS app.

Let's start wtih 2 basic kinds of events for now

* _New Product_. Happens when shopper creates new product which was missing from our catalog
* _Price Change_. So clients can bargain hunt without visiting store in person

{% highlight ruby %}

class Api::V1::NewsFeedController < Api::V1::ApiController
  respond_to :json

  def index
    p = Product.find(44806)

    events = [
      { event: 'new_product', 
        product: p.as_json, 
        time_ago: time_ago_in_words(p.created_at) },

      { event: 'price_updated', 
        product: p.as_json, 
        time_ago: time_ago_in_words(p.created_at), 
        old_price: 
        number_to_currency(55.0, locale: :ru) },

      { event: 'price_updated', 
        product: p.as_json, 
        time_ago: time_ago_in_words(p.created_at), 
        old_price: number_to_currency(75.0, locale: :ru) },

      { event: 'new_product', 
        product: p.as_json, 
        time_ago: time_ago_in_words(p.created_at) }
    ]

    render :json => events
  end

end

{% endhighlight %}

Format date and currency on backend so that clients won't have to do that

## Building iOS feed consumer

Fire up Xcode and create new Swift class NewsFeedViewController inherit it from UICollectionViewController

{% highlight ruby %}

class NewsFeedViewController : UICollectionViewController {
  
}

{% endhighlight %}

Drag onto storyboard UICollectionViewController and set it's class to NewsFeedViewController. We'll show each event in news feed as a list of collection view cells.

{:.center}
![News feed mockup](/assets/news-feed/mockup.jpg){:height="400px"}
*Rough mockup of news feed with 3 item types*

### Model & Network Layer

Create GRNewsFeedEvent Objective C class to represent news feed events. We use Objective C for model classes because I use [Mantle](https://github.com/Mantle/Mantle) for JSON to models mapping and it isn't compatible with Swift 2.0. I'll prefer Swift for new code and use Objective C only when necessary.

**GRNewsFeedEvent.h**
{% highlight objective-c %}

#import "GRProduct.h"

typedef NS_ENUM(NSUInteger, GRNewsFeedEventType) {
    GRNewsFeedPriceChangeEvent = 1,
    GRNewsFeedNewProductEvent = 2,
};

@interface GRNewsFeedEvent : MTLModel <MTLJSONSerializing>
@property (nonatomic, readonly) GRNewsFeedEventType eventType;
@property (nonatomic, strong, readonly) GRProduct *product;
@property (nonatomic, copy, readonly) NSString *timeAgo;
@property (nonatomic, copy, readonly) NSString *oldPrice;
@end


{% endhighlight %}

**GRNewsFeedEvent.m**
{% highlight objective-c %}

#import "GRNewsFeedEvent.h"

@implementation GRNewsFeedEvent

+ (NSDictionary *)JSONKeyPathsByPropertyKey {
    return @{
        @"eventType": @"event",
        @"product": @"product",
        @"timeAgo": @"time_ago",
        @"oldPrice": @"old_price"
    };
}

+ (NSValueTransformer *)eventTypeJSONTransformer {
    NSDictionary *events = @{
        @"price_updated": @(GRNewsFeedPriceChangeEvent),
        @"new_product": @(GRNewsFeedNewProductEvent),
    };
    
    return [MTLValueTransformer reversibleTransformerWithForwardBlock:^(NSString *event) {
        return events[event];
    } reverseBlock:^(NSNumber *event) {
        return [events allKeysForObject:event].lastObject;
    }];
}

+ (NSValueTransformer *)productJSONTransformer {
    return [NSValueTransformer mtl_JSONDictionaryTransformerWithModelClass:GRProduct.class];
}

@end


{% endhighlight %}

Next up, add new method to network service GRGroserService to request all news feed events (it will be disastrous in production, we'll add paging later)

**GRGroserService.m**
{% highlight objective-c %}

- (void)loadNewsFeedCompletion:(GRClientCompletionBlock)completion {
    NSDictionary *parameters = [self buildParameters:nil];
    
    [self GET:@"news_feed" parameters:parameters completion:^(OVCResponse *response, NSError *error) {
        completion(response.result, error);
    }];
}

{% endhighlight %}

Method *buildParameters* creates request parameters with device id, user access token for signed in user, timestamp, etc for logging.

Setup JSON to model mapping with [Overcoat](https://github.com/Overcoat/Overcoat), it acts as a glue between Mantle and AFNetworking returning parsed models from REST requests.

{% highlight objective-c %}

+ (NSDictionary *)modelClassesByResourcePath {
    return @{
        @"news_feed": GRNewsFeedEvent.class
    };
}

{% endhighlight %}

Completion block passed to *loadNewsFeedCompletion* will receive NSArray of *GRNewsFeedEvents*.

### Event presentation

Create new UICollectionViewCell descendant for each event type. Add initialization method to load data from model into view

Layout collection view cells in storyboard and setup subview outlets

{:.center}
![CollectionView cells layout](/assets/news-feed/collectionview-cells.png){:height="400px"}
*We'll use similar layout for New Product and Price Change cells*

Format product name, price and size using NSAttributedString. Display price and size under product name, without using second label and setting up contraints. Also make a thin grey border around cell in _awakeFromNib_.

**NewProductEventCell.swift**
{% highlight swift %}

class NewProductEventCell: UICollectionViewCell {
    @IBOutlet var imageView: UIImageView!
    @IBOutlet var nameLabel: UILabel!
    @IBOutlet var timeAgoLabel: UILabel!
    
    override func awakeFromNib() {
        super.awakeFromNib()
        
        layer.borderWidth = 0.5;
        layer.borderColor = UIColor.init(red:0.882, green:0.905, blue:0.900, alpha: 1.0).CGColor
    }
    
    func setup(event: GRNewsFeedEvent) {
        let product = event.product
        
        imageView.sd_setImageWithURL(product.imageSmallURL, placeholderImage: UIImage.init(named: "missing-item"))
        timeAgoLabel.text = event.timeAgo
        
        let details = "\(product.priceString) • \(product.sizeString)"
        
        let attributes = [
            NSForegroundColorAttributeName: nameLabel.textColor,
            NSFontAttributeName: nameLabel.font
        ]

        let attributedTitle = NSMutableAttributedString.init(string: "\(product.name)\n\(details)", attributes: attributes)
        
        let paragraphStyle = NSMutableParagraphStyle.init()
        paragraphStyle.paragraphSpacingBefore = 6
        
        attributedTitle.setAttributes([
            NSForegroundColorAttributeName: UIColor.init(white:0.568, alpha:1.000),
            NSParagraphStyleAttributeName : paragraphStyle],
            range:NSMakeRange(event.product.name.characters.count, details.characters.count));
        
        nameLabel.attributedText = attributedTitle
    }
}

{% endhighlight %}

### Display events in UICollectionView 

Returning to our view controller. Implement UICollectionViewDataSource protocol and make cells as wide as our view in _viewDidLoad_.

{% highlight swift %}

class NewsFeedViewController: UICollectionViewController {
    var events : [GRNewsFeedEvent]?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Set item width equal to view width
        let layout = self.collectionView!.collectionViewLayout as! UICollectionViewFlowLayout
        layout.itemSize = CGSizeMake(self.view.frame.size.width, layout.itemSize.height)
    }
    
    override func collectionView(collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return events!.count
    }
    
    override func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
        let event = events[indexPath.row]
        var cell: UICollectionViewCell
        
        switch event.eventType {
            case .NewProductEvent:
                cell = collectionView.dequeueReusableCellWithReuseIdentifier("NewProductCell", forIndexPath: indexPath)
                (cell as! NewProductEventCell).setup(event)

            case .PriceChangeEvent:
                cell = collectionView.dequeueReusableCellWithReuseIdentifier("PriceChangeCell", forIndexPath: indexPath)
                (cell as! PriceChangeEventCell).setup(event)
        }
        
        return cell
    }
}

{% endhighlight%}

Time to request news feed events from backend. Call following method from _viewDidLoad_.

{% highlight swift %}

func loadEvents() {
    GRGroserService.shared().loadNewsFeedCompletion({
        // Define capture list and make self reference weak, to avoid retain cycles
        [weak self] (events: AnyObject!, error: NSError!) -> Void in
        if let strongSelf = self {
            if (error == nil) {
                strongSelf.events = events as? [GRNewsFeedEvent]
                strongSelf.collectionView?.reloadData()
            }
        }
    })
}

{% endhighlight%}

Again, you don't have to know everything before starting to work. In this case I didn't know how to define weak self reference for closure with arguments in Swift (I'm still learning it), quick Google search and I found [Stack Overflow answer](http://stackoverflow.com/questions/24468336/how-to-correctly-handle-weak-self-in-swift-blocks-with-arguments).

### Wire up NewsFeedViewController with TabBarController

{% highlight swift %}

let feedController = UINavigationController.init(rootViewController: mainStoryboard.instantiateViewControllerWithIdentifier("NewsFeedViewController"))
feedController.tabBarItem = UITabBarItem.init(title: "НОВОСТИ", image: UIImage.init(named: "tab-feed"), tag: 0)

{% endhighlight %}

### Run the app

I declared _events_ var as optional and when tried to unwrap it in collectionView:numberOfItemsInSection I got EXC_BAD_ACCESS exception. From my Objective C days I got used that nil is equivalent to 0, so if events is nil, call to _count_ will be evaluated as nil and be casted to 0. 

{% highlight swift %}

override func collectionView(collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
    return events!.count
}

{% endhighlight %}

Swift is strongly typed language, so I had to initialize _events_ var as empty array.

{% highlight swift %}

var events : [GRNewsFeedEvent] = []

{% endhighlight %}

{:.center}
![CollectionView cells layout](/assets/news-feed/tab-feed1.png){:height="300px"}
*Yay, at least it doesn't crash :)*

Not what I expected, let's see what is happening. To start I'll add more pleasant background color to CollectionView, and make cells' background white.

Next, It seems that server returns HTTP 500 error, because I use _time_ago_in_words_ view helper inside _NewsFeedController_, update it to include _ActionView::Helpers::DateHelper_ and _ActionView::Helpers::NumberHelper_ for _number_to_currency_ helper.

{% highlight ruby %}

class Api::V1::NewsFeedController < Api::V1::ApiController
  include ActionView::Helpers::DateHelper
  include ActionView::Helpers::NumberHelper

{% endhighlight %}

It's nice to know that app is doing something, and actually works. Show UIActivityIndicator while events are being loaded. And here is the result.

{:.center}
![Groser News feed loading](/assets/news-feed/tab-feed-activity.png){:height="400px"}
![Groser News feed](/assets/news-feed/tab-feed2.png){:height="400px"}
*Groser News Feed*

## Making it real

I realized that product's price will keep changing after creation, and using current product's price in news feed is not correct. We have to keep track new product's price in news event.

I'll start by creating NewsItem model in our rails app

{% highlight console %}

rails g model NewsItem product:references attributes:json --no-test-framework

{% endhighlight %}

Obviously each news item is related to _Product_ so we reference it, everything else will be kept in Postgres json field. To keep up with Rails's convention I'll rename _NewsFeedController_ to _NewsItemsController_.

Sometimes product has to be deleted, because it's a duplicate, remove related news items as well.

**product.rb**
{% highlight ruby %}

class Product < ActiveRecord::Base
    has_many :news_items, dependent: :destroy
end

{% endhighlight %}

### Generating NewsItems for new products

To generate NewsItem for new products I'll use _Product_'s _after_create_ callback. But to keep _Product_ model focused, I'll use Publish/Subscribe mechanism. Since I already use [_Whisper_](https://github.com/krisleech/wisper) gem in other parts of the project, include _Wisper::Publisher_ into _Product_ model and broadcast _product_created_ event.

{% highlight ruby %}
class Product < ActiveRecord::Base
  include Wisper::Publisher
  has_many :news_items, dependent: :destroy

  after_create :broadcast_product_created

  private

  def broadcast_product_created
    broadcast(:product_created, self, self.unit_price)
  end
end
{% endhighlight %}

I can't subscribe to _product_created_ event on the Product instance before it's created and subscribing after creation is too late. Thankfully _Whisper_ has mechanism for temporary global observers, let's use it.

{% highlight ruby %}

class Api::V1::Shoppers::ProductsController < ApplicationController
  def create
    # Globally subscribe listener for the duration of a block
    Wisper.subscribe(ProductUpdatesListener.new) do
      product = Product.create(product_params)
    end      

    respond_with product
  end
end

class ProductUpdatesListener
  def product_created(product, price)
    attributes = {
      event: :new_product,
      price: price
    }
    NewsItem.create(product: product, attributes: attributes)
  end
end

{% endhighlight %}

### Generating NewsItems for product price updates

In the same vein, let's create NewsItems for product price updates. Update _Product_ definition with _after_update_ callback.

{% highlight ruby %}

class Product < ActiveRecord::Base
  include Wisper::Publisher
  has_many :news_items, dependent: :destroy

  after_create :broadcast_product_created
  before_update :broadcast_product_price_updated

  private

  def broadcast_product_created
    broadcast(:product_created, self, self.unit_price)
  end

  def broadcast_product_price_updated
    broadcast(:product_price_updated, self, self.unit_price, self.unit_price_was) if self.store_price_changed?
  end  
end

{% endhighlight %}

I use _unit_price_, and track _store_price_ changes because _unit_price_ is the calculated price shown to consumer, and _store_price_ is what will change when shopper updates price while picking an order in store.

Make _price_updated_ event more informative and extend it with change percentage and direction.

**product_updates_listener.rb**
{% highlight ruby %}
def product_price_updated(product, price, old_price)
  relative_change = price - old_price
  percent = (relative_change / old_price) * 100).abs.round

  attributes = {
    event: :price_updated,
    price: price,
    old_price: old_price,
    percent: percent,
    discounted: relative_change < 0
  }
  NewsItem.create(product: product, attributes: attributes)
end
{% endhighlight %}

### Update NewsItemsController to return NewsItems

Order _NewsItems_ from most to least recent and render them as JSON using JBuilder. Since we use views to generate JSON, view helper includes also gone from the controller.

**NewsItemsController**
{% highlight ruby %}
class Api::V1::NewsItemsController < Api::V1::ApiController
  respond_to :json

  def index
    @news_items = NewsItem.includes(:product).order('created_at DESC')
    respond_with @news_items
  end
end

{% endhighlight %}

**views/api/news_items/index.json.jbuilder**
{% highlight ruby %}

json.array!(@items) do |news_item|
  json.extract!  news_item, :product
  json.extract!  news_item.attributes, :event, :percent, :discount
  json.price     number_to_currency(news_item.attributes['price'], locale: :ru)
  json.old_price number_to_currency(news_item.attributes['old_price'], locale: :ru) if news_item.attributes['old_price']
  json.time_ago  time_ago_in_words(news_item.created_at)
end
{% endhighlight %}

## Seed with real news items

Here we are going to use treasure trove of historic data we've been tracking on server to create _NewsItems_. First step is to write rake task to generate NewsItems with new Products for the last 30 days.

To avoid code duplication extract specific _NewItem_ creation from _ProductUpdatesListener_ into _NewItem_ class methods.

**NewsItem**
{% highlight ruby %}
class NewsItem < ActiveRecord::Base
  belongs_to :product
  validates_presence_of :product

  def self.new_product(product, store_price, created_at = nil)
    parameters = {
      event: :new_product,
      price: product.marked_up_price(store_price)
    }
    create(product: product, parameters: parameters, created_at: created_at)
  end

  def self.price_updated(product, store_price, old_store_price)
    relative_change = store_price - old_store_price
    percent = ((relative_change / old_store_price) * 100).abs.round

    parameters = {
      event: :price_updated,
      price: product.marked_up_price(store_price),
      old_price: product.marked_up_price(old_store_price),
      percent: percent,
      discounted: relative_change < 0
    }
    create(product: product, parameters: parameters)
  end
end

{% endhighlight %}

In the following snippet we select all products which where created in the last month for the only store we deliver from, and create _NewsItems_ for each _Product_. _Product_ always contains current price, however it may have been created with another price. We try to find WarehouseProductPrice for this product created on the same date which keeps history of product price updates.

**lib/tasks/news_feed.rake**
{% highlight ruby %}
namespace :news_items do
  desc "Seed news feed with items"
  task seed: :environment do
    store = Store.find_by_name('Ашан')

    # Generate NewsItem new_product items
    Product.where('created_at >= ?', 1.month.ago.beginning_of_day).each do |product|
      warehouse_product_price = WarehouseProductPrice.where('DATE(created_at) = ?', product.created_at.to_date).first
      price = warehouse_product_price ? warehouse_product_price.price : product.store_price

      NewsItem.new_product(product, price, product.created_at)
    end

    # Generate NewsItem price_updated items
    WarehouseProductPrice.includes(:product)
                         .where('warehouse_product_prices.created_at >= ?', 1.month.ago.beginning_of_day)
                         .order('warehouse_product_prices.created_at DESC')
                         .group_by {|item| item.product }
                         .each do |product, items|
                           items.each_cons(2) do |price_after, price_before|
                             NewsItem.price_updated(product, price_after.price, price_before.price)
                           end
                         end    
  end
end
{% endhighlight %}

_WarehouseProductPrice_ keeps history of product price updates and _NewsItem_ requires previous product price. We iterate over _WarehouseProductPrices_ with sliding window of 2 elements. Since we ordered records from most recently created to least recently, first item in pair is the price after update and second is the price before update.

I pass _created_at_ attribute when creating NewsItem, otherwise _NewsItem.created_at_ will be current date. Final version of _NewsItem_:

{% highlight ruby %}

class NewsItem < ActiveRecord::Base
  belongs_to :product
  validates_presence_of :product

  def self.new_product(product, store_price, created_at = nil)
    parameters = {
      event: :new_product,
      price: product.marked_up_price(store_price)
    }
    create(product: product, parameters: parameters, created_at: created_at)
  end

  def self.price_updated(product, store_price, old_store_price, created_at = nil)
    relative_change = store_price - old_store_price
    percent = ((relative_change / old_store_price) * 100).abs.round

    parameters = {
      event: :price_updated,
      price: product.marked_up_price(store_price),
      old_price: product.marked_up_price(old_store_price),
      percent: percent,
      discounted: relative_change > 0
    }
    create(product: product, parameters: parameters, created_at: created_at)
  end
end

{% endhighlight %}

### Let's see how it all works

{% highlight console %}

rake news_items:seed
ActiveRecord::DangerousAttributeError: attributes is defined by Active Record

{% endhighlight %}

First error I see, is that I used existing attribute name _NewsItem.attributes_, rename it to _NewsItem.parameters_ and update NewsItem and JBuilder template. Now `rake news_items:seed` generates _NewsItems_ from history.

## Enhance news feed

I want to let users easily tell which CollectionView cell is new product and which is price update. Also I will show if price rose or declined and by how much, using text and color.

Create _NewsItemsHelper_ view helper and format title of the _NewsItem_.

**app/helpers/api/v1/news_items_helper.rb**
{% highlight ruby %}
module Api::V1::NewsItemsHelper
  def news_item_title(news_item)
    case news_item.params[:event]
    when 'new_product'
      "Новый продукт: "
    when 'price_updated'
      if news_item.params[:discounted]
        "Скидка #{news_item.params[:percent]}%: "
      else
        "Подорожание #{news_item.params[:percent]}%: "
      end
    end
  end
end
{% endhighlight %}

I also added two new convience methods to _NewsItem_. `price_updated?` needed for conditional JSON generation, and `params` allows access to _NewsItem.parameters_ by symbol.

{% highlight ruby %}
def price_updated?
  params[:event] == 'price_updated'
end

def params
  @params ||= parameters.with_indifferent_access
end
{% endhighlight %}


**views/api/news_items/index.json.jbuilder**
{% highlight ruby %}

json.array!(@news_items) do |news_item|
  json.extract!  news_item, :product
  json.title     news_item_title(news_item)
  json.extract!  news_item.params, :event
  json.time_ago  time_ago_in_words(news_item.created_at)
  json.price     number_to_currency(news_item.params[:price], locale: :ru)

  if news_item.price_updated?
    json.extract!  news_item.params, :percent, :discounted
    json.old_price number_to_currency(news_item.params[:old_price], locale: :ru)
  end
end

{% endhighlight %}

Quick check that everything works as expected. Instead of `curl` I use [httpie](https://github.com/jkbrzt/httpie) that formats response nicely.

{% highlight console %}
http localhost:3000/api/v1/news_items.json
{% endhighlight %}

{% highlight json %}
[
    {
        "event": "new_product",
        "price": "65,53 р.",
        "product": {
            "container": "bulk",
            "id": 44957,
            "name": "Ананас",
            "shelf_id": 945,
            "unit": "kg",
            "unit_price": 195.0,
            "unit_size": 1.0
        },
        "time_ago": "6ч",
        "title": "Новый продукт: "
    },
]
{% endhighlight %}

## Refactor iOS app news items related code

After finishing with our server side let's make naming consistent with our server side model, individual events become _News Items_ and one CollectionView cell can present both _NewsItem_ types.

 - Rename `GRNewsFeedEvent` to `GRNewsItem`, and `GRNewsFeedEventType` to `GRNewsItemType`.
 - Rename `NewProductEventCell` to `NewsItemCell`
 - Remove `PriceChangeEventCell` collection view cell and handle _NewsItem_ title formatting in `NewsItemCell`
 - Rename _NewsItems_ REST route from `news_feed` to `news_items` in our networking code
 - Change CollectionViewCell reuse identifier to `NewsItemCell`

**NewsItemCell.swift**
{% highlight swift %}
class NewsItemCell: UICollectionViewCell {
    @IBOutlet var imageView: UIImageView!
    @IBOutlet var nameLabel: UILabel!
    @IBOutlet var timeAgoLabel: UILabel!
    
    override func awakeFromNib() {
        super.awakeFromNib()
        
        layer.borderWidth = 0.5;
        layer.borderColor = UIColor.init(red:0.882, green:0.905, blue:0.900, alpha: 1.0).CGColor
    }
    
    func setup(event: GRNewsItem) {
        imageView.sd_setImageWithURL(event.product.imageSmallURL, placeholderImage: UIImage.init(named: "missing-item"))
        timeAgoLabel.text = event.timeAgo
        nameLabel.attributedText = buildName(event)
    }
    
    func buildName(event: GRNewsItem) -> NSAttributedString {
        let product = event.product
        
        let details = buildDetails(event)
        
        let attributes = [
            NSForegroundColorAttributeName: nameLabel.textColor,
            NSFontAttributeName: nameLabel.font
        ]
        
        let attributedTitle = NSMutableAttributedString.init(string: "\(event.title)\(product.name)\n\(details)", attributes: attributes)
        
        let paragraphStyle = NSMutableParagraphStyle.init()
        paragraphStyle.paragraphSpacingBefore = 6
        
        let beginningOfDetails = attributedTitle.length - details.characters.count
        let titleRange = NSMakeRange(0, event.title.characters.count - 1)
        // Format event title
        attributedTitle.addAttribute(NSFontAttributeName, value: UIFont.init(name: "HelveticaNeue-Medium", size: nameLabel.font.pointSize)!, range: titleRange)
        
        // Format product details
        attributedTitle.setAttributes([
            NSForegroundColorAttributeName: UIColor.init(white:0.568, alpha:1.000),
            NSParagraphStyleAttributeName : paragraphStyle],
            range:NSMakeRange(beginningOfDetails, details.characters.count - 1));
        
        nameLabel.attributedText = attributedTitle
        
        switch event.eventType {
        case .NewProduct:
            // We are good here
            break
        case .PriceChange:
            let color = event.discounted.boolValue ? UIColor( red: 0.2199, green: 0.652, blue: 0.4592, alpha: 1.0 ) : UIColor( red: 0.8055, green: 0.2338, blue: 0.2122, alpha: 1.0 )
            attributedTitle.addAttribute(NSForegroundColorAttributeName, value: color, range: titleRange)
            
            // Strike through old price
            attributedTitle.addAttribute(NSStrikethroughStyleAttributeName, value: NSUnderlineStyle.StyleSingle.rawValue, range: NSMakeRange(beginningOfDetails, event.oldPrice.characters.count - 1))
        }
        
        return attributedTitle
    }
    
    func buildDetails(event: GRNewsItem) -> String {
        var details: String
        
        switch event.eventType {
        case .NewProduct:
            details = "\(event.price) • \(event.product.sizeString)"
        case .PriceChange:
            details = "\(event.oldPrice)\n\(event.price) • \(event.product.sizeString)"
        }
        
        return details
   }
}
{% endhighlight %}

I hit an exception `NSUnknownKeyException reason: UICollectionViewCell setValue:forUndefinedKey: this class is not key value coding-compliant for the key imageView`, probably because there was some reference in storyboard to removed 2nd collection view cell, hit `Shift + Command + K` to clean project, built again and exception has gone. After couple of finishing touches our News Feed looks like this.

{:.center}
![News Feed complete](/assets/news-feed/tab-feed3.png){:height="500px"}
*News Feed after implementing all of the above steps*

## Empty state

Let's remove confusion around what News Feed is supposed to display when there are no news items or connection is offline and news items can't be loaded. I'll use [DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet) Cocoapod.

Implement _DZNEmptyDataSetSource_ and _DZNEmptyDataSetDelegate_ protocols. Below is the partial update of _NewsFeedViewController_ with parts relevant to DZNEmptyDataSet protocols' implementation. I added new boolean property to hide empty data set view while we are waiting for network response. And added button to allow user to manually reload news feed if it's empty.

**NewsFeedViewController.swift**
{% highlight swift %}
class NewsFeedViewController: UICollectionViewController, DZNEmptyDataSetSource, DZNEmptyDataSetDelegate {
    internal var isLoading = false
    override func viewDidLoad() {
        collectionView!.emptyDataSetSource = self
        collectionView!.emptyDataSetDelegate = self
    }

    func loadEvents() {
        isLoading = true
        startActivityIndicator()
        collectionView!.reloadEmptyDataSet()
        
        GRGroserService.shared().loadNewsFeedCompletion({
            // Define capture list and make self reference weak, to avoid retain cycles
            [weak self] (events: AnyObject!, error: NSError!) -> Void in
            if let strongSelf = self {
                strongSelf.isLoading = false
                strongSelf.stopActivityIndicator()
                
                if (error == nil) {
                    strongSelf.events = events as! [GRNewsItem]
                    strongSelf.collectionView?.reloadData()
                }
                
                strongSelf.collectionView?.reloadEmptyDataSet()
            }
        })
    }

    func titleForEmptyDataSet(scrollView: UIScrollView!) -> NSAttributedString! {
        let attributes = [NSFontAttributeName: UIFont(name: "HelveticaNeue", size: 20.0)!,
            NSForegroundColorAttributeName: UIColor(red:0.435, green:0.463, blue:0.479, alpha:1.000)]
        
        return NSAttributedString.init(string: "Нет изменений в каталоге товаров", attributes: attributes)

    }
    
    func descriptionForEmptyDataSet(scrollView: UIScrollView!) -> NSAttributedString! {
        let paragraph = NSMutableParagraphStyle()
        paragraph.lineBreakMode = .ByWordWrapping
        paragraph.alignment = .Center
        
        let attributes = [NSFontAttributeName: UIFont.systemFontOfSize(14.0),
            NSForegroundColorAttributeName: UIColor(red:0.519, green:0.531, blue:0.543, alpha:1.000),
            NSParagraphStyleAttributeName: paragraph]
        
        return NSAttributedString.init(string: "Здесь вы можете узнать о добавлении новых товаров в каталог и об изменении цен на существующие товары", attributes: attributes)
    }
    
    func imageForEmptyDataSet(scrollView: UIScrollView!) -> UIImage! {
        return UIImage.init(named: "blank-state-newsfeed")
    }
    
    func buttonTitleForEmptyDataSet(scrollView: UIScrollView!, forState state: UIControlState) -> NSAttributedString! {
        return NSAttributedString.init(string: "Загрузить снова", attributes: [NSForegroundColorAttributeName: GRTheme.emptyStateButtonColor()])
    }
    
    func configureButtonAppearance(button: UIButton!) {
        button.tintAdjustmentMode = .Dimmed;
        button.layer.borderColor = GRTheme.emptyStateButtonColor().CGColor;
        button.layer.borderWidth = 1.0;
        button.layer.cornerRadius = 3.0;
        button.tintColor = GRTheme.emptyStateButtonColor();
    }

    func emptyDataSetShouldDisplay(scrollView: UIScrollView!) -> Bool {
        return !isLoading
    }
    
    func emptyDataSetDidTapButton(scrollView: UIScrollView!) {
        loadEvents()
    }

    func offsetForEmptyDataSet(scrollView: UIScrollView!) -> CGPoint {
        return CGPoint(x: 0, y: -self.tabBarController!.tabBar.frame.height / 2)
    }    
}
{% endhighlight %}

{:.center}
![News Feed Empty State](/assets/news-feed/empty-state.jpg){:height="400px"}
*News feed empty state*

## Pull to refresh

Right now news feed loads one time when user opens the tab. For the impatient ones, who would like to refresh news feed at will we add _Pull to Refresh_ control, the one where you pull down on scrollable view to refresh it's contents. I stumbled on [MAGearRefreshControl](https://github.com/micazeve/MAGearRefreshControl) which I immediately liked.

{:.center}
![MAGearRefreshControl Sample](/assets/news-feed/gear.gif){:height="350px"}
*MAGearRefreshControl Sample*

I tried to install it using Cocoapods, but control requires minimum iOS 8, and I have to support iOS 7 for some time. I'll just import source file into Xcode project.

Next, implement _MAGearRefreshDelegate_ protocol and configure colors.

{% highlight swift %}
func setupRefreshControl() {
    navigationController?.navigationBar.translucent = false
    
    refreshControlView = MAGearRefreshControl(frame: CGRectMake(0, -collectionView!.bounds.height, collectionView!.frame.width, collectionView!.bounds.height))
    refreshControlView.backgroundColor = UIColor(red: 0.1221, green: 0.4543, blue: 0.3261, alpha:1.0) //UIColor(red: 34/255.0, green: 75/255.0, blue: 150/255.0, alpha: 1.0)
    
    // UIColor.initRGB(92, g: 133, b: 2316)
    refreshControlView.addInitialGear(12, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 1.0), radius:16)
    refreshControlView.addLinkedGear(0, nbTeeth:16, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 0.8), angleInDegree: 30)
    refreshControlView.addLinkedGear(0, nbTeeth:32, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 0.4), angleInDegree: 190)
    refreshControlView.addLinkedGear(1, nbTeeth:40, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 0.4), angleInDegree: -30)
    refreshControlView.addLinkedGear(2, nbTeeth:24, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 0.8), angleInDegree: -190)
    refreshControlView.addLinkedGear(3, nbTeeth:10, color: UIColor(red:57.0/255, green: 237.0/255, blue: 166.0/255, alpha: 1.0), angleInDegree: 40)
    refreshControlView.setMainGearPhase(0)
    refreshControlView.delegate = self
    refreshControlView.showBars = false
    collectionView!.addSubview(refreshControlView)
}

// MARK: - UIScrollViewDelegate protocol conformance

override func scrollViewDidScroll(scrollView: UIScrollView) {
    refreshControlView.MAGearRefreshScrollViewDidScroll(scrollView)
}

override func scrollViewDidEndDragging(scrollView: UIScrollView, willDecelerate decelerate: Bool) {
    refreshControlView.MAGearRefreshScrollViewDidEndDragging(scrollView)
}


// MARK: - MAGearRefreshDelegate protocol conformance

func MAGearRefreshTableHeaderDataSourceIsLoading(view: MAGearRefreshControl) -> Bool {
    return isLoading
}

func MAGearRefreshTableHeaderDidTriggerRefresh(view: MAGearRefreshControl) {
    loadEvents()
}
{% endhighlight %}

I also added boolean argument to `loadEvents(showActivityIndicator: Bool)` to disable activity indicator when update was triggered by refresh control.

<!-- ![News Feed Pull to Refresh](/assets/news-feed/refresh-control.gif){:height="400px"} -->

{:.center}
![News Feed Pull to Refresh](/assets/news-feed/refresh-control.gif){:height="500px"}
*News Feed Pull to Refresh*

## Show related product

It would be strange to see product price updates and won't be able to act on it. Final piece, show _ProductViewController_ when user taps news item.

{% highlight swift %}
override func collectionView(collectionView: UICollectionView, didSelectItemAtIndexPath indexPath: NSIndexPath) {
    let event = events[indexPath.row]
    
    let navController = UIStoryboard.init(name: "Main", bundle: nil).instantiateViewControllerWithIdentifier("ProductNavViewController") as! UINavigationController

    let controller = navController.topViewController as! GRProductViewController
    controller.order = GRShoppingCart.shared()
    controller.product = event.product;
    
    presentViewController(navController, animated: true, completion: nil)
}

{% endhighlight %}

## Production News Feed

Show maximum 50 events for the last month.

{% highlight ruby %}
class Api::V1::NewsItemsController < Api::V1::ApiController
  respond_to :json

  def index
    @news_items = NewsItem.includes(:product)
                          .order('created_at DESC')
                          .where('created_at >= ?', 1.month.ago.beginning_of_day)
                          .limit(50)

    respond_with @news_items
  end

end
{% endhighlight %}

I use [Capistrano](http://capistranorb.com/) for code deployment, so I need to add deploy.rb action to run rake task on server.

{% highlight ruby %}

desc 'Seed news items'
task :seed_news_items do
  on roles(:app) do
    within release_path do
      with rails_env: fetch(:rails_env) do
        execute :rake, 'news_items:seed'
      end
    end
  end
end
{% endhighlight %}

Commit, deploy code to production and seed news items.

{% highlight console %}
cap production deploy
cap production deploy:seed_news_items
{% endhighlight %}

And here is the final result:

{:.center}
![Production News Feed](/assets/news-feed/production-1.jpg){:height="500px"}
*Production News Feed*

As I expected there is a lot of noise, shoppers make mistakes and reverse price updates, or price deviates too little, there is no sense in displaying such noise.

I added filter to discard price updated news items if percetage change less than 3%:

{% highlight ruby %}
return if percent <= 3
{% endhighlight %}

And now we are done!

{:.center}
![Final production News Feed](/assets/news-feed/production-2.jpg){:height="500px"}
*Final production News Feed*
