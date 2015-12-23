---
layout: post
title: Extending Groser iOS and Rails apps to support grocery delivery in multiple cities - Part 2
categories: []
tags: []
published: True

---

This is Part 2 of multi part series about how I re-architect Rails and iOS apps to support grocery delivery in multiple cities to enable our new business model of selling SaaS solution to independent entrepreneurs wanting to create their own grocery delivery service. To understand rationale behind my decisions please read [Part 1]({% post_url 2015-11-25-expanding-groser-ios-and-rails-apps-to-support-grocery-delivery-in-multiple-cities-part-1 %})

## Upgrading Client app search API

I use [pg_search](https://github.com/Casecommons/pg_search) with scopes to search products by name. With `ProductVariant` I will need to narrow down `Product` search to specific store by `ProductVariant`. I will add `Product.search` scope, cause it will be used in multiple places.

<figcaption>product.rb</figcaption>
{% highlight ruby %}

class Product < ActiveRecord::Base
  scope :search, -> { |store_id, query| joins(:variants).where('product_variants.store_id = ? AND (product_variants.flags & 1 = 0 OR product_variants.flags IS NULL)', store_id).search_by_name(query).limit(30) }

  pg_search_scope :search_by_name,
                  :against => [:name], 
                  :using => {
                    :tsearch => { :normalization => 16, :dictionary => 'russian', :any_word => true, :prefix => true }, 
                    :trigram => { :threshold => 0.6 },
                  }
end
{% endhighlight %}

I'm sure it can be optimized further, but I want to get it done in shortest time possible, so querying `Product` ids and then selecting `ProductVariant` will do for now.

<figcaption>searches_controller.rb</figcaption>
{% highlight ruby %}
class Api::V2::SearchesController < Api::V2::ApiController
  respond_to :json

  def index
    raise ArgumentError.new("store_id must not be blank") if params[:store_id].blank?

    @products_grouped_by_shelf = []

    store_id = params[:store_id]
    query = params[:q].split.join(' ')

    product_ids = Product.search(store_id, query).pluck(:id)
    product_variants = ProductVariant.includes(:shelf).where(product_id: product_ids, store_id: store_id)

    @products_grouped_by_shelf = product_variants.group_by { |v| v.shelf }

    respond_with @products_grouped_by_shelf
  end

end
{% endhighlight %}

## Upgrading Shopper app search API

Next I need to upgrade shopper API which can search by `Product` name or barcode.

Since barcode can be universal as well as store specific (for bulk products), I query both `Product.barcode` and `ProductVariant.barcode`

<figcaption>product.rb</figcaption>
{% highlight ruby %}

class Product < ActiveRecord::Base
scope :search_barcode, -> { |store_id, barcode| joins(:variants).where('product_variants.store_id = ? AND (products.barcode = ? OR product_variants.barcode = ?)', store_id, barcode, barcode) }
end
{% endhighlight %}

<figcaption>shoppers/products_controller.rb</figcaption>
{% highlight ruby %}

class Api::V2::Shoppers::ProductsController < ApplicationController
  ...

  def index
    raise ArgumentError.new("store_id must be present") if params[:store_id].blank?
    raise ArgumentError.new("q or barcode param must be present") if params[:q].blank? and params[:barcode].blank?

    store_id = params[:store_id]

    if params[:q].present?
      products = Product.search(store_id, params[:q])
    else
      products = Product.search_barcode(store_id, params[:barcode])
    end 

    product_variants = ProductVariant.where(product_id: products.map(&:id), store_id: store_id)

    render :json => product_variants.as_json(:only => [:store_price, :department_id])
  end


  ...
end
{% endhighlight %}

## Upgrading Order Dispatcher

Order dispatcher assigns incoming orders to available shoppers according to schedule set by shoppers. First, I need to query all active and available shoppers for the client's city. I'll delegate `Order.city` to associated `User`.

<figcaption>order.rb</figcaption>
{% highlight ruby %}
class Order < ActiveRecord::Base
  ...
  delegate :city, :to => :user
  ...
end
{% endhighlight %}

Next, I'll add city query into shopper scopes, extracted into mixin.

<figcaption>has_roles.rb</figcaption>
{% highlight ruby %}
module HasRoles
  extend ActiveSupport::Concern
  ...
  scope :active_shoppers, lambda { |city| where('roles_mask & ? = ? AND city_id = ?', mask_for([:shopper, :active]), mask_for([:shopper, :active]), city.id) }
  scope :admin_shoppers, lambda { |city| where('roles_mask & ? = ? AND city_id = ?', mask_for([:shopper, :admin]), mask_for([:shopper, :admin]), city.id) }
  ...
end
{% endhighlight %}

Now I can begin working on `OrderDispatcher`. First I'll handle easy case of admin customer placing an order, and then general case.

<figcaption>order_dispatcher.rb</figcaption>
{% highlight ruby %}
class OrderDispatcher

  def self.dispatch(order)
    if order.user.is? :admin
      shopper = User.admin_shoppers(order.city).sample
      order.assign(shopper)
      return
    end

    shoppers = User.active_shoppers(order.city)
                   .includes(:schedule_hours)
                   .where(:schedule_hours => { active: true, deliveries_count: 0, blocked: false, starts_at: order.delivery_window_starts_at })
                   .select { |s| s.schedule_hours.count > 0 }

    ...

    shoppers_with_same_day_deliveries = 
      OrderDelivery.joins(:shopper, :delivery_option)
                   .where.not(shopper: nil, state: OrderDelivery.states[:canceled])
                   .where(:shopper => { city: city })
                   .where('DATE(delivery_options.window_starts_at) = ?', order.delivery_window_starts_at.to_date)
                   .map(&:shopper)
                   .select {|s| s.schedule_hours.where(active: true, deliveries_count: 0, blocked: false, starts_at: order.delivery_window_starts_at).count > 0 }
    
    ...    
  end

  ...
end
{% endhighlight %}

In [Part 3]({% post_url 2015-12-02-extending-groser-ios-and-rails-apps-to-support-grocery-delivery-in-multiple-cities---part-3 %}) I will extend iOS app so that customers can change which city they are interested in and app will dynamically load `Store` with `Shelves` and `Departments`.