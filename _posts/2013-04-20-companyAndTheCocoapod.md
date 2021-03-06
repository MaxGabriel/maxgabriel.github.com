---
layout: post
title: The Company and the Cocoapod
---

### Notice

This blog post is oudated—please refer to [the Cocoapods Podspecs guide](http://guides.cocoapods.org/syntax/podspec.html#group_file_patterns), which now includes native support for vendored frameworks, libraries, and bundles. 

### *Companies should make a Cocoapod for their commercial SDKs*

Cocoapods is a dependency manager for iOS and OS X projects that makes it easy to integrate 3rd party software. It's just like Ruby Gems, right down to being written in Ruby. The vast majority of open-source Objective-C projects have been made into Cocoapods, but unfortunately many commercial libraries have not been (or else are maintained by their users, rather than the companies themselves). I urge companies to make a Cocoapod for their SDK.

### You should support Cocoapods

Supporting Cocoapods requires creating a Podspec, which is a small file describing information about your code, like what version it is and what source files compose it. Briefly, here's why you should make one (details on actually making one below):

1. Developers really like using Cocoapods -- so much that they've maintained pods for commercial SDKs like [Parse](https://github.com/CocoaPods/Specs/tree/master/Parse), [Facebook](https://github.com/CocoaPods/Specs/tree/master/Facebook-iOS-SDK), and [Chartboost](https://github.com/CocoaPods/Specs/tree/master/ChartboostSDK). 
2. Easier for developers to integrate. No linking frameworks or setting compiler flags. 
3. Updating is manageable: Before Cocoapods, updating an SDK wasn't worth it if it wasn't deeply-integrated into the app. Cocoapods makes it easy to update -- and it changes the dependency workflow. With Cocoapods, it's natural to keep all your dependencies up-to-date with `pod install`, whereas before it required an explicit decision to update an SDK. If you ever add new features or make important bug fixes to your SDK, you should care about this.

<br>
### You should document header files using Appledoc

When you install a pod, Cocoapods automatically uses Appledoc to generate documentation for it. Documenting with Appledoc is very simple: just add slight markup to comments in your header files. The generated documentation looks and functions just like Apple's and is automatically added to Xcode by Cocoapods. Why you should use it:

1. Having API-level documentation is extremely helpful for devs, full-stop.
2. Devs that don't use Appledoc or Cocoapods will still benefit from the header file comments.
3. Writing documentation improves API quality. Once you're documenting each method, you'll notice when changing an argument name would make the method's function clearer, or when an edge case hasn't been considered (e.g. *can this argument be `nil`?)*. (Unfortunately, for that benefit you really have to start early, since changing selector and property names breaks backwards compatibility). 
4. [Cocoadocs](cocoadocs.org). Cocoadocs is a sister-project of Cocoapods, and it's freaking awesome. It runs Appledoc on every version of each Cocoapod, generates documentation for them, and hosts them all at [CocoaDocs.org](http://cocoadocs.org/). Just by making a podspec and documenting your header files, your SDK now has great looking API-level documentation for every version of your SDK, equaling API docs like those at [Parse](https://parse.com/docs/ios/api/) or [Facebook](https://developers.facebook.com/docs/reference/ios/3.0/) -- and you don't have to do anything to support it. 

I've made [code snippets](https://gist.github.com/MaxGabriel/5385841) for the three things you'll need to document: classes, methods, and properties. (Unfortunately, Appledoc doesn't yet support [functions, enums, or constants](https://github.com/tomaz/appledoc/issues/2).) You can read more about Appledoc [at its website](http://gentlebytes.com/appledoc/) and [on Github](https://github.com/tomaz/appledoc).

####Making a Cocoapod for closed-source projects

To support Cocoapods, you'll need to make a standard Podspec:

{% highlight bash %}
$ gem install cocoapods # May require sudo
$ pod setup
$ pod spec create Bugsnag
$ edit Bugsnag.podspec
{% endhighlight %}


Making a Cocoapod for open-source projects is straightforward. If you just scan down the keys on the left you can see that you could almost make the Podspec from memory:

{% highlight ruby %}
Pod::Spec.new do |s|
  s.name         = "Bugsnag"
  s.version      = "2.2.3" 
  s.summary      = "iOS notifier for SDK for bugsnag.com."
  s.homepage     = "https://bugsnag.com"
  s.license      = 'MIT'
  s.author       = { "Bugsnag" => "notifiers@bugsnag.com" }
  s.source       = { :git => "https://github.com/bugsnag/bugsnag-ios.git", :tag => "2.2.3" }
  s.platform     = :ios, '4.3'
  s.source_files = ['Bugsnag Plugin', 'Bugsnag Plugin/Categories']
  s.requires_arc = true
  s.public_header_files = 'Bugsnag Plugin/Bugsnag.h'
  s.framework  = 'SystemConfiguration' # Framework dependency
  s.dependency 'Reachability', '~> 3.1' # Another Cocoapod
end
{% endhighlight %}


All the details of the Podspec are beyond the scope of this blog post, but you can check out each attribute of a Podspec in detail on [Cocoapods' documentation](http://docs.cocoapods.org/specification.html). I haven't read it, but Lars Anderson seems to have a [pretty comprehensive guide](http://theonlylars.com/blog/2013/01/20/cocoapods-creating-a-pod-spec/), too.


If your company distributes its [SDK already compiled](https://keen.io/docs/clients/iOS/usage-guide/), but your source code is [available on Github](https://github.com/keenlabs/KeenClient-iOS), I highly recommend making the [Cocoapod from that version](https://github.com/CocoaPods/Specs/blob/master/KeenClient/3.2.0/KeenClient.podspec). Not only will making the Podspec be simpler for you, but because Cocoapods handles the integration it will negate any advantage of precompiled code and also show the source. It will be smaller because it doesn't have to be compiled for multiple architectures, and it will be more resilient: you don't want your SDK [causing compiler errors during the rush to support a new iPhone](http://stackoverflow.com/questions/12402092/file-is-universal-three-slices-but-it-does-not-contain-an-armv7-s-slice-err). 
	
But if you are distributing a closed source project, it's not too difficult. Just upload the latest version of your framework/library/bundle to a public Github repo. I recommend keeping all the files at the top-level since closed-source projects expose so few files.

The only difference from regular Podspecs will be these types of files:

#### Bundles
{% highlight ruby %}
s.resources = 'Heyzap.bundle' # Just like regular images.
{% endhighlight %}

<br>
#### Frameworks
{% highlight ruby %}
s.frameworks 	 = 'Heyzap'
s.xcconfig       = { 'FRAMEWORK_SEARCH_PATHS' => '"$(PODS_ROOT)/Heyzap"' }
s.preserve_paths = 'Heyzap.framework'
s.source_files 	 = 'Heyzap.framework/Headers/*.{h}'
{% endhighlight %}

If you don't include the header files, things will still work, but they won't be visible from Xcode or have Appledoc documentation generated for them.

#### Libraries
{% highlight ruby %}
s.library         = 'GoogleAdMobAds'
s.xcconfig        =  { 'LIBRARY_SEARCH_PATHS' => '$(PODS_ROOT)/AdMob' }
s.preserve_paths  = 'libGoogleAdMobAds.a'
{% endhighlight %}

<br>

#### Check the correctness of your Podspec
{% highlight bash %}
$ pod spec lint *.podspec
{% endhighlight %}


If you run into any trouble, I recommend cloning the specs repo and just referencing someone else's Podspec.

#### Publish

Fork the [Cocoapods Specs repo](https://github.com/CocoaPods/Specs), add your new Podspec, and make a pull request. After your Pull Request is approved Cocoapods will offer push access for easy updating. [Details are on their wiki](https://github.com/CocoaPods/CocoaPods/wiki/Contributing-to-the-master-repo#sharing-podspecs).

Creating a Cocoapod for your library is easy and will be a big convenience to developers. You can start making it right now: `gem install cocoapods`

####Postscript

It's important that open-source projects not have their 'documentation' scattered across outdated blog posts. Consequently, I've added the instructions for Libraries, Frameworks, and Bundles as a [Cocoapods Guide](http://docs.cocoapods.org/guides/index.html). The guide has been merged into their repo, but as of April 28th it hasn't been deployed yet.


**Thanks to** James Smith of Bugsnag for reviewing a draft of this post.