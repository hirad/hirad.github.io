---
layout: post
title: "Ordering Asynchronous Network Calls in Objective-C"
date: 2014-04-17 15:40:42 +0200
comments: true
categories: 
published: false
---

Any iOS app that needs to talk to an API will probably need to control the orders of its network calls. For example, let's say your application needs to ask a server for a list of users and a list of reports. Let's say each report belongs to a user. On the server side, the relationship between users and their reports is maintained through the unique IDs of users and tables. When the server gives you the users and reports that you asked for, it might look something like this:

{% codeblock /api/users.json lang:json %}
[
  {
    id : 1,
    name: "Bob Loblaw",
    email: "bobloblaw@lawblog.com"
  },
  {
    id : 2,
    name: "Johnny Appleseed",
    email:"johnny@me.me"
  }
]
{% endcodeblock %}

{% codeblock /api/reports.json lang:json %}
[
  {
    id : 1,
    title: "Official Report",
    notes: "long lorem ipsum... ",
    score: 8.2,
    user_id : 2
  },
  {
    id : 2,
    title: "Unofficial Report",
    notes: "long lorem ipsum... ",
    score: 4.3,
    user_id : 2
  },
  {
    id : 3,
    title: "Official Report",
    notes: "long lorem ipsum... ",
    score: 7.0,
    user_id : 1
  }
]
{% endcodeblock %}

Note that the reports tell you which user they belong to. This is great if you want to look up users by ID, but we in the Cocoa world have CoreData. We build our data relationship with objects not with their database ID. Let's say we have the following data model in the app:

{% img left /images/datamodel.png 395 143 'datamodel' %}

So if your CoreData app is trying to talk to this API, it must make sure that the results from the reports request are processed AFTER the results from the users request. This way, when you're importing the data from the API, you can find the owner of each report by ID. Then you assign the user object to the `owner` relationship of the newly created report and save the managed object context. The problem is that `AFNetworking`, the defacto standard networking library on iOS, only sends network requests asynchronously. So this won't work:

{% codeblock lang:objc %}
-(void)getUsers
{
  void (^success)(AFHTTPRequestOperation*, id);
  success = ^(AFHTTPRequestOperation* op, id responseObject)
  {
      // create objects with data in responseObject
  };
  // send network request to /api/users.json with this success block
}

-(void)getReports
{
  void (^success)(AFHTTPRequestOperation*, id);
  success = ^(AFHTTPRequestOperation* op, id responseObject)
  {
      // create report objects with data in responseObject
  };
  // send network request to /api/reports.json with this success block
}

-(void)getDataFromServer
{
  [self getUsers];
  [self getReports];
}
{% endcodeblock %}

This doesn't work because the network requests are asynchronous and you have no guarantee that the reports request will get its response before the users request gets its. 

I've seen people pull all sorts of shenanigans to solve this problem:

{% codeblock lang:objc %}
-(void)getUsers
{
  void (^success)(AFHTTPRequestOperation*, id);
  success = ^(AFHTTPRequestOperation* op, id responseObject)
  {
      // create objects with data in responseObject 
  };
  AFHTTPRequestOperation* op = ... ;
  [op waitUntilFinished];
}

-(void)getReports
{
  void (^success)(AFHTTPRequestOperation*, id);
  success = ^(AFHTTPRequestOperation* op, id responseObject)
  {
      // create report objects with data in responseObject
  };
  // send network request to /api/reports.json with this success block
}

-(void)getDataFromServer
{
  [self getUsers];
  [self getReports];
}
{% endcodeblock %}

Not only is this not the recommended way of using AFNetworking, it also fails because the operation spawns its success block on the main thread and then finishes immediately without waiting for the block to finish. This creates a race condition where you might be trying to create reports when the users success block hasn't finished yet. 

The other thing you can do is calling `[self getReports]` inside the success block of the users request. This will work, but it's a recipe for spaghetti code - specially if you have more than two requests. 

Here's a how I solve this problem:

- There is a bitmask enum that assigns a flag to each network opration I have. 
- There is a `NSOperationQueue` that runs only 1 operation at a time.
- Every time an API request comes back, it creates an NSOperation object that is responsible for processing the server response. It notifies the sync controller that it finished, including its own flag and this NSOperation.
- One method in the sync controller handles these notifications and encapsulates all ordering logic.

This is what it'd look like in our example:

{% codeblock lang:objc SyncController.m %}
typedef NS_OPTIONS(NSInteger, HMAPIJob) {
    HMAPIJobNone = 0,
    HMAPIJobGetUsers = 1 << 0,
    HMAPIJobGetReports = 1 << 1,
    HMAPIJobComplete = HMAPIJobGetUsers | HMAPIJobGetReports
};

@interface SyncController ()
@property (nonatomic, assign) HMAPIJob completedJobs;
@property (nonatomic, strong) NSMutableArray* waitingOperations;
@property (nonatomic, strong) NSOperationQueue* operationQueue; // set its maxConcurrentOperations to 1
@end

@implementation

-(void)APIJobCompleted:(HMAPIJob)flag withCompletionOperation:(NSOperation*)operation error:(NSError*)error
{
  // all logic for controlling the order of completion operations goes here.
  if (/* we must wait for prerequisite operations */)
  {
    [self.waitingOperations addObject:operation];
  }
  else
  {
    [self.operationQueue addOperation:operation];
  }

  self.completedJobs = self.completedJobs | flag;
  if (self.completedJobs == HMAPIJobsComplete)
  {
    // take care of remaining operations in waitingOperations
  }
}

-(void)getDataFromServer
{
  self.completedJobs = HMAPIJobNone;
  [self getUsers];
  [self getReports];
}

-(void)getUsers
{
  void (^success)(AFHTTPRequestOperation*, id);
  success = ^(AFHTTPRequestOperation* op, id responseObject)
  {
      NSInvocationOperation* op = [[NSInvocationOperation alloc] initWithTarget:self 
                                                                       selector:@selector(updateUsersWithJSON:)
                                                                         object:responseObject];

      [self APIJobCompleted:HMAPIJobGetUsers 
    withCompletionOperation:op
                      error:nil];
  };

  void (^failure)(AFHTTPRequestOperation*, NSError*);
  failure = ^(AFHTTPRequestOperation* op, NSError* error)
  {
      [self APIJobCompleted:HMAPIJobGetUsers 
    withCompletionOperation:nil
                      error:error];
  };

  // send the request with these blocks... 
}

-(void)updateUsersWithJSON:(NSArray*)serverResponse
{
  // ... 
}

-(void)getReports
{
  // same pattern as getUsers
}

-(void)updateReportsWithJSON:(NSArray*)serverResponse
{
  // ... 
}

@end
{% endcodeblock %}

The advantage of this system is that all the logic for ordering is in one place, and it's very clear. However, one caveat is that even with `maxConcurrentOperations` set to 1, `NSOperationQueue` still creates an extra thread for its operations. You have to make sure whatever you do with `NSManagedObject`s in the update methods is wrapped inside a `performBlock:` call on your managed object context.