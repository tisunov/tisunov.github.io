---
layout: single
title: Extending Groser iOS and Rails apps to support grocery delivery in multiple cities - Part 4
categories: []
tags: []
published: True

---

In this part I will modify sign up flow and include one more step, by asking user to detect or manually select his delivery city.

Sign up will start with social signup options using Facebook, VKontakte and email. Proceed to explain Location Services permission request. 

In order to enhance user's first app experience, I want to convince him to allow access to Location Services. After that I will reverse geocode his coordinates to get delivery city name. If he declines Location Services access, I will ask user to select his delivery city from list of active delivery cities.

I decided to use city names to differentiate between delivery zones instead of postal codes, cause in Russia we rarely use them, and few recall them right away.

Each city will be devided into delivery zones, either to vary delivery fee or exclusively split city between delivery partners.

Initially I wanted to ask user his street address (last screen in mockup) to check if his address inside active delivery zone. But decided to keep sign up flow short and check delivery availablility upon entering delivery address, after they decided to place an order. 

On one side short signup flow should help with signup rate, but I anticipate frustration from some clients, who happen to be in the active city, but outside of supported delivery zones after they've spent time assembling carts.

{:.center}
![Mockup Sign Up Flow](/assets/multi-city-4/signup-flow.jpg){:width="800px"}
*Mockup of Sign Up Flow*

## Location Services pre-permission

Now, let's create `LocationPermissionController` to teach user why location services access is required. View controller will be shown right after User proceeds to with Facebook, VKontakte or Email option. Text says "Groser uses your location information to show stores near you from where delivery is possible". 

User can either allow or skip, in which case he'll be prompted with system dialog, or will be transfered to `CitiesViewController` to select his city manually.

<figcaption>LocationPermissionController.swift</figcaption>
{% highlight swift %}
class LocationPermissionController: UIViewController {    
    var didDetectLocation:((GRLocation?, selectedCity:GRCity?)->())?

    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Make UINavigationBar transparent
        navigationController?.navigationBar.setBackgroundImage(UIImage(), forBarMetrics: UIBarMetrics.Default)
        navigationController?.navigationBar.shadowImage = UIImage()
        navigationController?.navigationBar.translucent = true
    }
    
    @IBAction func skipButtonTapped(sender: AnyObject) {
        let controller = CitiesViewController()
        controller.didSelectCity = { [weak self](city: GRCity!) in
            self?.didDetectLocation?(nil, selectedCity:city)
        }
        navigationController?.pushViewController(controller, animated: true)
    }
}
{% endhighlight %}

{:.center}
![LocationPermissionController.xib](/assets/multi-city-4/location-permission.jpg){:width="500px"}
*LocationPermissionController.xib*

## Reverse geocode user location to get city name

I'll use my existing class `GRCityLocator` to request location and do reverse geocoding using Google API.

<figcaption>LocationPermissionController.swift</figcaption>
{% highlight swift %}
private let cityLocator = GRCityLocator()

@IBAction func allowButtonTapped(sender: AnyObject) {
    showGlobalHUDWithTitle("Поиск Города")
    cityLocator.locateCityCompletionHandler({ [weak self](location: GRLocation?) in
        self?.hideGlobalHUD()
        // location will be nil if user denied access to his location,
        // there was an error detecting location or reverse geocoding error
        self?.didDetectLocation?(location, selectedCity:nil)
    })
}

{% endhighlight %}

Below is `GRCityLocator` code along with `GRLocation`. It uses excellend [INTULocationManager](https://github.com/intuit/LocationManager) from Intuit to request current location with timeout. 

<figcaption>GRCityLocator.m</figcaption>
{% highlight objective-c %}

@interface GRCityLocator ()
@property (nonatomic, strong) GRLocation *currentLocation;
@property (nonatomic, strong) GRGoogleGeocoder *geocoder;
@end

@implementation GRCityLocator {
    GRCityLocatorCompletionHandler _completionHandler;
}

- (void)locateCityCompletionHandler:(GRCityLocatorCompletionHandler)completionHandler
{
    _completionHandler = [completionHandler copy];

    [self findUserLocationAndReverseGeocodeIt];
}

- (GRGoogleGeocoder *)geocoder {
    if (_geocoder == nil) {
        _geocoder = [[GRGoogleGeocoder alloc] init];
    }
    return _geocoder;
}

- (void)findUserLocationAndReverseGeocodeIt {
    if (self.currentLocation) {
        _completionHandler(self.currentLocation);
        return;
    }
    
    INTULocationManager *locMgr = [INTULocationManager sharedInstance];
    [locMgr requestLocationWithDesiredAccuracy:INTULocationAccuracyBlock
                                       timeout:8.0
                          delayUntilAuthorized:NO
                                         block:^(CLLocation *currentLocation, INTULocationAccuracy achievedAccuracy, INTULocationStatus status) {
                                             if ( status == INTULocationStatusSuccess ||
                                                  (status == INTULocationStatusTimedOut && currentLocation != nil) )
                                             {
                                                 [[NSNotificationCenter defaultCenter] postNotificationName:kCurrentLocationUpdatedNotification object:nil userInfo:@{ @"location": currentLocation }];
                                                 
                                                 [self reverseGeocode:currentLocation];
                                             }
                                             else {
                                                 _completionHandler(nil);
                                             }
                                         }];
}

- (void)reverseGeocode:(CLLocation *)currentLocation {
    @weakify(self)
    [self.geocoder reverseGeocodeLocation:currentLocation.coordinate
                        completionHandler:^(GRLocation *location, NSError *_) {
                            @strongify(self)
                            self.currentLocation = location;
                            _completionHandler(location);
                        }];
}

{% endhighlight %}

Quick Google API geocoder wrapper.

<figcaption>GRGoogleGeocoder.m</figcaption>
{% highlight objective-c %}
 
#import "GRGoogleGeocoder.h"
#import "AFHTTPRequestOperation.h"
#import "AFURLResponseSerialization.h"
#import "AFURLRequestSerialization.h"
#import "AFHTTPRequestOperationManager.h"

@implementation AFHTTPRequestOperationManager (TimeoutCategory)

- (AFHTTPRequestOperation *)GET:(NSString *)URLString
                     parameters:(NSDictionary *)parameters
                timeoutInterval:(NSTimeInterval)timeoutInterval
                        success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:@"GET" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:nil];
    [request setTimeoutInterval:timeoutInterval];
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithRequest:request success:success failure:failure];
    [self.operationQueue addOperation:operation];
    
    return operation;
}

@end

@implementation GRGoogleGeocoder {
    AFHTTPRequestOperationManager *_manager;
}

#if TARGET_IPHONE_SIMULATOR
NSString * const kSensorParam = @"false";
#else
NSString * const kSensorParam = @"true";
#endif


-(id)init {
    self = [super init];
    if (self) {
        _manager = [AFHTTPRequestOperationManager manager];
        _manager.responseSerializer = [AFJSONResponseSerializer serializer];
    }
    return self;
}

- (void)reverseGeocodeLocation:(CLLocationCoordinate2D)location completionHandler:(GRGoogleGeocoderCompletionHandler)completionHandler {
    NSLog(@"Reverse geocode lat=%f, lon=%f", location.latitude, location.longitude);
    
    [_manager.operationQueue cancelAllOperations];
    
    [_manager GET:@"http://maps.googleapis.com/maps/api/geocode/json"
       parameters:@{@"latlng": [NSString stringWithFormat:@"%f,%f", location.latitude, location.longitude],
                    @"sensor": kSensorParam,
                    @"language": @"ru"}
          success:^(AFHTTPRequestOperation *operation, NSDictionary *responseObject) {
              NSArray *results = [responseObject objectForKey:@"results"];
              NSString *status = [responseObject objectForKey:@"status"];
              BOOL isStatusOk = [status isEqualToString:@"OK"];
              if (!isStatusOk || !results) {
                  NSLog(@"Google geocoder failed with status: %@", status);
                  
                  completionHandler(nil, [NSError errorWithDomain:@"com.brightstripe.groser" code:1000 userInfo:NULL]);
                  return;
              }

              GRLocation *loc = [[GRLocation alloc] initWithReverseGeocoderResponse:responseObject latitude:location.latitude longitude:location.longitude];
              
              completionHandler(loc, nil);
          }
          failure:^(AFHTTPRequestOperation *operation, NSError *error) {
              if (!operation.isCancelled) {
                  NSLog(@"Google geocoder failed: %@", error);
                  
                  completionHandler(nil, error);
              }
          }
    ];
}

@end
{% endhighlight %} 

I parse Google geocoder response and extract only important bit like street address and city in `GRLocation` class.

<figcaption>GRLocation.m</figcaption>
{% highlight objective-c %}

#import "GRLocation.h"

@interface GRLocation()
@property (nonatomic, copy) NSString *streetAddress;
@property (nonatomic, copy) NSString *region;
@property (nonatomic, copy) NSString *city;
@property (nonatomic, copy) NSString *locationType;
@end

@implementation GRLocation {
    NSString *_countryLong;
    NSString *_postalCode;
    NSString *_area;
    NSString *_streetName;
    NSString *_streetNameLong;
    NSString *_streetNumber;
}

-(id)initWithGoogleAddress:(NSDictionary *)address {
    self = [super init];
    if (self) {
        [self parseAddress:address];
    }
    return self;
}

-(id)initWithReverseGeocoderResponse:(NSDictionary *)response latitude:(double)latitude longitude:(double)longitude {
    self = [super init];
    if (self) {
        self.latitude = @(latitude);
        self.longitude = @(longitude);
        
        [self parseReverseGeocodeResults:[response objectForKey:@"results"]];
    }
    return self;
}

-(id)initWithCoordinate:(CLLocationCoordinate2D)coordinate {
    self = [super init];
    if (self) {
        _latitude = @(coordinate.latitude);
        _longitude = @(coordinate.longitude);
    }
    return self;
}

- (void)parseAddress:(NSDictionary *)address
{
    NSArray *addressComponents = address[@"address_components"];
    
    NSDictionary *location = address[@"geometry"][@"location"];
    _latitude = location[@"lat"];
    _longitude = location[@"lng"];

    for (NSDictionary *component in addressComponents) {
        [self parseAddressComponent:component components:addressComponents];
    }
    
    [self formatAddress];
}

-(void)parseReverseGeocodeResults:(NSArray *)results
{
    NSDictionary *firstAddress = [results firstObject];
    self.locationType = firstAddress[@"geometry"][@"location_type"];
    NSArray *addressComponents = firstAddress[@"address_components"];
    
    for (NSDictionary *component in addressComponents) {
        if (component[@"types"]) {
            [self parseAddressComponent:component components:addressComponents];
        }
    }
    
    [self formatAddress];
}

-(void)formatAddress {
    if (_streetNumber.length != 0)
        _streetAddress = [NSString stringWithFormat:@"%@, %@", _streetName, _streetNumber];
    else
        _streetAddress = _streetName;
}

-(void)parseAddressComponent:(NSDictionary *)component components:(NSArray *)addressComponents {
    NSString *componentType = [component[@"types"] firstObject];
    if (![componentType isEqualToString:@"street_number"]) {
        if (![componentType isEqualToString:@"route"]) {
            if (![componentType isEqualToString:@"sublocality"]) {
                // locality indicates city
                if (![componentType isEqualToString:@"locality"]) {
                    if (![componentType isEqualToString:@"administrative_area_level_1"]) {
                        if (![componentType isEqualToString:@"postal_code"]) {
                            if ([componentType isEqualToString:@"country"]) {
                                _countryLong = [component objectForKey:@"long_name"];
                            }
                        }
                        else
                            _postalCode = [component objectForKey:@"long_name"];
                    }
                    else
                        _region = [component objectForKey:@"long_name"];
                }
                else
                    _city = [component objectForKey:@"long_name"];
            }
            else
                _area = [component objectForKey:@"long_name"];
        }
        // route
        else {
            _streetName = [component objectForKey:@"short_name"];
            _streetNameLong = [component objectForKey:@"long_name"];
        }
    }
    else {
        _streetNumber = [component objectForKey:@"short_name"];
        
        // route
        NSDictionary *routeComponent = [addressComponents objectAtIndex:1];
        if (routeComponent) {
            _streetName = [routeComponent objectForKey:@"short_name"];
            _streetNameLong = [routeComponent objectForKey:@"long_name"];
        }
    }
}

-(CLLocationCoordinate2D)coordinate {
    return CLLocationCoordinate2DMake([self.latitude doubleValue], [self.longitude doubleValue]);
}

-(NSString *)formattedAddressWithCity:(BOOL)includeCity country:(BOOL)includeCountry {
    NSString *address;
    
    if (_streetNumber.length != 0)
        address = [NSString stringWithFormat:@"%@, %@", _streetNameLong, _streetNumber];
    else
        address = _streetNameLong.length != 0 ? _streetNameLong : _streetAddress;
    
    if (includeCity && _city.length != 0)
        address = [address stringByAppendingFormat:@", %@", _city];
    
    if (includeCountry && _countryLong.length != 0)
        address = [address stringByAppendingFormat:@", %@", _countryLong];
    
    return address ? address : @"";
}

@end

{% endhighlight %}

## Sign up flow with Location Services enabled

{:.center}
![Facebook Signup with Location Services](/assets/multi-city-4/sign-up-facebook.gif){:height="500px"}
*Signup with Facebook and Location Services to detect city*

## Sign up flow with manual city selection

{:.center}
![Facebook Signup with Manual City Selection](/assets/multi-city-4/sign-up-facebook-manual.gif){:height="500px"}
*Signup with Facebook and manual City Selection*

That's it. All this work just to switch between cities :-). I'll be pushing new code to production this week and we'll begin deliveries in Saint Petersburg early January 2016.