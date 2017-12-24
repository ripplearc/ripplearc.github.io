---
layout: default 
title: Build a scrolling photo gallery with UICollectionViewFlowLayout
---
`UICollectionView` was introduced in iOS 6 to display the content in a much more flexible way. Compared to `UITableView`, not only does `UICollectionView` provide simliar interfaces of `datasource` and `delegate` to configure its layout, but there are much more customization options to go beyond the list or grid view thanks to **UICollectionViewFlowLayout**. The final visual effect is only limited by a developer's artistic imagination and quantitative reasoning. 

# 1. Overview
Check out the `Photos` native app on iOS, you will find out the main interface is composed of two horizontal scroll views. The bigger one displays the full resolution copies while the smaller displays the thumbnails. The scrolling motion on either one of them is reflected on the other one. Look closer as you scroll photos in the bigger scroll view, you may notice more portion of the photo is revealed as it being slided in. The same image also presents the accordion animation effect on the smaller scroll view. 

In this lab, we will build the main interface of `Photos` using two `UICollectionViews`. To be more specific, we will customize the **UICollectionViewFlowLayout** associated with each `UICollectionView` to decide the `size` and `center` of the items that are visible on the screen. The changes on these attributes create the `reveal` and `accordion` animation. 

<img src="/images/iOS/UI/ScrollingAlbum/scroll.gif" width="288" height="420" />


## What you will learn
:checkered_flag: Set up the basic datasource and delegate of UICollectionView
:checkered_flag: Override methods in UICollectionViewFlowLayout to achieve customized layout.
:checkered_flag: Synchronize between two UICollectionViewFlowLayout  

## What you'll need
- XCode 9.0 and above
- The sample code
- Familiar with Swift language
- Familiar with UICollectionView datasource and delegate
- Familiar with Interface Builder
- Exposed to Protocol Oriented Programming methodology


# 2. Get the sample code

:octocat: Clone the repository to your local computer, and switch to the `bootcamp` branch
```git
git clone git@github.com:ripplearc/AutoSizeTableViewCell.git
git checkout bootcamp
```
Take a look at the project file structure: we will be mainly working on the `view` :file_folder:; `model` :file_folder: provides some mock data captured from TechCrunch for display; `uiview` :file_folder: contains protocol extensions to make the code in UITableViewController cleaner.

Build and run, it should show a blank UITableView.

<img src="/images/iOS/UI/AutoSizeUITableViewCell/project_file_structure.png" alt="Drawing" style="width: 300px;"/>


# 3. Basics about UIStackView

:green_apple: [Apple's official documentation](https://developer.apple.com/documentation/uikit/uistackview) provides an excellent interpretation of the design philosophy of **UIStackView**. UIStackView layouts its **arrangedSubviews** through its properties of `axis`, `distribution`, `alignment`, and `spacing`. Distribution decides how arrangedSubviews are laid out along the axis, and alignment decides how they are laid out perpendicular to the axis. Spacing are the exact or the minimum intervals betwween the subviews.

While there are five enumerations in distribution, **fill** is the default one and probably the most useful one. It instructs the interface builder to fill up the entire space along the axis with specified spacing. If there is too much space to be filled, then whichever has the lower `Hugging Priority` will be stretched. On the contrary, if there is too little space to accomodate all the arrangedSubviews, then whichever has the lower `Compression Priority` will be compressed. 

If there is no width or height constraint on a horizontal or vertical UIStackView respectively, then UIStackView decides the total height of itself based on the sum of the **IntrinsicContentSize** of its arrangedSubviews, such as UILabel, UIButton or UIImageView.

:key:
>  The key to building an autosizing UITableViewCell is to allow the widgets with **IntrinsicContentSize**, whose content size could change, to decide their own height. To put that in a simple term, don't set the height constraint on a widget whose height could change at runtime. 

In terms of the UITableViewCell, the size of the width is fixed, and we let the UIStackView grows vertically based on the widgets' IntrinsicContentSize, which frees the developer from calcuating the size of UILabel using `sizeThatFits`.

We do not have to worry that a UITableViewCell grows out of spaces along the vertical axis, therefore, it is unnecessary to set hugging and compression priority of the arrangedSubviews in a column. However, since the width is fixed, we will see an example where we can leverage hugging and compression. 

<img src="/images/iOS/UI/AutoSizeUITableViewCell/fixed_width_grow_vertically.png" alt="Drawing" style="width: 400px;"/>


# 4. Layout using IntrinsicContentSize

In this section, we will build most of the UITableViewCell except the UIImageView. As you may notice, the majority of the arrangedSubviews are UILabels where we will have IntrinsicContentSize to decide their sizes. 

- Drag a `vertical UIStackView` underneath the `Content View` and name it as `Content Container` by pressing `Return` on a Mac. Add `New Constraints` on all four sides.  
- Click `Content Container`, go to `Attributes Inspector`, set `Spacing` to 20.

:memo:
> It is a good habit to name the components on the interface builder, then the error messages about missing or conflict constraints are more meaningful. 
<div class="image123">
    <img src="/images/iOS/UI/AutoSizeUITableViewCell/constraints_2_content_view_1.png"  height="220" style="float:left">
    <img src="/images/iOS/UI/AutoSizeUITableViewCell/constraints_2_content_view_2.png" height="220">
</div>  

- Drag one `UILabel` and two `horizontal UIStackView` underneath the `Content Container`, and name them `Category Label`, `Body Container` and `Footer Container` respectively.
- Drag one `UILabel` underneath `Body Container` and name it `Title Label`. Set the `Alignment` to `Center`.
- Drag two `UILabel` and one `UIView` underneath `Footer Container`, and name them `Author Label`, `Time Label` and `Dot View` respectively. Set the `Alignment` to `Center`, and `Spacing` to 20.
- Set the height and width constraint of `Dot View` to 3.
- Set the `Hugging Priority` of `Time Label` to 249 in `Size Inspector`, this allows the UIStackView to stretch `Time Label` when there is extra space as it has lower Hugging Priority than `Author Label` and `Dot View`.
- Set placeholder texts of `UILabels`.

<img src="/images/iOS/UI/AutoSizeUITableViewCell/constraints_2_content_container.png" alt="Drawing" style="width: 600px;"/>

This completes all the interface builder setup :dart:. Connect the widgets from the interface builder to their corresponding `IBOutlets`, go to `ListTableViewCell.swift` and populate the widgets with the model.

```swift
@IBOutlet weak var categoryLabel: UILabel!
@IBOutlet weak var titleLabel: UILabel!
@IBOutlet weak var timeLabel: UILabel!
@IBOutlet weak var authorLabel: UILabel!

func configure(with model:ListTableViewCellModel) {
    categoryLabel.text = model.category.uppercased()
    titleLabel.text = model.title
    authorLabel.text = model.author
    timeLabel.text = model.timeStamp
}
```
Go to `ListViewController.swift` and configure the tableView to auto sizing in the `viewDidLoad`. `estimatedRowHeight` is nothing but a guess.
```
tableView.estimatedRowHeight = 200
tableView.rowHeight = UITableViewAutomaticDimension
tableView.separatorColor = .clear
```
Run the app in both `SE` and `8 Plus` :tada: 

<img src="/images/iOS/UI/AutoSizeUITableViewCell/techcrunch_table_view_uilabel.png" alt="Drawing" style="width: 300px;"/>

1. The UITableViewCell now accommodates `Title Label` with proper height given any number of lines.
2. `Author Label` and `Time Label` are also sized with proper width while `Author Label` pushes `Time Label` towards the right edge when its content grows. 

:octocat: If you have any difficulty in setting things up in the interface builder, commit all the changes and shift to the branch `uilabel`
```git
git add --all
git commit -m 'setting up labels'
git checkout uilabel
```

# 5 UIImageView

We will place the UIImageView side by side with the `Title Label`, and let whichever has the greater height to decide the height the UITableViewCell. In addtion, we want bigger image when the screen size increases. 

- Drag one `UIImageView` underneath the `Body Container`, and name it `Cover Image`. Set `Spacing` of `Body Container` to 20.
- Ctrl click `Cover Image`, drag it to `Body Container` and select `Equal Width`. 
- Select the constraint been created and go to the `Size Inspector`, change `Multiplier` to `1:3`. This way the UIImageView will always occupy 1/3 of the `Body Container`. 
- Ctrl click `Cover Image`, drag to itself and select `Aspect Ratio`.
- Select the constraint and change `Multiplier` to `3:2`.
- Change the `Alignment` of the `Body Container` to `Center`. This allows the taller widget between `Title Label` and `Cover Image` to decide the height of `Body Container`.

<img src="/images/iOS/UI/AutoSizeUITableViewCell/uiimageview_constraint.png" alt="Drawing" style="width: 600px;"/>

:memo:
> At this point, the interface builder should be free of errors. If you still have one, and you are certain you have followed all the steps, move the cursor to the bottom of the cell, and manually adjust the height of the cell until the error disappears. 

Connect the IBOutlet and populate the `Cover Image`.

```swift
...
@IBOutlet weak var coverImage: UIImageView!
...
func configure(with model:ListTableViewCellModel) {
...
if let image = model.imageName {
    coverImage.isHidden = false
    coverImage.image = UIImage(named:image)
} else {
    coverImage.isHidden = true
}
...
}
```
If you are interested, you can also add a bookmark UIButton on the left upper corner, and a separator line. However those are not the focus of the lab. 

Run the app in both `SE` and `8 Plus` :tada: 

<img src="/images/iOS/UI/AutoSizeUITableViewCell/uiimageview_proper_height.png" alt="Drawing" style="width: 300px;"/>

:octocat: You can also shift to the `master` branch.
```git
git add --all
git commit -m `add uiimageview`
git checkout master
```

The `Bitcoin` one shows the case where the `Cover Image` decides the height of the cell, and the `Automotive GM` one shows that the `Title Label` can also push the height of the cell. The `Title Label` in the `Automotive Cruise` news occupies the entire cell width as the `Cover Image` is absent. The hiding behavior is taken care of by UIStackView. 


# 6 Summary

We conclude the lab with an anatomy drawing of the layout, and highlights a few important settings. Other than the `Dot View` and `Cover Image` whose height are pre-defined by constant or aspect ratio, the rest of the widgets possess no height constraint and subject to their own `IntrinsicContentSize`.

<img src="/images/iOS/UI/AutoSizeUITableViewCell/layout_overview.png" alt="Drawing" style="width: 600px;"/>

## What we've learned 
:white_check_mark: Use `IntrinsicContentSize` to decide the height of `UILabel`  
:white_check_mark: Use `Center Alignment` to let the tallest widget to decide the height of the UIStackView  
:white_check_mark: Let UIStackView to handle the hiding behavior of the widget  
:white_check_mark: Automatic sizing of the `UITableViewCell`  

## References
1. [Apple Developer Documentation UIStackView](https://developer.apple.com/documentation/uikit/uistackview) 
It explains the a few common ways to pin the UIStackView relative to its superview.
2. [Self-sizing Table View Cells](https://www.raywenderlich.com/129059/self-sizing-table-view-cells) A very good example to build a gallery app where its UITableViewCell self sizes. 
3. [UIStackView Demystified](https://www.raizlabs.com/dev/2016/04/uistackview/) Illustration about the distribution, alignment, spacing and axis properties of UIStackView.




