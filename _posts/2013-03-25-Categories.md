---
layout: post
title: Categories
---
Objective-C Categories

Categories are an Objective-C feature that allow you to add methods to an already defined class--even one for which you don't have access to the source. Categories enable highly semantic, decomposable programming. Let's explore what categories can do.

###Cleaning up code

Categories have the potential to massively clean up code, and do so in a way that makes testing easy. I was recently working on a data structure for a grouped table view that automatically inserts/deletes/replaces the rows of the table view; here's one method:

    - (void)addSections:(NSArray *)sections
    {
        NSUInteger currentLastSection = [self.sectionObjects count];
        [self.sectionObjects addObjectsFromArray:sections];
        [self.tableView insertSections:[NSIndexSet indexSetWithIndexesInRange:NSMakeRange(currentLastSection, [sections count])]
                      withRowAnimation:UITableViewRowAnimationAutomatic];
    }

An `NSMutableArray` category, `addObjectsFromArray:rangeOfNewObjects:` makes for a clear win here:

    - (void)addSections:(NSArray *)sections rowAnimation:(UITableViewRowAnimation)rowAnimation
    {
        NSRange range;
        [self.sectionObjects addObjectsFromArray:sections
                               rangeOfNewObjects:&range];
        [self.tableView insertSections:[NSIndexSet indexSetWithIndexesInRange:range]
                      withRowAnimation:rowAnimation];
    }
    
Categories are just the classic pattern of subroutines improving code: naming, comprehension, testability -- everything improves. Categories are perfectly suited to this task in an object-oriented language.



###Using categories to enable KVC

I was recently refactoring this code, which took `NSString`s, made them into `NSURL`s, and passed them to a view for display:

    
    NSMutableArray *iconUrls = [[NSMutableArray alloc] initWithCapacity:4];
    
    for (Game *game in self.game.similarGames) {
        [iconUrls addObject:[NSURL URLWithString:game.iconUrl]];
    }
    self.similar.urls = iconUrls;
    

This code sucked. It's verbose and nothing like the English equivalent: "I need the strings made into URLs". It also caused hundreds of crashes: when the `iconUrl` string was `nil`, the app would crash when added to the array.

I refactored this by adding `asURL` as a readonly property on NSString, with this implementation:

    
    -(NSURL *)asURL
    {
        return [NSURL urlWithString:self];
    }
    

 Then, I used Key-Value Coding to access the URLs:

    
    self.similar.urls = [games valueForKeyPath:@"iconURL.asURL"];
    
Perfect. Combining categories with KVC, we have a one-liner that completely eliminates loops and explicit `NSNull`/`nil` checking. This is a bit of a cheat, as what I was using the URLs for already supported `NSNull`. Still, I'd rather use this approach and add a `filterNSNull` category to `NSArray` than use the previous code.

###Readwrite properties in categories

A notable drawback of categories is that they don't allow the programmer to add instance variables. However, by using the Objective-C runtime, you can simulate properties by writing the getter and setter logic using associated objects. This code, which adds a `tag` property to `PFObject`, was in answer to a question on Parse's help forums:
    
    

    @interface PFObject (Tag)
    @property (nonatomic) int tag;
    @end

    @implementation PFObject (Tag)

    static const void * pfObjectTagKey = &pfObjectTagKey;

    - (void)setTag:(int)tag
    {
        objc_setAssociatedObject(self, pfObjectTagKey, [NSNumber numberWithInt:tag], OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

    - (int)tag
    {
        NSNumber *number = objc_getAssociatedObject(self, pfObjectTagKey);
        return [number intValue];
    }

    @end
    
    

    
###Caveat: Duplicate methods (too basic?)

If you add a method in a category with the same name as a method in the original class, the category will override the class's method. If two categories implement a method with the same name, it's non-deterministic which will be used. The obnoxious ramification is that you'll either have to prefix your category methods (ugly) or risk a conflict.

###Backporting methods with +load -- Code for this is long+boilerplate, move it to a gist.

Category methods with the same name as a method in the class will override that method, while category methods themselves will be chosen 

Category methods override their class's methods,

The exception to the rule that categories override the class's methods is `+load`, [which is special cased](http://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html) to be invoked in the class *and* in every category on that class. In addition to swizzling, you can use this feature to backport methods


// Something on answering SO questions with a category.
// Something on how categories make naming easier -- check
// Something on namespacing
// Using +load to backport methods.
// Something on 
// For MXGroupedDataModel -- have first commit be previous way, and the next commit is with categories replacing the methods, and then use history browser to see side-by-side.
// Uh, unrelated, but head/tail properties on NSIndexPath. 

// Maybe show how I added the alignmentEdgeInsets property to UIImage, and then next show how I backported it?