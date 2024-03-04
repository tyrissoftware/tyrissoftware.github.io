---
layout: post
title: "Introducing UFont 🖋️"
date: 2024-03-04
categories: ios library swift apple fonts
---

# Introducing [UFont](https://github.com/tyrissoftware/swift-ufont): a model for describing fonts

UIKit, AppKit and SwiftUI all have their own font representations, and they all have significant limitations. They aren't multiplatform, they are opaque, they can't be serialized or a combination of those. We introduce UFont: a simple model to describe fonts that doesn't have any of the previous limitations.

## The problem

If you need to develop a multiplatform app (for one of Apple's platforms, that is) you'll find issues when trying to deal with custom fonts and themes. If you are using UIKit or AppKit, you'll find two incompatible representations of fonts (UIFont and NSFont), each only supported in the respective platform. One way of managing this is to use a typealias:

{% highlight swift %}
#if os(iOS)
typealias AppFont = UIFont
#else if os(macOS)
typealias AppFont = NSFont
#endif
{% endhighlight %}

This is not a great solution, however. UIFont and NSFont don't share the exact functions and properties and you'll need to use platform conditionals in many cases.

SwiftUI somewhat alleviates this by providing a *Font* type that can be used in all platforms, but this is an opaque type: we can't see or change any of its properties (size, font family, etc), and we can't serialize it.

## The solution

UFont is a simple struct. It contains a font family and a size, and it is Equatable, Hashable and Codable. You can create a font and later change its size, switch font families, … And when you are ready to use it, you can create a UIFont, a NSFont or a SwiftUI.Font from it.

## Installation

Use it with swift package manager:

```swift
.package(url: "https://github.org/tyrissoftware/swift-ufont.git", from: "0.1.1")
```

## Usage

Define a struct to hold all the fonts in your theme:

{% highlight swift %}
struct ThemeFont {
	var title1: UFont
	var title2: UFont
	[…]
}
{% endhighlight %}

If you are using SwiftUI, you can conform to EnvironmentKey so you can easily access your fonts in any view:

{% highlight swift %}
extension ThemeFont: EnvironmentKey {
	public static let defaultValue = ThemeFont()
}

extension EnvironmentValues {
	public var fonts: ThemeFont {
		get {
			return self[ThemeFont.self]
		}
		set {
			self[ThemeFont.self] = newValue
		}
	}
}

struct MyView: View {
	@Environment(\.fonts.title1.swiftUI) var font
	var body: some View {
		Text("Hello, World!").font(font)
	}
}
{% endhighlight %}


Review the documentation or the tests for a better understanding on how to use it, and all the utilities that come with it. [Check it now](https://github.com/tyrissoftware/swift-ufont).
