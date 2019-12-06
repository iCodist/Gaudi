<p align="center">
<img src="./GaudiLogo.png" alt="MERLin"/>
</p>

**Gaudí** is a framework for theme management in UIKit. It allows to easily swap themes in runtime, revert theming applied through `UIAppearance` proxies.

Gaudí also provides a DSL for `UIAppearance` rules and `NSAttributedString`.

This framework uses semantic colors names to better adapt to [dark mode](https://developer.apple.com/documentation/xcode/supporting_dark_mode_in_your_interface) and other possible themes living within the same app.

# What's wrong with explicit color names?

Nothing, except that reading `red`, one would expect as result a color that is a shade of red, while reading `primary` there are no expectations.

This framework aims to make theming easy. If you are using a `black` color for a text it would be strange to see `black` actually be rendered as a white color in a dark mode theme. For this reason i decided to adopt Apple recommendations about using Semantic colors, not only to support Dark Mode, but also to allow different themes to work together maintaining a layer of semantic abstraction from the theme color and the actual rendered color.

## Semantic colors

Work with your designer to get these right. Like apple recommends don't take shortcuts and don't change the semantic meaning of the semantic color. **Gaudí**'s `SemanticColor` enum provides a clear hint about what that color atually is:

```swift
public enum SemanticColor: CaseIterable {
    case label(LabelColor)
    case fill(FillColor)
    case background(BackgroundColor)
    case groupedBackground(GroupedContentBackgroundColor)
    case separator(SeparatorColor)
}
```

Each one of these `LabelColor`, `FillColor`, `BackgroundColor`, `GroupedContentBackgroundColor` have different specific semantic color such as `primary`, `secondary`, `tertiary` and so on.

Don't use a `LabelColor` as a fill color. This will introduce entropy in your project. Work closely with your designer to adhere to this specification. When in your code you will be using just `SemanticColor`s in the correct way, to re-skin your app will be as easy as change 20 lines of code. You will also be able to A/B test the different theme by creating a new theme object with the new colors.

# How to create a theme

Creating a theme is as simple as creating a class conforming the protocol `ThemeProtocol`.

```swift
public protocol ThemeProtocol: class {
    var appearanceRules: AppearanceRuleSet { get }
        
    // MARK: Colors
    
    func color(forSemanticColor semanticColor: SemanticColor) -> UIColor
    
    // MARK: Fonts
    
    func font(forStyle style: FontStyle) -> UIFont
    func fontSize(forStyle style: FontStyle) -> CGFloat
    func kern(forStyle style: FontStyle) -> CGFloat
}
```

The only requirements for the `ThemeProtocol` are a mapping function from `SemanticColor` to `UIColor` and the equivalent mapping functions for `FontStyle`.

## Appearance Rule Set

An appearance rule set is a set of appearance rules obtained by using `UIAppearance` proxies.

```swift
AppearanceRuleSet {
    AppearanceRuleSet {
        UINavigationBar[\.barTintColor, self.color(forSemanticColor: .background(.primary))]
        PropertyAppearanceRule<UINavigationBar, UIColor?>(keypath: \.tintColor, value: self.color(forColorPalette: .primary))
        UINavigationBar[\.titleTextAttributes, [
            .font: self.font(forStyle: .caption(attribute: .regular)),
            .foregroundColor: self.color(forSemanticColor: .label(.primary))
            ]
        ]
    }
            
    AppearanceRuleSet {
        UITabBar[\.barTintColor, self.color(forSemanticColor: .background(.primary))]
        UITabBarItem[\.badgeColor, self.color(forSemanticColor: .fill(.primary))]
        UITabBarItem[
            get: { $0.titleTextAttributes(for: .selected) },
            set: { $0.setTitleTextAttributes($1, for: .selected) }
            value: [
                NSAttributedString.Key.font: self.font(forStyle: .caption(attribute: .regular)),
                NSAttributedString.Key.foregroundColor: self.color(forSemanticColor: .label(.primary))
            ]
        ]
    }
}
```

This is an appearance rule set that customize the appearance of all the navigation bars, all the tab bars and tab bar items of the app. The DSL allows to create a rule by using `KeyPath` to the customizabe property of the `UIAppearance` object.

Appearance Rule Sets are reversible. This means that you can revert your theme to default settings in runtime. 

If you don't need Global Appearance for yur theme you can use the `.empty` appearance rule set.

## Setting up the theme
Once your Theme object is created, you are ready to use it. Assign your Theme to the ThemeContainer in your AppDelegate.

```swift
func application(_ application: UIApplication,
                   didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    ThemeContainer.currentTheme = YourTheme()
}
```

**Gaudí** provides many UIKit extensions to easily access colors and fonts, and to easily configure labels, buttons and Strings (`NSAttributedString`). For example to setup a title label you can use

```swift
label.applyLabelStyle(.title(.regular), semanticColor: .label(.primary))
```

This will change the font (and size) and the color for the text of the `UILabel`. To obtain a color for a semantic color you can use the UIColor extension: `UIColor.semanticColor(.fill(.primary))`

