---
layout: post
title: "Creating View Controllers for Deep Linked iOS Apps"
author: "Ben Roesch"
---
While building [55Prophets][1] for iOS (you can download it [here][2]), I wanted to allow deep linking into the app. For example, I wanted users to be able to share links to specific questions. When someone accessed the link, I wanted the app to take them directly to that question, instead of dropping them into the home screen that appears on a normal app launch. However, the question page is typically several layers deep into a navigation stack. So how do you link directly into a view controller like this?  
  
To accomplish this, first you need to register a url scheme for your application. Apple provides documentation on how to do this [here][3]. Once you set this up, an iOS device with your app installed will launch your app any time a user touches a link with your url scheme (e.g. myappurlscheme://leagues/123).  
  
To centralize deep linking, I created a class called FFDeepLinker that would process all links into the app. This class contains a singleton instance that can be used to process links. The FFDeepLinker.h file looks something like this:  
  
> `@interface FFDeepLinker : NSObject  @property (nonatomic, strong)
> UIWindow *appWindow;  @property (nonatomic, strong)
> NSManagedObjectContext
> *managedObjectContext;  +(instancetype)sharedLinker;  +(void)setSharedLinker:(FFDeepLinker
> *)linker;  -(void)handleUrl:(NSURL *)url;@end`

  
I then set up the shared singleton instance in the app delegate\'s `- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions` method. The setup looks something like this:  
  
> `FFDeepLinker *deepLinker = [[FFDeepLinker alloc]
> init];deepLinker.appWindow =
> self.window;deepLinker.managedObjectContext = // the main core data
> context containing the app's data;[FFDeepLinker
> setSharedLinker:deepLinker];`

  
When your app is launched via one of these types of links, it will automatically call `-(BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation` in your app delegate. In this method, I simply call `[[FFDeepLinker sharedLinker] handleUrl:url];` to process the url.  
  
Inside FFDeepLinker is where the magic happens. There, I keep a dictionary that contains url pattern regular expressions as the keys and a corresponding method name as the value. It looks something like this:  
  
> `+(NSDictionary *)deepLinkPatterns{  return @{   
> @"//leagues/(\\\\d+)/join" : @"handleJoinLeagueURL:", 
>   @"//leagues/(\\\\d+)/questions/(\\\\d+)" :
> @"handleShowQuestionURL:",    @"//leagues/(\\\\d+)/comments" :
> @"handleShowLeagueCommentsURL:",         
>   @"//leagues/(\\\\d+)/questions/(\\\\d+)/review" :
> @"handleShowReviewQuestionURL:"  };}`

  
Each of the entries in this dictionary corresponds to one type of potential deep link that the app handles. The `-(void)handleUrl:(NSURL *)url` method cycles through the regular expressions and checks for a match against the url that we are opening. If it finds a match, it captures the variable path components (typically ID\'s of resources) and calls the corresponding method. It looks like this:  
> `-(void)handleUrl:(NSURL *)url{  NSString *linkString = [url
> resourceSpecifier];  NSError *error = NULL;  for (NSString *pattern in
> [[FFDeepLinker deepLinkPatterns] allKeys]) {    NSRegularExpression
> *regex = [NSRegularExpression regularExpressionWithPattern:pattern    
> options:NSRegularExpressionCaseInsensitive error:&error];  
>  NSTextCheckingResult *result = [regex firstMatchInString:linkString
> options:NSMatchingAnchored                                       
> range:NSMakeRange(0, linkString.length)];    if (result) {   
>   NSMutableArray *captures = [NSMutableArray array];      for (int
> i=1; i<result.numberOfRanges; i++) {        NSRange range = [result
> rangeAtIndex:i];        NSString *capture = [linkString
> substringWithRange:range];        [captures addObject:capture];    } 
>   NSString *selector = [[FFDeepLinker deepLinkPatterns]
> objectForKey:pattern];    [self
> performSelector:NSSelectorFromString(selector) withObject:captures]; 
>   return;  }}`

  
So, if we open a url with the pattern //leagues/123/questions/456, it will match to //leagues/(\\\\\\\\d+)/questions/(\\\\\\\\d+) and call the method `-(void)handleShowQuestionURL:(NSArray *)capturedComponents` with an array containing the strings \"123\" and \"456\". Within `-(void)handleShowQuestionURL:(NSArray *)capturedComponents`, I then load the league with ID=123 and question with ID=456 out of core data, create the appropriate series of view controllers, and present the nav stack to the user:  
> `MembershipsViewController *membershipsVC =
> [self.appWindow.rootViewController.storyboard
> instantiateViewControllerWithIdentifier:@"MembershipsViewController"];LeagueTabBarController
> *leagueVC = [self.appWindow.rootViewController.storyboard
> instantiateViewControllerWithIdentifier:@"LeagueTabBarController"];leagueVC.membership
> = membership;QuestionsViewController *questionsVC =
> [self.appWindow.rootViewController.storyboard
> instantiateViewControllerWithIdentifier:@"QuestionsViewController"];questionsVC.membership
> = membership;AnswersViewController *answersVC =
> [self.appWindow.rootViewController.storyboard
> instantiateViewControllerWithIdentifier:@"AnswersViewController"];answersVC.question
> = question;answersVC.membership = membership;UINavigationController
> *navController = [[UINavigationController alloc] init];[navController
> setViewControllers:@[membershipsVC, leagueVC, questionsVC, answersVC]
> animated:NO];if(self.appWindow.rootViewController.presentedViewController
> != nil){  [self.appWindow.rootViewController
> dismissViewControllerAnimated:NO
> completion:^{}];}[self.appWindow.rootViewController
> presentViewController:navController animated:NO completion:^{}];`

  
Now, users can have links directly to questions, and the normal navigation back-buttons will be preserved and operate as expected.  
  
P.S. If you enjoyed this post, feel free to hit me up on twitter [@bcroesch][4]. 

[1]: http://55prophets.com
[2]: https://itunes.apple.com/us/app/55prophets/id622424094?ls=1&amp;mt=8
[3]: http://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/AdvancedAppTricks/AdvancedAppTricks.html#//apple_ref/doc/uid/TP40007072-CH7-SW50
[4]: http://twitter.com/bcroesch
