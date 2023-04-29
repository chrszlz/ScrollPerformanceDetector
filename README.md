# ScrollPerformanceDetector
Kyle Sherman's KSScrollPerformanceDetector ([_original blog post_](https://thisiskyle.me/posts/measuring-ios-scroll-performance-is-tough-use-this-to-make-it-simple-and-automated.html))

---

# Measuring iOS scroll performance is tough. Use this to make it simple & automated

## Introduction
iOS Developers who want to create the best experience strive to have a buttery smooth 60 frames-per-second scrolling. There’s nothing more frustrating — both for the user and developer — than having the UI stutter while scrolling. It breaks the illusion of a fluid interface that responds to your every gesture. But now there is an easy way for developers to ensure that scrolling is always nice and smooth. Historically, analyzing scroll performance has been a tedious, manual process that is hard to measure precisely. This process usually involves manually running Instruments on a device, which requires building for profiling (longer build time) and observing the frame rate manually.

I would like to introduce automated scroll performance testing. An important note is that you will need to run these types of tests on a physical device; running on the simulator will not give accurate numbers. In order to get the best data, you should run on an older, slower device that you support to ensure the performance is adequate.

## Analyzing Scroll Performance
I spent a long time analyzing scroll performance and looking at different numbers to understand what criteria can be used to judge the scroll performance of a view controller. The average frame rate is not fine-grained enough to use as a specific measure.

I came up with two metrics that should never exceed a certain threshold. These two numbers combined give you the best picture of scroll performance:

### Number of dropped frames at any instant
Number of times a frame drop event occurs during a single scroll event A large number of frames dropped at a single instant will cause a very noticeable stop in the fluidity of the interface. Too many frame drop events during one scroll will make it seem like the whole scroll event is sluggish even if the number of frames dropped at a time is small. These limits can change based on expectations of scroll performance and the type of views being measured.

### Frame Rate Reporter
Below you will see the code for the class that monitors the frame rate and communicates it back to your code. Don’t be afraid of all the Objective-C code in this article. Everything can be used from Swift. The code hooks into CADisplayLink in order to do this. Using the duration of a frame, we compare the current and previous timestamps and divide by the duration to get the number of frames. Using this information, we can calculate how many frames were dropped. We also keep a running count of how many frames were dropped. These values can be reset at any time.

```objc
#import <Foundation/Foundation.h>
/**
 These methods will not be called on the main thread. So,
 if you will be doing anything with UIKit, ensure you dispatch back
 to the main thread.
 */
@protocol KSScrollPerformanceDetectorDelegate<NSObject>
@optional
- (void)framesDropped:(NSInteger)framesDroppedCount cumulativeFramesDropped:(NSInteger)cumulativeFramesDropped cumulativeFrameDropEvents:(NSInteger)cumulativeFrameDropEvents;
@end
@interface KSScrollPerformanceDetector : NSObject
@property(nonatomic, assign, readonly) NSInteger currentFrameDropCount;
@property(nonatomic, assign, readonly) NSInteger currentFrameDropEventCount;
@property(nonatomic, weak, nullable) id<KSScrollPerformanceDetectorDelegate> delegate;
- (nonnull instancetype)init NS_DESIGNATED_INITIALIZER;
- (void)resume;
- (void)pause;
- (void)clearFrameDropCount;
@end
view raw KSScrollPerformanceDetector.h hosted with ❤ by GitHub
#import "KSScrollPerformanceDetector.h"
#import <UIKit/UIKit.h>
@interface KSScrollPerformanceDetector()
@property(nonatomic, strong, nonnull) CADisplayLink *displayLink;
@property(nonatomic, assign, readwrite) CFTimeInterval lastTimestamp;
@property(nonatomic, assign, readwrite) NSInteger currentFrameDropCount;
@property(nonatomic, assign, readwrite) NSInteger currentFrameDropEventCount;
@property(nonatomic, strong, nonnull) dispatch_queue_t workerQueue;
- (void)displayLinkTriggered:(CADisplayLink *)sender;
- (void)calculateFrameDropForNumberOfFrames:(NSInteger)numberOfFrames;
- (void)registerLifecyleNotifications;
@end
@implementation KSScrollPerformanceDetector
#pragma mark - Initializers
- (instancetype)init {
    self = [super init];
    [self registerLifecyleNotifications];
    _displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkTriggered:)];
    _displayLink.paused = YES;
    [_displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
    _lastTimestamp = 0;
dispatch_queue_attr_t attributes = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, DISPATCH_QUEUE_PRIORITY_HIGH);
    _workerQueue = dispatch_queue_create("com.frame-rate-reporter.worker", attributes);
return self;
}
#pragma mark - Public Methods
- (void)clearFrameDropCount {
    __weak typeof(self) weakSelf = self;
dispatch_async(self.workerQueue, ^{
typeof(self) strongSelf = weakSelf;
if (!strongSelf) {
return;
        }
        self->_currentFrameDropCount = 0;
        self->_currentFrameDropEventCount = 0;
    });
}
- (void)resume {
    self.displayLink.paused = NO;
}
- (void)pause {
    self.displayLink.paused = YES;
}
#pragma mark - Private Methods
#pragma mark Lifecyle
- (void)registerLifecyleNotifications {
    [[NSNotificationCenter defaultCenter] addObserver:self
selector:@selector(notifyActivate:)
name:UIApplicationDidBecomeActiveNotification
object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self
selector:@selector(notifyDeactivate:)
name:UIApplicationWillResignActiveNotification
object:nil];
}
- (void)notifyActivate:(NSNotification *)notification {
    self.displayLink.paused = NO;
    self.lastTimestamp = 0;
}
- (void)notifyDeactivate:(NSNotification *)notification {
    self.displayLink.paused = YES;
}
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    self.displayLink.paused = YES;
    [self.displayLink invalidate];
}
#pragma mark Calculations
- (void)displayLinkTriggered:(CADisplayLink *)sender {
dispatch_async(self.workerQueue, ^{
if (self.lastTimestamp == 0) {
            self.lastTimestamp = sender.timestamp;
return;
        }
CFTimeInterval duration = sender.duration;
if (duration == 0) {
return;
        }
double numberOfFramesDouble = round((sender.timestamp - self.lastTimestamp) / duration);
NSAssert(numberOfFramesDouble <= NSIntegerMax && numberOfFramesDouble >= NSIntegerMin, @"fps double value out of range of NSInteger");
NSInteger numberOfFrames = numberOfFramesDouble;
        self.lastTimestamp = sender.timestamp;
        [self calculateFrameDropForNumberOfFrames:numberOfFrames];
    });
}
- (void)calculateFrameDropForNumberOfFrames:(NSInteger)numberOfFrames {
NSInteger droppedFrameCount = numberOfFrames - 1 > 0 ? numberOfFrames : 0;
    self.currentFrameDropCount += droppedFrameCount;
if (droppedFrameCount > 0) {
        self.currentFrameDropEventCount += 1;
    }
if (droppedFrameCount > 0 &&
        self.listener &&
        [self.delegate respondsToSelector:@selector(framesDropped:cumulativeFramesDropped:cumulativeFrameDropEvents:)]) {
        [self.delegate framesDropped:droppedFrameCount
cumulativeFramesDropped:self.currentFrameDropCount
cumulativeFrameDropEvents:self.currentFrameDropEventCount];
    }
}
@end
```

## Ways To Run Your Test
You can use this class and the associated delegate to either run tests in an automated fashion or run the detector while using your debug app to collect information. You can also choose both.

Other factors can influence your scroll performance that is outside the control of your app. It is important to ensure that there are not many background tasks using up the CPU and GPU while you are running your test. The code below gives you the current CPU and GPU activity. I borrowed this from code that my previous coworker, Benjamin Hendricks. You can use this to check the activity before each test to ensure a reasonable baseline that will not skew your results.

```objc
#ifndef usage_h
#define usage_h

#import <mach/mach.h>

double cpu_usage(int64_t *count);
float mem_usage();

#endif /* usage_h */
view raw usage.h hosted with ❤ by GitHub
#import <mach/mach.h>
#import "usage.h"

double cpu_usage(int64_t *count)
{
    kern_return_t kr;
    task_info_data_t tinfo;
    mach_msg_type_number_t task_info_count;

    task_info_count = TASK_INFO_MAX;
    kr = task_info(mach_task_self(), TASK_BASIC_INFO, (task_info_t)tinfo, &task_info_count);
    if (kr != KERN_SUCCESS) {
        return -1;
    }

    task_basic_info_t      basic_info;
    thread_array_t         thread_list;
    mach_msg_type_number_t thread_count;

    thread_info_data_t     thinfo;
    mach_msg_type_number_t thread_info_count;

    thread_basic_info_t basic_info_th;
    uint32_t stat_thread = 0;
    basic_info = (task_basic_info_t)tinfo;

    kr = task_threads(mach_task_self(), &thread_list, &thread_count);
    if (kr != KERN_SUCCESS) {
        return -1;
    }
    if (thread_count > 0)
        stat_thread += thread_count;

    long tot_sec = 0;
    long tot_usec = 0;
    double tot_cpu = 0;
    uint j;

    for (j = 0; j < thread_count; j++)
    {
        thread_info_count = THREAD_INFO_MAX;
        kr = thread_info(thread_list[j], THREAD_BASIC_INFO,
                         (thread_info_t)thinfo, &thread_info_count);
        if (kr != KERN_SUCCESS) {
            return -1;
        }

        basic_info_th = (thread_basic_info_t)thinfo;

        if (!(basic_info_th->flags & TH_FLAGS_IDLE)) {
            tot_sec = tot_sec + basic_info_th->user_time.seconds + basic_info_th->system_time.seconds;
            tot_usec = tot_usec + basic_info_th->user_time.microseconds + basic_info_th->system_time.microseconds;
            tot_cpu = tot_cpu + basic_info_th->cpu_usage / (double)TH_USAGE_SCALE * 100.0;
        }

    }

    kr = vm_deallocate(mach_task_self(), (vm_offset_t)thread_list, thread_count * sizeof(thread_t));

    if (count) {
        *count = thread_count;
    }
    
    return tot_cpu;
}

float mem_usage()
{
    struct mach_task_basic_info info;
    mach_msg_type_number_t size = MACH_TASK_BASIC_INFO_COUNT;
    kern_return_t kerr = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
    return ( kerr == KERN_SUCCESS ) ? info.resident_size : 0;
}
```

## UI Tests
For UI tests, you need a method that reliably scrolls at a steady speed. A fast scroll may cause certain optimizations in things like UICollectionView or UITableView. Also, you need to know when you have reached the end of the scroll view. I tried using both KIF and XCUITest for this purpose. Unfortunately, XCUITest did not allow me to control the speed. I also could not reliably detect when we hit the bottom of the list. Therefore, KIF was my chosen framework. You can use KIF to scroll through a view until it hits the bottom. As you see in the code below, we reset the counts of the frame rate reporter after each scroll is finished.

```objc
#import <XCTest/XCTest.h>
#import <KIF/KIF.h>

#import "KSScrollPerformanceDetector.h"
#import "usage.h"

// This is specifically chosen to be the right speed to not trigger optimizations
static const CGFloat KSScrollPerformanceTestsVelocity = -0.15f;

@interface ButtonScrollPerformanceTests : KIFTestCase <KSScrollPerformanceDetectorDelegate>

@property(nonatomic, assign, readwrite) NSInteger droppedFramesLimit;
@property(nonatomic, assign, readwrite) NSInteger frameDropEventsLimit;
@property(nonatomic, strong, nonnull) KSScrollPerformanceDetector *scrollPerformanceDetector;
@property(nonatomic, strong, nonnull) UITableView *tableView;

@end

@implementation ButtonScrollPerformanceTests

- (void)beforeEach {
    self.scrollPerformanceDetector = [[KSScrollPerformanceDetector alloc] init];
    self.scrollPerformanceDetector.delegate = self;
    self.droppedFramesLimit = 3;
    self.frameDropEventsLimit = 3;
}

- (void)setUp {
    [super setUp];
    // Put setup code here. This method is called before the invocation of each test method in the class.
    self.continueAfterFailure = YES;
}

- (void)testButtonPerformance {
    [tester tapViewWithAccessibilityLabel:@"Button"];
    [tester tapViewWithAccessibilityLabel:@"Performance"];
    [self scrollTableToEndWithIdentifier:@"performance-test-table"];
}

- (void)scrollTableToEndWithIdentifier:(NSString *)accessibilityIdentifier {
    UITableView *tableView;
    [tester waitForAccessibilityElement:NULL view:&tableView withIdentifier:accessibilityIdentifier tappable:NO];
    self.tableView = tableView;
    NSIndexPath *lastIndexPath = [NSIndexPath indexPathForRow:[self.tableView numberOfRowsInSection:0] - 1 inSection:0];
    NSArray<NSIndexPath *> *visibleIndexPaths = [self.tableView indexPathsForVisibleRows];

    [self.scrollPerformanceDetector resume];
    while (![visibleIndexPaths containsObject:lastIndexPath]) {
        [tester scrollViewWithAccessibilityIdentifier:accessibilityIdentifier byFractionOfSizeHorizontal:0.0f vertical:KSScrollPerformanceTestsVelocity];
        visibleIndexPaths = [self.tableView indexPathsForVisibleRows];
        [self.scrollPerformanceDetector clearFrameDropCount];
    }
    [self.scrollPerformanceDetector pause];
}

- (void)framesDropped:(NSInteger)framesDroppedCount cumulativeFramesDropped:(NSInteger)cumulativeFramesDropped cumulativeFrameDropEvents:(NSInteger)cumulativeFrameDropEvents {
    if (framesDroppedCount >= self.droppedFramesLimit) {
        XCTFail(@"Past limit for number of dropped frames");
    }

    if (cumulativeFrameDropEvents >= self.frameDropEventsLimit) {
        XCTFail(@"Past limit for number of frame drop events");
    }
}

@end
```

In the GIF below you can see what the test looks like when it runs. We are measuring the frame drops from the time a scroll starts until it ends (comes to rest). Then, we reset the counts and measure on the next scroll event. Using this method, we can know exactly when it fails if it does.



## While Running Debug Builds
You can also use the frame rate reporter in your app code. The code should be safe to run for production apps and should be able to make it through app review as no private APIs are being used. Therefore, you could set it up to log any time your frame drop limits are hit. Or, you could just run it in debug builds and assert if your limits are hit. I’d love to hear about the ways you use it for your app.

## Writing Your Test
You can use this framework to test the scroll performance of two different kinds of elements: a view controller’s view, or a single UI component. Measuring scroll performance at a view controller level will quickly verify that the interaction of many different components will not cause any scroll performance degradation. However, if you are building a shared component for use throughout your app, you can catch any issues early on if you test that single component in isolation.

## KIF
In order to write an automated UI test to test scroll performance, you need to set up a test app. If you are testing your app’s view controllers, you will want to ensure you have accessibility identifiers for the controls that will take you to the correct view to test. Also, you’ll need an accessibility identifier on the table or collection view to scroll. If you want to test single views in isolation, you should create a separate test app and create a collection view or table view with many different versions of that view. You will see an easy way to create this in the next section.

## LayoutTest
In order to ensure optimizations of classes like UICollectionView and UITableView are not triggered, you want to make sure that your rows have different data and/or configurations. A great way to ensure this is to use a library called LayoutTest. If you haven’t seen this library, please take a look at what it does. I am a huge fan of this library and was working with the author, Peter Livesey when he was building it. The specific feature of LayoutTest I suggest using for your scroll performance tests is called Catalog. You can learn more about this feature and how to use it by watching this video from my course on LinkedIn Learning on LayoutTest.

LayoutTest’s Catalog feature can be used to write a scroll performance test for a single view like a custom button, a particular type of cell, etc. The Catalog feature creates a view controller with every possible combination of specifications for that view very easily. The result is a perfect sample test app with a long, scrolling view of all of the different ways this view can be laid out. Every edge case of your layout will be tested to ensure it doesn’t degrade scroll performance.

## Thanks
Special thanks to Benjamin Hendricks for writing the code that was the basis for this project. Also, special thanks to Peter Livesey for creating the awesome LayoutTest library that continues to be extremely useful.

I hope you can use this code to help you improve the performance and user experience of your apps! Please let me know your experience by visiting the Contact page of my site.

You can see a gist with all of the code from this article here. Please feel free to use it as is in your own projects.
