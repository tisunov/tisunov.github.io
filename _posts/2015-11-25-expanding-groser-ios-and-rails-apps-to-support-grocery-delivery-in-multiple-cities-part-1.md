---
layout: single
title: Extending Groser iOS and Rails apps to support grocery delivery in multiple cities - Part 1
categories: []
tags: []
published: True

---

Initially we were trying to proof a concept and didn't think it will work, now over 1 year after our first delivery we're facing new opportunity. After getting multiple inquiries from eager entrepreneurs in large cities, we decided to franchise our model. That requires extending internal architecture to support multiple cities.

Current architecture supports grocery deliveries from one store in one city only. We have 1 iOS client app, 1 iOS Shopper app and Rails back end. All apps are working over REST API namespaced under _API::V1_, supporting multiple cities will require creating new and introducing breaking changes into old API.

## Setting up Rails routes

Let's start with copying _API::V1_ code in `app/views/api/v1`, `app/controllers/api/v1`, `app/helpers/api/v1` into new `v2` directory.

{% highlight console %}
$ cp -R app/controllers/api/v1 app/controllers/api/v2
$ cp -R app/views/api/v1 app/views/api/v2
$ cp -R app/helpers/api/v1 app/helpers/api/v2
{% endhighlight %}

Then I'll update `routes.rb` and simply copy `v1` namespace into `v2`.

{% highlight ruby %}
namespace :v2 do
  # client app
  devise_scope :user do
    post "/sign_up", :to => 'registrations#create'
    post "/sign_in", :to => 'sessions#create'
    post "/facebook", :to => 'registrations#facebook'
    post "/vkontakte", :to => 'registrations#vkontakte'
    delete "/sign_out", :to => 'sessions#destroy'
  end
  ...
{% endhighlight %}

I use excellent [Sublime Text 2](http://www.sublimetext.com/2) and will use its project file `folder_exclude_patterns` setting to exclude `V1` directories to avoid accedentally changing `V1` of API.

{% highlight json %}
{
  "folders":
  [
    {
      "folder_exclude_patterns":
      [
        "tmp", "*/v1"
      ],
      "file_exclude_patterns":
      [
        ".keep"
      ]
    }
  ]
}
{% endhighlight %}

After excluding old API from text editor, I'll use global Replace command to change module V1 namespace to V2.

{:.center}
![Sublime Text 2 Replace](/assets/multi-city-1/rename-v1-v2.png){:width="500px"}

After that cleaned up deprecated code used to support old version of the iOS apps, when I didn't think it merited upgrading API version.

## Switch iOS app to API V2

I use [AFNetworking](https://github.com/AFNetworking/AFNetworking) with `AFHTTPSessionManager` descendant initialized with the base URL of an API. Just change v1 to v2 at the end of the URL.

{% highlight objective-c %}

#if !(TARGET_IPHONE_SIMULATOR)
NSString * const kBaseUrl = @"http://www.groser.ru/api/v2";
#else
NSString * const kBaseUrl = @"http://localhost:3000/api/v2";
#endif

{% endhighlight %}

## Multi city delivery data model

Before extending current architecture for multi city deliveries, I drew the diagram of the current architecture and added necessary models and associations.

{:.center}
![Object Model](/assets/multi-city-1/model.jpg){:width="800px"}
*Left: Current Model, Right: Extension*

### Delivery Zones

We'll do limited roll out of delivery across city with by dividing city into geogrpahical zones. I'll create new `Zone` model with `geometry:json` attribute to keep `GeoJSON`-encoded geometry of the delivery zone:

{% highlight console %}
rails g model Zone city:references geometry:json --no-test-framework
{% endhighlight %}

Then I need to create default delivery zone for the current city we work in, Moscow.

{% highlight ruby %}

class AddDefaultMoscowDeliveryZone < ActiveRecord::Migration
  def self.up
    city = City.find_by_name('Москва')
    Zone.create!(city: city, geometry: city.geometry)
  end

  def self.down
    raise ActiveRecord::IrreversibleMigration
  end  
end

{% endhighlight %}

### Delivery windows

At the start we'll support delivery from one chain of supermarkets in the city, so I'll create many-to-one association between `DeliveryOption` used to track availability of delivery windows, and `City`. Later when we'll support delivery from multiple supermarkets in the city, I'll move that association to `Store` model.

{% highlight console %}
rails g migration add_city_to_delivery_option city:references
{% endhighlight %}

### Product variants

To enable variable prices for products in multiple cities, I need to extract `Product` attributes that will vary between cities. Attributes that will stay the same: 

  - title
  - weight
  - photos
  - description
  - barcode 

Attributes that I need to extract are: 

 - unit_price
 - store_price
 - department_id
 - shelf_id
 - store_id
 - often_out_of_stock
 - unlisted
 - frequently_found
 - loss_leader
 - out_of_stock
 - out_of_stock_count
 - found_count
 - user_favorites_count
 - order_items_count

I also need to keep `barcode` attribute in 2 places, cause bulk product barcode varies between stores. Let's create `ProductVariant` model.

{% highlight console %}
rails g model ProductVariant product:references store:references department:references shelf:references unit_price:float store_price:float editor_id:integer user_favorites_count:integer order_items_count:integer found_count:integer out_of_stock_count:integer flags:integer barcode:string --no-test-framework
{% endhighlight %}

Instead of boolean fields I will use [bitmask_attributes](https://github.com/joelmoss/bitmask_attributes)
<figcaption>product_variant.rb</figcaption>
{% highlight ruby %}
class ProductVariant < ActiveRecord::Base
  belongs_to :product
  belongs_to :store
  belongs_to :department
  belongs_to :shelf
  belongs_to :editor, class_name: 'User'

  # Don't change the order of the values, remove any values, or 
  # insert any new values in the `:as` array anywhere except at the end
  bitmask    :flags, :as => [:unlisted, :out_of_stock, :often_out_of_stock, :frequently_found, :loss_leader]

  validates_presence_of :product, :store, :department, :shelf, :store_price

  default_scope { without_flags(:unlisted).includes(:product) }
end
{% endhighlight %}

Right now we have only one `Store` with one set of products. To make it all work, let's create `ProductVariant` for each `Product` currently in database.

{% highlight console %}
rails g migration create_product_variants_from_current_products
{% endhighlight %}


{% highlight ruby %}
class CreateProductVariantsFromCurrentProducts < ActiveRecord::Migration
  def self.up
    default_store = Store.find_by_name('Ашан')

    Product.where('store_id = ? and department_id IS NOT NULL and shelf_id IS NOT NULL', default_store.id).find_each do |product|
      attributes = {
        product:              product,
        store:                default_store,
        shelf_id:             product.shelf_id,
        department_id:        product.department_id,
        store_price:          product.store_price,
        user_favorites_count: product.user_favorites_count,
        order_items_count:    product.order_items_count,
        found_count:          product.found_count,
        out_of_stock_count:   product.out_of_stock_count,
      }
      
      attributes[:barcode] = product.barcode if product.bulk?

      flags = []
      flags << :frequently_found if product.frequently_found
      flags << :loss_leader if product.loss_leader
      flags << :often_out_of_stock if product.often_out_of_stock
      flags << :out_of_stock if product.out_of_stock
      flags << :unlisted if product.unlisted
      
      attributes[:flags] = flags unless flags.empty?

      ProductVariant.create!(attributes)
    end
  end
end
{% endhighlight %}

### Replace Product references in queries and associations with ProductVariant

First up is an aisle model which I mistakenly named `Shelf`

<figcaption>shelf.rb</figcaption>
{% highlight ruby %}
class Shelf < ActiveRecord::Base
  has_many :product_variants, -> { order('products.name ASC') }, dependent: :destroy
  has_many :popular, -> { order('product_variants.order_items_count DESC') }, class_name: 'ProductVariant'

  ...
end

{% endhighlight %}

It's `Department` turn.

<figcaption>department.rb</figcaption>
{% highlight ruby %}
class Department < ActiveRecord::Base
  has_many :product_variants
  has_many :popular, -> { order('order_items_count DESC').limit(6) }, class_name: 'ProductVariant'

  ...
end
{% endhighlight %}

Next up is `ShelvesController`.
<figcaption>shelves_controller.rb</figcaption>
{% highlight ruby %}
class Api::V2::ShelvesController < Api::V2::ApiController
  respond_to :json

  def show
    shelf = Shelf.find(params[:id])
    products_json = Rails.cache.fetch("shelves/#{shelf.id}_#{shelf.updated_at}", {raw: true}) do
      products = ProductVariant.where(shelf_id: params[:id]).order('products.name ASC')
      products.to_json
    end

    render :json => products_json      
  end

  ...
end
{% endhighlight %}

Time to move `Product.as_json` method to `ProductVariant`. I override `Product.as_json` method to avoid using partials, which do slow down serialization to JSON.

<figcaption>product_variant.rb</figcaption>
{% highlight ruby %}

def as_json(options = {})
  attrs = [:id, :shelf_id].concat(options[:only] || [])
  options.delete :only

  json = super({:only => attrs}.merge(options))
  json[:name]               = self.product.name
  json[:unit]               = self.product.unit
  json[:unit_price]         = self.product.unit_price
  json[:unit_size]          = self.product.unit_size
  json[:container]          = self.product.container
  json[:brand]              = self.product.brand if self.product.brand.present?
  json[:image_small_url]    = self.product.photo.small.url if self.product.photo?
  json[:often_out_of_stock] = true if self.flags?(:often_out_of_stock)
  json[:out_of_stock]       = true if self.flags?(:out_of_stock)
  json
end

{% endhighlight %}

### Upgrade UserFavorites to ProductVariant

Turns out there are many more dependencies on `Product`, I use `UserFavorite` to track which products user has liked which, now `UserFavorite` should reference `ProductVariant` instead of `Product`.

{% highlight console %}
rails g migration upgrade_user_favorites_to_product_variant
{% endhighlight %}

I'm iterating over each `UserFavorite` setting `product_variant` attribute to `ProductVariant` created in previous migration for the given `UserFavorite.product`

{% highlight ruby %}

class UpgradeUserFavoritesToProductVariant < ActiveRecord::Migration
  def self.up
    add_column :user_favorites, :product_variant_id, :integer
    add_index :user_favorites, :product_variant_id

    ActiveRecord::Base.transaction do
      UserFavorite.all.each do |favorite|
        favorite.update(product_variant: ProductVariant.find_by_product(favorite.product))
      end

      remove_column :user_favorites, :product_id
    end
  end

  def self.down
    raise ActiveRecord::IrreversibleMigration
  end  
end

{% endhighlight %}

Update `User.favorite_products` association to `ProductVariant`

<figcaption>user.rb</figcaption>
{% highlight ruby %}
class User < ActiveRecord::Base
  has_many :favorite_products, class_name: 'ProductVariant', :through => :user_favorites, :source => :product_variant

  def likes?(product_variant)
    self.user_favorites.where(product_variant: product_variant).any?
  end

  def like(product_variant, likes)
    if likes
      UserFavorite.where(product_variant: product_variant, user: self).first_or_create
    else
      favorite = UserFavorite.where(product_variant: product_variant, user: self).first
      favorite.destroy if favorite
    end

    product_variant.reload
  end  
end
{% endhighlight %}

### Move Product markup calculation to ProductVariant

We mark up product prices for customers to incentivize shoppers who pickup and deliver products.

<figcaption>product_variant.rb</figcaption>
{% highlight ruby %}
class ProductVariant < ActiveRecord::Base
  ...
  before_save   :markup_unit_price

  def marked_up_price(store_price)
    if self.loss_leader
      store_price
    elsif store_price > 0
      expensive_item = store_price >= EXPENSIVE_ITEM_THRESHOLD_PRICE

      markup_percent = (self.product.is_alcoholic or expensive_item) ? ALCOHOLIC_AND_EXPENSIVE_ITEMS_MARKUP_PERCENT : MARKUP_PERCENT
      
      # mark it up and round to 2 floating point digits 
      ((store_price + store_price * (markup_percent / 100.0)) * 100).round / 100.0
    end
  end

  ...

  private

  def markup_unit_price
    self.unit_price = marked_up_price(self.store_price)
    true
  end

end
{% endhighlight %}

### Upgrade OrderItem to ProductVariant

I fear this the most, cause 1 mistake in migration can break everything. I'll keep original `OrderItem.product_id` attribute intact, and remove it when everything works again.

First, I'll add `OrderItem.product_variant` reference and replace `product_id`, `replacement_id` and `chosen_replacement_id` with `ProductVariant` ids for given product's id.

{% highlight console %}
rails g migration add_order_item_product_variant_reference product_variant:references
{% endhighlight %}

{% highlight ruby %}
class AddOrderItemProductVariantReference < ActiveRecord::Migration
  def self.up
    add_reference :order_items, :product_variant, index: true

    replacements_count = 0

    ActiveRecord::Base.transaction do
      OrderItem.all.find_each do |item|
        product_variant = ProductVariant.find_by_product_id(item.product_id)
        
        update = {
          product_variant: product_variant
        }

        if item.replacement_id and replacement_variant = ProductVariant.find_by_product_id(item.replacement_id)
          update[:replacement] = replacement_variant
          replacements_count += 1
        end

        if item.chosen_replacement_id and chosen_replacement_variant = ProductVariant.find_by_product_id(item.chosen_replacement_id)
          update[:chosen_replacement] = chosen_replacement_variant
          replacements_count += 1
        end

        item.update(update)
      end
    end

    puts "Upgraded #{replacements_count} replacements"
  end

  def self.down
    raise ActiveRecord::IrreversibleMigration
  end
end

{% endhighlight %}

Change `replacement` and `chosen_replacement` class name to `ProductVariant`.
<figcaption>order_item.rb</figcaption>
{% highlight ruby %}
class OrderItem < ActiveRecord::Base
  ...

  belongs_to :product
  belongs_to :product_variant, counter_cache: true
  belongs_to :replacement, class_name: 'ProductVariant'
  belongs_to :chosen_replacement, class_name: 'ProductVariant'

  delegate :shelf, :to => :product_variant
  delegate :department, :to => :product_variant

  validates_presence_of :product_variant, :unless => :special_request?
  ...
end
{% endhighlight %}

### Upgrade possible product replacements

Oftentimes some products are sold out at the particular store, especially meat and produce in the evening, we give customer choice of possible replacements before placing an order. Let's upgrade from 'Product' to 'ProductVariant' for replacements.

{% highlight console %}
rails g migration add_product_variant_to_related_products product_variant:references
{% endhighlight %}

{% highlight ruby %}

class AddProductVariantToRelatedProducts < ActiveRecord::Migration
  def self.up
    add_reference :related_products, :product_variant, index: true
    remove_column :related_products, :product_id

    RelatedProduct.delete_all
  end

  def self.down
    raise ActiveRecord::IrreversibleMigration
  end

end

{% endhighlight %}

There are hundred thousands of `RelatedProduct` records and it's eastier to recreate them later than find `ProductVariant` for each `Product` that it used to reference.

### Upgrade user's preferred product quantity

We keep track of quantity user ordered before, so if he bought 3kg of apples, next time he wants to order same apples, he won't have to select the weight.

{% highlight console %}
rails g migration add_product_variant_to_product_preference product_variant:references
{% endhighlight %}

{% highlight ruby %}
class AddProductVariantToProductPreference < ActiveRecord::Migration
  def change
    add_reference :product_preferences, :product_variant, index: true

    ProductPreference.all.find_each do |pref|
      pref.update(product_variant: ProductVariant.find_by_product_id(pref.product_id))
    end

    remove_column :product_preferences, :product_id
  end
end
{% endhighlight %}

<figcaption>user.rb</figcaption>
{% highlight ruby %}
def preferred_qty(product_variant)
  product_preferences.find_by(product_variant: product_variant).try(:qty)
end
{% endhighlight %}

In [Part 2]({% post_url 2015-11-30-extending-groser-ios-and-rails-apps-to-support-grocery-delivery-in-multiple-cities---part-2 %}), I'll upgrade search to use `ProductVariant`, upgrade order dispatcher to support multiple stores and route orders to city's shoppers.