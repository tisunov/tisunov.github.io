---
layout: single
title: Extending Groser iOS and Rails apps to support grocery delivery in multiple cities - Part 3
categories: []
tags: []
published: True

---

Right now you can shop and order delivery only in Moscow from single chain of supermarkets. My goal in this post, is to allow customers to switch cities, and load catalog of the supermarket chain we happen to deliver from in the given city.

## Floating Menu Bar

I want to devide floating department selection bar in half and make left part open city selection view controller.

{:.center}
![Floating Menu Bar](/assets/multi-city-3/floating-bar.png){:width="300px"}
*Floating Menu Bar*

I need to move labels to the right, embedd labels and image view into wrapper view and anchor it to the center of the parent view. 

I'll use separator view with auto layout constraints set to center it vertically and horizontally and then will constrain left edge of the department wrapper view to the separator view. Repeat steps for the left half and constrain right edge of the left wrapper view to separator view.

{:.center}
![Floating Menu Bar](/assets/multi-city-3/floating-bar-1.png){:width="500px"}
*City and Department Bar*

## City selection

Tapping on the left part of `FloatingMenuBar` should show available cities in `CitiesViewController`, which I'll create now.

<figcaption>CitiesViewController.swift</figcaption>
{% highlight swift %}
class CitiesViewController: UITableViewController, UISearchBarDelegate {
    private var searchBar: UISearchBar!
    private var cities: [GRCity] = []
    static let CellIdentifier = "cityCell"
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Выбрать Город Доставки"
        tableView.backgroundColor = GRTheme.tableViewBackgroundColor()
        
        loadCities()
    }
    
    func loadCities() {
        GRGroserService.shared().loadCitiesWithCompletion {
            [weak self] (cities: AnyObject!, error: NSError!) -> Void in
            if let strongSelf = self {
                strongSelf.cities = (error == nil) ? cities as! [GRCity] : []
                
                strongSelf.tableView.reloadData()
            }
        }
    }
    
    // MARK: - UITableViewDataSource
    
    override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return cities.count
    }
    
    override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCellWithIdentifier(CitiesViewController.CellIdentifier)!
        cell.textLabel?.text = cities[indexPath.row].name
        return cell
    }
    
    override func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
        let city = cities[indexPath.row]
        
        NSNotificationCenter.defaultCenter().postNotificationName(kCityChangedNotification, object: nil, userInfo: ["city": city])
        
        navigationController?.popViewControllerAnimated(true)
    }
}
{% endhighlight %}

Hook up `FloatingMenuBar` with `CitiesViewController`.

<figcaption>GRFloatingMenuBar.m</figcaption>
{% highlight objective-c %}

- (void)handleCityTap:(UITapGestureRecognizer *)recognizer {
    if (recognizer.state != UIGestureRecognizerStateEnded) return;
    
    if ([self.delegate respondsToSelector:@selector(selectCity)]) {
        [self.delegate selectCity];
    }
}

{% endhighlight %}

<figcaption>GRStoreViewController.m</figcaption>
{% highlight objective-c %}
...

- (void)subscribeForNotifications {
    ...
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(cityChanged:)
                                                 name:kCityChangedNotification
                                               object:nil];
}

...

#pragma mark - GRDepartmentsMenuDelegate

- (void)selectCity {
    CitiesViewController *controller = [[CitiesViewController alloc] init];
    [self.navigationController pushViewController:controller animated:YES];
}

...
{% endhighlight %}

Update city and store labels in `FloatingMenuBar` when city changes.

<figcaption>GRFloatingMenuBar.m</figcaption>
{% highlight objective-c %}
- (void)subscribeForStoreChanges {
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(cityChanged:)
                                                 name:kCityChangedNotification
                                               object:nil];
}

- (void)cityChanged:(NSNotification *)notif {
    GRCity *city = notif.userInfo[@"city"];
    self.cityLabel.text = city.name;
    self.storeLabel.text = city.defaultStore.name;
}

{% endhighlight %}

## Change current city and load default store

Let's download default city's `Store` with shelves and departments.

<figcaption>GRStoreViewController.m</figcaption>
{% highlight objective-c %}

- (void)cityChanged:(NSNotification *)notif {
    GRCity *city = notif.userInfo[@"city"];

    [GRGroser shared].city = city;
    
    [self loadContent];
}

- (void)loadContent {
    self.departments = @[];
    
    [self.collectionView reloadData];
    
    [self startActivityIndicator];
    
    @weakify(self)
    [[GRGroser shared] loadStore].then(^(GRStore *store) {
        @strongify(self)
        self.departments = store.departments;
        
        [self stopActivityIndicator];
        
        [self reloadData];
         
        self.collectionView.scrollEnabled = YES;
        
        [self startCoaching];
    }).catch(^(NSError *error) {
        @strongify(self)
        [self loadContentError:error];
    });
}

{% endhighlight %}

## Save and restore selected city

To simplify things I'll store selected `GRCity` object into `NSUserDefaults`. To do that I'll need to conform `GRCity` and `GRStore` to `NSCoding` protocol.

<figcaption>GRCity.m</figcaption>
{% highlight objective-c %}

- (void)encodeWithCoder:(NSCoder *)encoder {
    [encoder encodeObject:self.uid forKey:@"uid"];
    [encoder encodeObject:self.name forKey:@"name"];
    [encoder encodeObject:self.stores forKey:@"stores"];
}

- (id)initWithCoder:(NSCoder *)decoder {
    if((self = [super init])) {
        self.uid = [decoder decodeObjectForKey:@"uid"];
        self.name = [decoder decodeObjectForKey:@"name"];
        self.stores = [decoder decodeObjectForKey:@"stores"];
    }
    return self;
}

{% endhighlight %}

<figcaption>GRStore.m</figcaption>
{% highlight objective-c %}

- (void)encodeWithCoder:(NSCoder *)encoder {
    [encoder encodeObject:self.uid forKey:@"uid"];
    [encoder encodeObject:self.name forKey:@"name"];
}

- (id)initWithCoder:(NSCoder *)decoder {
    if((self = [super init])) {
        self.uid = [decoder decodeObjectForKey:@"uid"];
        self.name = [decoder decodeObjectForKey:@"name"];
    }
    return self;
}

{% endhighlight %}

Now I can save and restore selected `GRCity` in `GRGroser`.

<figcaption>GRGroser.m</figcaption>
{% highlight objective-c %}

- (void)loadLastSelectedCity {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    NSData *data = [defaults objectForKey:@"selectedCity"];
    if (data) {
        _city = [NSKeyedUnarchiver unarchiveObjectWithData:data];
    }
}

- (void)setCity:(GRCity *)city {
    _city = city;
    
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    [defaults setObject:[NSKeyedArchiver archivedDataWithRootObject:city] forKey:@"selectedCity"];
    [defaults synchronize];
}

{% endhighlight %}

{:.center}
![Switching Cities](/assets/multi-city-3/switch-cities.gif){:width="300px"}
*Switching Cities*


In [Part 4]({% post_url 2015-12-03-extending-groser-ios-and-rails-apps-to-support-grocery-delivery-in-multiple-cities---part-4 %}) I will ask customers to select their delivery city or allow app to detect their city using Location Services.