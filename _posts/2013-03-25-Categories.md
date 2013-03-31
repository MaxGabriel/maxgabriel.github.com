---
layout: post
title: Categories
---
Objective-C Categories

Categories are an Objective-C feature that allow you to add methods to an already defined class--even one for which you don't have access to the source.

While I occasionally hear vague complaints about categories, I think they enable highly semantic, decomposable programming. Let's explore what categories can do.

A recent login screen design I implemented called for a navigation bar button to push to a signup view controller. But in the case where the user got to the login page *from* signup, it needed to pop backwards to the signup controller. Here was my first approach:
    {% highlight objective-c %}
    int controllersCount = self.navigationController.viewControllers.count;
    if (controllersCount >= 2) {
	    if ([[self.navigationController.viewControllers objectAtIndex:controllersCount-2] isKindOfClass:SignupViewController.class] {
	        [self.navigationController popViewControllerAnimated:YES];
        } else {
            // Custom transition method
        }
    } else {
        // Custom transition method
    }
    {% endhighlight %}


This is my refactor using categories:
    {% highlight objective-c %}
    if ([self.previousViewController isKindOfClass:SignupViewController.class]) {
        [self.navigationController popViewControllerAnimated:YES];
    } else {
        //Custom transition method
    }
    {% endhighlight %}

Much better. This code isn't prone to off-by-one errors, is more semantic, more concise, and doesn't instantiate any temporary variables. Here's the category backing it:
    
    @interface UIViewController (NavigationStack)
    @property (nonatomic, readonly) UIViewController *previousViewController;
    @end
    @implementation UIViewController (NavigationStack)
    - (UIViewController *)previousViewController
    {
        return [self.navigationController viewControllerPreviousTo:self];
    }
    @end
    
    
Which is in turn backed by this `UINavigationController` category:
    {% highlight objective-c %}
    - (UIViewController *)viewControllerPreviousTo:(UIViewController *)viewController
    {
        if (![self.viewControllers containsObject:viewController]) {
            return nil;
        } else if (self.rootViewController == viewController) {
            return nil;
        } else {
            return self.viewControllers[[self.viewControllers indexOfObjectIdenticalTo:viewController]-1];
        }
    }
    {% endhighlight %}

This code is documentable, testable, and reusable. It keeps logic about the navigation stack outside of a view controller subclass responsible for logging in users. 


I'm moving more of my logic into categories. I was recently refactoring this code, which took `NSString`s, made them into `NSURL`s, and passed them to a view for display:
    {% highlight objective-c %}
    NSMutableArray *iconUrls = [[NSMutableArray alloc] initWithCapacity:4];
    
    for (Game *game in self.game.similarGames) {
        [iconUrls addObject:[NSURL URLWithString:game.iconUrl]];
    }
    self.similar.urls = iconUrls;
    {% endhighlight %}

This code sucked. It's verbose and nothing like the English equivalent: "I need the strings made into URLs". It also caused hundreds of crashes: when the `iconUrl` string was `nil`, the app would crash when added to the array.

I refactored this by adding `asURL` as a readonly property on NSString, with this implementation:
    {% highlight objective-c %}
    -(NSURL *)asURL
    {
        return [NSURL urlWithString:self];
    }
    {% endhighlight %}

 Then, I used Key-Value Coding to access the URLs:
    {% highlight objective-c %}
    self.similar.urls = [games valueForKeyPath:@"iconURL.asURL"];
    {% endhighlight %}
Perfect. Combining categories with KVC, we have a one-liner that completely eliminates loops and explicit `NSNull`/`nil` checking. 



A notable drawback of categories is that they don't allow the programmer to add instance variables. However, by using the Objective-C runtime, you can simulate properties by writing the getter and setter logic using associated objects. This code, which adds a `tag` property to `PFObject`, was in answer to a question on Parse's help forums:

    @interface PFObject (Tag)
    @property (nonatomic) int tag;
    @end

    @implementation PFObject (Tag)

    static char pfObjectTagKey

    - (void)setTag:(int)tag
    {
        objc_setAssociatedObject(self, &pfObjectTagKey, [NSNumber numberWithInt:tag], OBJC_ASSOCIATION_RETAIN);
    }

    - (int)tag
    {
        NSNumber *number = objc_getAssociatedObject(self, &pfObjectTagKey);
        return [number intValue];
    }

    @end



// Something on answering SO questions with a category.