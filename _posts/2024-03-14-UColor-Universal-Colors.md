---
layout: post
title: "Introducing UColor 🍭"
date: 2024-03-14
categories: ios library swift apple colors uikit appkit
---

# Introducing [UColor](https://github.com/tyrissoftware/swift-ucolor): a model for describing colors

UIKit, AppKit and SwiftUI all have their own color representations, and they all have significant limitations. They aren't multiplatform, they are opaque, they can't be serialized or a combination of those. We introduce UColor: a simple model to describe fonts that doesn't have any of the previous limitations.

This library is very similar to [UFont](https://github.com/tyrissoftware/swift-ufont), with the same design and goals.

## The problem

If you need to develop a multiplatform app (for one of Apple's platforms, that is) you'll find issues when trying to deal with colors. If you are using UIKit or AppKit, you'll find two incompatible representations of fonts (UIColor and NSColor), each only supported in the respective platform. One way of managing this is to use a typealias:

{% highlight swift %}
#if os(iOS)
typealias AppColor = UIColor
#else if os(macOS)
typealias AppColor = NSColor
#endif
{% endhighlight %}

This is not a great solution, however. UIColor and NSColor don't share the exact functions and properties and you'll need to use platform conditionals in many cases.

SwiftUI somewhat alleviates this by providing a *Color* type that can be used in all platforms, but this is an opaque type: we can't see or change any of its color components, and we can't serialize it.

## The solution

UColor is a simple struct. It contains the color components and color space, and it is Equatable, Hashable and Codable. You can create a color and later change its size, switch color families, … And when you are ready to use it, you can create a UIColor, a NSColor or a SwiftUI.Color from it.

## Installation

Use it with swift package manager:

```swift
.package(url: "https://github.org/tyrissoftware/swift-UColor.git", from: "0.1.3") // Check the latest version here
```

## Usage

Define a struct to hold all the colors in your theme:

{% highlight swift %}
struct ThemeColor {
	var background: UColor
	var foreground: UColor
	[…]
}
{% endhighlight %}

If you are using SwiftUI, you can conform to EnvironmentKey so you can easily access your colors in any view:

{% highlight swift %}
extension ThemeColor: EnvironmentKey {
	public static let defaultValue = ThemeColor()
}

extension EnvironmentValues {
	public var Colors: ThemeColor {
		get {
			return self[ThemeColor.self]
		}
		set {
			self[ThemeColor.self] = newValue
		}
	}
}

struct MyView: View {
	@Environment(\.colors.background) var background
	var body: some View {
		background
	}
}
{% endhighlight %}


[Review the documentation or the tests](https://github.com/tyrissoftware/swift-ucolor) for a better understanding on how to use it, and all the utilities that come with it.
