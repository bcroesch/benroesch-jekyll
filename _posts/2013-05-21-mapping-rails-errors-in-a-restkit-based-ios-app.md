---
layout: post
title: "Mapping Rails Errors in a Restkit-based iOS App"
author: "Ben Roesch"
---
We used [RestKit][1] to build [55Prophets][2] (available in the [app store][3]). Restkit\'s readme page has some instructions for mapping error responses, but I ran into several problems along the way, so I came up with my own setup for taking errors that come out of our back end and presenting them to a user.  
  
To start, I created a class called ErrorCollection. It is relatively simple and the header file looks like this:  
> `// ErrorCollection.h#import @interface ErrorCollection :
> NSObject@property (nonatomic, strong) NSArray *messages;@property
> (nonatomic, strong) NSString *message;-(NSString
> *)messagesString;@end`

  
  
The implementation is also simple. It looks like this:  
> `// ErrorCollection.m#import "ErrorCollection.h"@implementation
> ErrorCollection  -(NSString *)messagesString{    if (self.message) { 
>     return self.message;    }    else{      NSMutableString *str =
> [[NSMutableString alloc] init];      for (int i=0;
> i<self.messages.count; i++) {           NSString *msg = [self.messages
> objectAtIndex:i];        if (i==self.messages.count-1) {          [str
> appendFormat:@"%@", msg];        }         else {          [str
> appendFormat:@"%@\\n", msg];        }      }              return str; 
>   }  }@end`

  
<span>Our back-end rails app either returns json in the form of `{"errors": ["error1", "error2" ... ] }` or `{"error": "error message" }`. In the case of the former, we want to map each error string in the array to a string in the messages array of an ErrorCollection. In the case of the latter, we want to map the string to the message property of the ErrorCollection. Then, we can use messagesString method, which will give us a single string to present to the user, containing the error message or messages. </span>  
  
To get RestKit to use the ErrorCollection class, we need to add a response descriptor for it. Wherever you set up your response descriptors, add something like this:  
> `RKObjectMapping *errorMapping = [RKObjectMapping
> mappingForClass:[ErrorCollection class]];[errorMapping
> addPropertyMapping:[RKAttributeMapping
> attributeMappingFromKeyPath:@"errors"
> toKeyPath:@"messages"]];[errorMapping
> addPropertyMapping:[RKAttributeMapping
> attributeMappingFromKeyPath:@"error"
> toKeyPath:@"message"]];[[RKObjectManager sharedManager]
> addResponseDescriptor:[RKResponseDescriptor
>   responseDescriptorWithMapping:errorMapping pathPattern:nil
> keyPath:nil
>   statusCodes:RKStatusCodeIndexSetForClass(RKStatusCodeClassClientError)]];`

  
Now you can access the error collection in the failure block many of the methods of the RestKit object manager. For example:  
> `[[RKObjectManager sharedManager] postObject:someObject
> path:@"/object_path" parameters:nil 
> success:^(RKObjectRequestOperation *operation, RKMappingResult
> *mappingResult){  // handle success}failure:^(RKObjectRequestOperation
> *operation, NSError *error){  ErrorCollection *errors = [[[error
> userInfo] objectForKey:RKObjectMapperErrorObjectsKey] lastObject]; 
> NSLog(@"%@", [errors messagesString]);}];`



[1]: http://restkit.org/
[2]: http://55prophets.com
[3]: https://itunes.apple.com/us/app/55prophets/id622424094?ls=1&amp;mt=8
