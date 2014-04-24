#XMARTLABS - XLDataLoader 
---------------
Purpose
--------------

XLDataLoader is a library for iOS that facilitates retrieval of large amounts of data coming from internet, handles the storage and retrieval of such data for quick access to them.

The purpose of this library is to encapsulate the pattern that solves XLDataLoader for **reuse** in different scenarios, this allows you to **reduce the amount of code** and do all this work **very easy** and **very fast!**.
 
#####Let's see an example that uses XLDataLoader in iOS 7

![Screenshot of Example](Examples/Images/Cap.gif)

What XLDataLoader does
----------------
* Work with data pagination.
* Present local data to the user, and in parallel make a request to the server.
* Support search.
* Reuse the same pattern in several scenarios.
* Reduce programming time.
* Do all this **very fast!**.

Working with the example
----------------
The model with which we will work is as follows:

![Screenshot of model](Examples/Images/Model.jpg)

A User is associated with many Posts and a Post is associated with one User.

PostsTableViewController
----------------

* Create a subclass of **XLTableViewController**.
* Override the **initWithNibName** method.

```
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil {
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
        
        // Enable the pagination
        self.loadingPagingEnabled = YES;
        
        // Initialize Data Loaders
        [self initializeDataLoaders];
    }
    return self;
}
```

Initialize the data loaders, **local** and **remote**.

```
-(void)initializeDataLoaders {
    [self setLocalDataLoader:[[PostLocalDataLoader alloc] init]];
    [self setRemoteDataLoader:[[PostRemoteDataLoader alloc] init]];
}
```

* Implement the **cellForRowAtIndexPath** method.

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    PostCell *cell = (PostCell *) [tableView dequeueReusableCellWithIdentifier:kCellIdentifier forIndexPath:indexPath];
    
    Post * post = (Post *)[self.localDataLoader objectAtIndexPath:indexPath];
    User * user = post.user;
    
    cell.userName.text = user.userName;
    cell.postDate.text = [self timeAgo:post.postDate];
    cell.postText.text = post.postText;
    
    NSMutableURLRequest* imageRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:user.userImageURL]];
    [imageRequest setValue:@"image/*" forHTTPHeaderField:@"Accept"];
    __typeof__(cell) __weak weakCell = cell;
    [cell.userImage setImageWithURLRequest: imageRequest
                          placeholderImage:[User defaultProfileImage]
                                   success:^(NSURLRequest *request, NSHTTPURLResponse *response, UIImage *image) {
                                       if (image) {
                                           [weakCell.userImage setImage:image];
                                       }
                                   }
                                   failure:^(NSURLRequest *request, NSHTTPURLResponse *response, NSError *error) {
                                   }];

    return cell;
}
```

We get the post with the local data loader, using the **objectAtIndexPath** method.

```
Post * post = (Post *)[self.localDataLoader objectAtIndexPath:indexPath];
```

PostLocalDataLoader
----------------
* Create a subclass of **XLLocalDataLoader**.
* Override the **init** method.

``` 
- (id)init {
    self = [super init];
    if (self) {

        // Post Fetch Request
        NSFetchRequest * fetchRequest = [Post getFetchRequest];
        NSFetchedResultsController * fetchResultController = [[NSFetchedResultsController alloc]
        										        initWithFetchRequest:fetchRequest
                                                      managedObjectContext:[AppDelegate managedObjectContext]                                                                                                
                                                        sectionNameKeyPath:nil
                                                                 cacheName:nil];
        [self setFetchedResultsController:fetchResultController];
    }
    return self;
}
```
In the example the fetch request is sorted in descending order by **postDate**.

```
+ (NSFetchRequest *)getFetchRequest
{
    NSFetchRequest * fetchRequest = [Post fetchRequest];
    fetchRequest.sortDescriptors = @[[NSSortDescriptor sortDescriptorWithKey:@"postDate" ascending:NO]];
    return fetchRequest;
}
```
PostRemoteDataLoader
----------------
* Create a subclass of **XLRemoteDataLoader**.
* Override the **sessionManager** method that returns an **AFHTTPSessionManager** object.
 
```
-(AFHTTPSessionManager *)sessionManager
{
    return [HTTPSessionManager sharedClient];
}
```

* Override the **URLString** method and return the **path** to the service.

```
-(NSString *)URLString
{
    return @"/mobile/posts.json";
}

```

* Override the **parameters** method.

```
-(NSDictionary *)parameters
{
    return @{@"offset": @(self.offset),
             @"limit": @(self.limit)};
}
```
**Offset** and **limit** allow you to retrieve just a portion of the Posts that are generated by the **server**.
If a **limit** count is given, no more than that many Posts will be returned.
**Offset** says to skip that many Posts before beginning to return Posts.

By defoult *self.limit* is 20.

* Override the successulDataLoad method.

```
-(void)successulDataLoad {
    // Change flags
    
    // This flag indicates whether more data is being loaded
    _isLoadingMore = NO;

    // [self fetchedData] contains the data coming from the server
    NSArray * itemsArray = [[self fetchedData] valueOrNil];
    
    // This flag indicates if there is more data to load
    _hasMoreToLoad = !((itemsArray.count == 0) || (itemsArray.count < _limit && itemsArray.count != 0));
    
    [[AppDelegate managedObjectContext] performBlockAndWait:^{
        for (NSDictionary *item in itemsArray) {
            // Creates or updates the Post and the user who created it with the data that came from the server
            [Post createOrUpdateWithServiceResult:item[POST_TAG] saveContext:NO];
        }
        
        // Remove outdated data
        [self removeOutdatedData:itemsArray];
        
        [AppDelegate saveContext];
    }];
    
    // call super
    [super successulDataLoad];
}


#pragma mark - Auxiliary Functions

- (void)removeOutdatedData:(NSArray *)data
{
    // First, remove older data
    NSFetchRequest * fetchRequest = [Post getFetchRequest];
    fetchRequest.fetchLimit = self.limit;
    fetchRequest.fetchOffset = self.offset;
    
    NSError *error;
    NSArray * oldObjects = [[AppDelegate managedObjectContext] executeFetchRequest:fetchRequest error:&error];
    
    NSArray * arrayToIterate = [oldObjects copy];
    
    if (error) {
        DBLog(@"[ERR] %@", error.localizedDescription);
        [error showAlertView];
        return;
    }
    
    for (Post *post in arrayToIterate)
    {
        NSArray *filteredArray = [data filteredArrayUsingPredicate:[NSPredicate predicateWithFormat:@"post.id = %@" argumentArray:@[post.postId]]];
       if (filteredArray.count == 0) {
            // This Post no longer exists
            [[AppDelegate managedObjectContext] deleteObject:post];
        }
    }
}


```

HTTPSessionManager
----------------
* Create a subclass of **AFHTTPSessionManager**. 
* Set the **Server Base URL**.

```
@implementation HTTPSessionManager

// Server Base URL
static NSString * const AFAppDotNetAPIBaseURLString = @"http://obscure-refuge-3149.herokuapp.com";

// Server Base URL for Staging
// static NSString * const AFAppDotNetAPIBaseURLString = @"http://obscure-refuge-3149-staging.herokuapp.com";

+ (instancetype)sharedClient {
    static HTTPSessionManager *_sharedClient = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedClient = [[HTTPSessionManager alloc] initWithBaseURL:[NSURL URLWithString:AFAppDotNetAPIBaseURLString]];
        _sharedClient.securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeNone];
    });
    
    return _sharedClient;
}

@end

```


#Support Search
----------------

UsersTableViewController
----------------
* Create a subclass of **XLTableViewController**.
* Override the **initWithNibName** method.

```
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
        
        // Enable the pagination
        self.loadingPagingEnabled = NO;
        
        // Support Search Controller
        self.supportSearchController = YES;
        
        // Initialize Data Loaders
        [self initializeDataLoaders];
    }
    return self;
}
```

Initialize the data loaders, **local**, **remote**, **search local** and **search remote**.

```
-(void)initializeDataLoaders
{
    [self setLocalDataLoader:[[UserLocalDataLoader alloc] init]];
    [self setRemoteDataLoader:[[UserRemoteDataLoader alloc] init]];
    
    // Search
    [self setSearchLocalDataLoader:[[UserLocalDataLoader alloc] init]];
    [self setSearchRemoteDataLoader:[[UserRemoteDataLoader alloc] init]];
}
```

* Override the **cellForRowAtIndexPath** method.

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserCell *cell = (UserCell *) [tableView dequeueReusableCellWithIdentifier:kCellIdentifier forIndexPath:indexPath];;

    User * user = nil;
    if (tableView == self.tableView){
        user = (User *)[self.localDataLoader objectAtIndexPath:indexPath];
    }
    else{
        user = (User *)[self.searchLocalDataLoader objectAtIndexPath:indexPath];
    }
    
    cell.userName.text = user.userName;
    NSMutableURLRequest* imageRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:user.userImageURL]];
    [imageRequest setValue:@"image/*" forHTTPHeaderField:@"Accept"];
    __typeof__(cell) __weak weakCell = cell;
    [cell.userImage setImageWithURLRequest: imageRequest
                          placeholderImage:[User defaultProfileImage]
                                   success:^(NSURLRequest *request, NSHTTPURLResponse *response, UIImage *image) {
                                       if (image) {
                                           [weakCell.userImage setImage:image];
                                       }
                                   }
                                   failure:^(NSURLRequest *request, NSHTTPURLResponse *response, NSError *error) {
                                   }];
    return cell;
}
```

If the tableView is a subview of **XLTableViewController**, we get the User from the **self.localDataLoader**, if not, then the tableview is a subview of **UISearchResultController** and we get the User from the **self.searchLocalDataLoader**.

```
User * user = nil;
if (tableView == self.tableView){
    user = (User *)[self.localDataLoader objectAtIndexPath:indexPath];
}
else{
    user = (User *)[self.searchLocalDataLoader objectAtIndexPath:indexPath];
}
```

UserLocalDataLoader
----------------
* Create a subclass of **XLLocalDataLoader**.
* Create a local variable to store the **search string**.

```
{
    NSString *_searchString;
}
```

* Override the **init** method.

``` 
- (id)init {
    self = [super init];
    if (self) {
        
        NSFetchRequest * fetchRequest = [User getFetchRequest];
        NSFetchedResultsController * fetchResultController = [[NSFetchedResultsController alloc]           
                                                       initWithFetchRequest:fetchRequest
                                                       managedObjectContext:[AppDelegate managedObjectContext]
                                                         sectionNameKeyPath:nil                                                                                                                                        
                                                         cacheName:nil];
        [self setFetchedResultsController:fetchResultController];
    }
    return self;
}
```

* Refresh the search **predicate** when the **search string** change.

```
-(void)changeSearchString:(NSString *)searchString
{
    _searchString = searchString;
    [self refreshPredicate];
}

- (void)refreshPredicate
{
    [self setPredicate:[User getPredicateBySearchInput:_searchString]];
}
```
The **predicate** searches for the users which their names contains the **search string**.

```
+ (NSPredicate *)getPredicateBySearchInput:(NSString *)search {

    if (![NSString stringIsNilOrEmpty:search]) {
        return [NSPredicate predicateWithFormat:@"userName CONTAINS[cd] %@" , search];
    }
    return nil;
}
```

UserRemoteDataLoader
----------------
* Create a subclass of **XLRemoteDataLoader**.
* Override the **sessionManager** method that returns an **AFHTTPSessionManager** object.
 
```
-(AFHTTPSessionManager *)sessionManager
{
    return [HTTPSessionManager sharedClient];
}
```
* Override the **URLString** method and return the **path** to the service.

```
-(NSString *)URLString
{
    return @"/mobile/users.json";
}

```
* Override the **parameters** method.

```
-(NSDictionary *)parameters
{
	NSString *filterParam = self.searchString ?: @"";
    return @{@"filter" : filterParam,
             @"offset" : @(self.offset),
             @"limit"  : @(self.limit)};
}
```

We spent the **searchString** as a search parameter to the **server**.

* Override the successulDataLoad method.

```
-(void)successulDataLoad {
    // change flags
    
    // This flag indicates whether more data is being loaded
    _isLoadingMore = NO;
    
    // [self fetchedData] contains the data coming from the server
    NSArray * itemsArray = [[self fetchedData] valueOrNil];
    
    // This flag indicates if there is more data to load
    _hasMoreToLoad = !((itemsArray.count == 0) || (itemsArray.count < _limit && itemsArray.count != 0));

    
    [[AppDelegate managedObjectContext] performBlockAndWait:^{
        for (NSDictionary *item in itemsArray) {
            // Creates or updates the User and the user who created it with the data that came from the server
            [User createOrUpdateWithServiceResult:item[USER_TAG] saveContext:NO];
        }
        [self removeOutdatedData:itemsArray];
        [AppDelegate saveContext];
    }];
    
    // call super
    [super successulDataLoad];
}

#pragma mark - Auxiliary Functions

- (void)removeOutdatedData:(NSArray *)data
{
    // First, remove older data
    NSFetchRequest * fetchRequest = [User getFetchRequestBySearchInput:self.searchString];

    fetchRequest.fetchLimit = self.limit;
    fetchRequest.fetchOffset = self.offset;
    
    NSError *error;
    NSArray * oldObjects = [[AppDelegate managedObjectContext] executeFetchRequest:fetchRequest error:&error];
    
    NSArray * arrayToIterate = [oldObjects copy];
    
    if (error) {
        [error showAlertView];
        return;
    }
    
    for (User *user in arrayToIterate)
    {
        NSArray *filteredArray = [data filteredArrayUsingPredicate:[NSPredicate predicateWithFormat:@"user.id = %@" argumentArray:@[user.userId]]];
        if (filteredArray.count == 0) {
            // This User no longer exists
            [[AppDelegate managedObjectContext] deleteObject:user];
        }
    }
}
```


UsersTableWithFixedSearchViewController
----------------
*  Create a subclass of **TableWithFixedSearchViewController**.
*  Override the **initWithNibName** method.

```
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
        [self setEdgesForExtendedLayout:UIRectEdgeNone];
         self.loadingPagingEnabled = NO;
        [self initializeDataLoaders];
    }
    return self;
}
```

Initialize the data loaders, **local**, **remote**, **local search** and **remote search**.

```
-(void)initializeDataLoaders
{
    [self setLocalDataLoader:[[UserLocalDataLoader alloc] init]];
    [self setRemoteDataLoader:[[UserRemoteDataLoader alloc] init]];
    
    // Search 
    [self setSearchLocalDataLoader:[[UserLocalDataLoader alloc] init]];
    [self setSearchRemoteDataLoader:[[UserRemoteDataLoader alloc] init]];
}
```

* Override the **cellForRowAtIndexPath** method.

```
#pragma mark - UITableViewDataSource

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserCell *cell = (UserCell *) [tableView dequeueReusableCellWithIdentifier:kCellIdentifier forIndexPath:indexPath];;
    
    User * user = nil;
    if (tableView == self.tableView){
        user = (User *)[self.localDataLoader objectAtIndexPath:indexPath];
    }
    else{
        user = (User *)[self.searchLocalDataLoader objectAtIndexPath:indexPath];
    }
    
    cell.userName.text = user.userName;
    NSMutableURLRequest* imageRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:user.userImageURL]];
    [imageRequest setValue:@"image/*" forHTTPHeaderField:@"Accept"];
    __typeof__(cell) __weak weakCell = cell;
    [cell.userImage setImageWithURLRequest: imageRequest
                          placeholderImage:[User defaultProfileImage]
                                   success:^(NSURLRequest *request, NSHTTPURLResponse *response, UIImage *image) {
                                       if (image) {
                                           [weakCell.userImage setImage:image];
                                       }
                                   }
                                   failure:^(NSURLRequest *request, NSHTTPURLResponse *response, NSError *error) {
                                   }];
    return cell;
}
```


How it works
----------------
Here is a picture of all these components together:

![Screenshot of diagram](Examples/Images/XLDataLoader.jpg)

Try it with dynamic data
----------------

Go to **HTTPSessionManager** and change the base url to **http://obscure-refuge-3149-staging.herokuapp.com**

```
// Server Base URL for Staging
static NSString * const AFAppDotNetAPIBaseURLString = @"http://obscure-refuge-3149-staging.herokuapp.com";

```
In your browser, go to [http://obscure-refuge-3149-staging.herokuapp.com](http://obscure-refuge-3149-staging.herokuapp.com) and create or upader users and posts, and see how it works.

Installation
--------------------------

The easiest way to use XLDataLoader in your app is via [CocoaPods](http://cocoapods.org/ "CocoaPods").

1. Add the following line in the project's Podfile file:
`pod 'XLDataLoader'`.
2. Run the command `pod install` from the Podfile folder directory.


Requirements
-----------------------------

* ARC
* iOS 7.0 and above


Release Notes
--------------

Version 1.0.0

* Initial release

Contact
----------------

If you are using XLDataLoader in your app and have any suggestion or question:

Martin Barreto, <martin@xmartlabs.com>, [@mtnBarreto](http://twitter.com/mtnBarreto "@mtnBarreto")

[@xmartlabs](http://twitter.com/xmartlabs "@xmartlabs")


