---
layout: default 
title: Build a scrolling photo gallery with UICollectionViewFlowLayout
---
`UICollectionView` was introduced in iOS 6 to display the content in a much more flexible way. Compared to `UITableView`, not only does `UICollectionView` provide simliar interfaces of `dataSource` and `delegate` to configure its layout, but there are much more customization options to go beyond the list or grid view thanks to **UICollectionViewFlowLayout**. The final visual effect is only limited by a developer's artistic imagination and quantitative reasoning. 

# 1. Overview
Check out the `Photos` native app on iOS, you will find out the main interface is composed of two horizontal scroll views. The bigger one displays the full resolution copies while the smaller displays the thumbnails. The scrolling motion on either one of them is reflected on the other. Look closer as you scroll the photos, the bigger scroll view renders the photo with parallax effect, and the same image also presents the accordion animation effect in the smaller scroll view. 

In this lab, we will build the main interface of `Photos` using two `UICollectionViews`. To be more specific, we will customize the **UICollectionViewFlowLayout** associated with each `UICollectionView` to decide the `size` and `center` of the items that are visible on the screen. The changes on these attributes create the `parallax` and `accordion` animation. 

<img src="/images/iOS/UI/ScrollingAlbum/scroll.gif" width="288" height="420" />


## What you will learn
:checkered_flag: Set up the basic datasource and delegate of UICollectionView  
:checkered_flag: Override methods in UICollectionViewFlowLayout to achieve customized layout  
:checkered_flag: Implement parallax animation   
:checkered_flag: Implement accordion animation  
:checkered_flag: Synchronize between two UICollectionViewFlowLayout  

## What you'll need
- XCode 9.0 and above
- The sample code
- Familiar with Swift language
- Familiar with UICollectionView datasource and delegate
- Comfortable with Protocol Oriented Programming methodology


# 2. Get the sample code

:octocat: Clone the repository to your local computer, and switch to the `bootcamp` branch
```git
git clone git@github.com:ripplearc/ScrollingAlbum.git
git checkout bootcamp
```
Build and run, the app already shows the photos in two UICollectionViews, named `hdCollectionView` and `thumbnailCollectionView` in the `AlbumViewController` respectively. 

`Main.storyboard` uses `AutoLayout`: the height of the toolbar is determined by its intrinsic size; the height of the `thumbnailCollectionView` is prefixed; `hdCollectionView` autosizes itself to take over the rest of the space. If you are interested in learning more about autosizing, the codelab [Build autosizing UITableViewCell with UIStackView](https://ripplearc.github.io/iOS-UI-AutoSize-UITableViewCell/) is an excellent place to go.

You will notice `AlbumViewController` relies on `PhotoModel` to provide the photos as well as their sizes and names. The photo list are populated in the `AppDelegate`and the photo is displayed through the `UIImageView` contained in either `HDCollectionViewCell` or `ThumbnailCollectionViewCell`. 

We also need to specify the size of the cell and the spacing between them in `AlbumViewController.swift` 

:pencil2: `AlbumViewController.swift` :straight_ruler: `line 27`
``` swift
override func viewDidLayoutSubviews() {
    CollectionView!.collectionViewLayout as? UICollectionViewFlowLayout {
        layout.itemSize = hdCollectionView.frame.size
        layout.minimumLineSpacing = 0
    }
    if let layout = thumbnailCollectionView!.collectionViewLayout as? UICollectionViewFlowLayout {
        layout.itemSize = CGSize(width: 30, height: thumbnailCollectionView.frame.size.height)
        layout.minimumLineSpacing = 2
    }
}
```
It should be noted that the width of the `UIImageView` in `thumbnailCollectionView` is longer than that of the cell. By setting `clipsToBounds` of the cell to true, and `contentMode` of the UIImageView to `.scaleAspectFill`, the `UIImageView` fills up the cell without being distorted. 

:pencil2: `AlbumViewController.swift` :straight_ruler: `line 65`
```swift
cell.clipsToBounds = true
cell.photoView?.contentMode = .scaleAspectFill
cell.photoView?.image = image
``` 
As a bonus, throughout the lab, if you need some assistance to check out if the layout behaves correctly, you can turn on the debug option which shows a vertical line in the middle and labels each image with its index: 

:pencil2: `AppDelegate.swift` :straight_ruler: `line 24`
```swift
albumViewController.debug = true
```

# 3. Parallax Animation of the HD CollectionView 
In this section, we will implement the parallax animation through the customized flowLayout of `hdCollectionView`, called `HDFlowLayout`. 

:octocat: Switch to the `bootcamp_hd_flowlayout_parallax` branch
```git
git co -b bootcamp_hd_flowlayout_parallax 
```
Open `HDFlowLayout.swift`. This class inherits from `UICollectionViewFlowLayout`, which provides many overridable methods to customize the behavior of the `UICollectionView`. In this lab, we need to override four of them to achieve the `parallax` effect. 

- `prepare()` is where values that do not change often should be computed and stored 
- `collectionViewContentSize` returns the size of the content view 
- `layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]?` calls each element in the content view contained by the rect 
- `layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes?` returns the attribute which contains the center and size of the cell 

Before filling out the blank, let's use landscape photos as an example to illustrate how the parallax effect is implemented. 

<img src="/images/iOS/UI/ScrollingAlbum/hd_parallax.png" width="900" height="450" />

1. `cellMaximumWidth`: The maximum width of a cell, and it is usually the width of UICollectionView
2. `cellFullSpacing`: The spacing between the current and the next cell, when both of them are landscape. 
3. `cellMaximumHeight`: The maximum height of a cell, and it is usually the height of UICollectionView 
4. `currentFractionComplete`: How much the current cell has moved away from the center, ranging from 0 to 1

`cellMaximumWidth`, `cellFullSpacing`, `cellHeight` is set at `viewDidLayoutSubviews` of `AlbumViewController`, and `currentFractionComplete` is computed on the fly.  

:pencil2: `AlbumViewController.swift` 
```swift
fileprivate func setupHDCollectionViewMeasurement() {
    hdCollectionView.cellFullSpacing = 100
    hdCollectionView.cellNormalWidth = hdCollectionView!.bounds.size.width - hdCollectionView.cellFullSpacing
    hdCollectionView.cellMaximumWidth = hdCollectionView!.bounds.size.width
    hdCollectionView.cellNormalSpacing = 0
    hdCollectionView.cellHeight = hdCollectionView.bounds.size.height
    ...
}
```

As the user swipes left, the `cellFullSpacing` and cell center remain the same, the width of the current cell, however, decreases by `cellFullSpacing x currentFractionComplete`, and the width of the next cell increases by `cellFullSpacing x (1-currentFractionComplete)`. It thus creates the parallax visual effect. 

We can now fill out the blank of the `HDFlowLayout`:

:pencil2: `HDFlowLayout.swift`
```swift
override func prepare() {
    cellEstimatedCenterPoints = []
    cellEstimatedFrames = []
    for itemIndex in 0 ..< cellCount {
        var cellCenter: CGPoint = CGPoint(x: 0, y: 0)
        cellCenter.y = collectionView!.frame.size.height / 2.0
        cellCenter.x = cellMaximumWidth * CGFloat(itemIndex) + cellMaximumWidth  / 2.0
        cellEstimatedCenterPoints.append(cellCenter)
        cellEstimatedFrames.append(CGRect.init(origin: CGPoint.init(x: cellMaximumWidth * CGFloat(itemIndex), y: 0), size: CGSize.init(width: cellMaximumWidth, height: cellHeight)))
    } 
}
```
The estimated size will be applied to all cells rather than the current and next cell, and the estimated center is used for testing which cells are visible as shown below.  

:pencil2: `HDFlowLayout.swift`
```swift
override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    var allAttributes: [UICollectionViewLayoutAttributes] = []
    for itemIndex in 0 ..< cellCount {
        if rect.intersects(cellEstimatedFrames[itemIndex]) {
            let indexPath = IndexPath(item: itemIndex, section: 0)
            let attributes = layoutAttributesForItem(at: indexPath)!
            allAttributes.append(attributes)
        }
    }
    return allAttributes
}
```
When the user swipes, the layout is called with `rect` which is the portion of the content view that is present on the screen. It is the timing for us to adjust the size of the cells that are visible. To save the computing time, we first intersect the `rect` with the esitimated cells, and call `layoutAttributesForItem` on the intersected candidate cells. 

:pencil2: `HDFlowLayout.swift`
```swift
override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    guard let attributes = super.layoutAttributesForItem(at: indexPath) else {
        return nil
    }
    
    if let collectionView = collectionView as? CellConfiguratedCollectionView,
        let cellSize = collectionView.cellSize(for: indexPath) {
        switch indexPath.item {
        case currentCellIndex:
            attributes.size = CGSize(width: max(minimumPhotoWidth, cellSize.width - cellFullSpacing * currentFractionComplete), height: cellSize.height)
            
        case currentCellIndex + 1:
            attributes.size = CGSize(width: max(minimumPhotoWidth, cellSize.width - cellFullSpacing * (1-currentFractionComplete)), height: cellSize.height)
            
        default:
            attributes.size = CGSize(width: cellMaximumWidth, height: cellHeight)
        }
        attributes.center = cellEstimatedCenterPoints[indexPath.row]
    }
    
    return attributes
}
```
As metioned above, the width of the current and next cell needs to be adjusted according to the `currentFractionComplete`.

You should be able to run the project and see the parallax effect. :tada: 
So far, `prepare()` is called every time the layout is invalidated. However, the estimated centers and sizes only need to be recomputed when the bound or data source changes. 

First, make `HDFlowLayout` implements `FlowLayoutInvalidateBehavior`, and guard the computation of estimated centers and sizes with `shouldLayoutEverything`. Then reset `shouldLayoutEverything` to `true` when bounds or data source changes.  

:pencil2: `HDFlowLayout.swift`
```swift
class HDFlowLayout: UICollectionViewFlowLayout, FlowLayoutInvalidateBehavior {
    ...
    var shouldLayoutEverything = true
    ...
}
```
:pencil2: `HDFlowLayout.swift`
```swift
override func prepare() {
    guard shouldLayoutEverything else { return }
    ...
    shouldLayoutEverything = false
}
```
:pencil2: `HDFlowLayout.swift`
```swift
//MARK: - Invalidate Context

extension HDFlowLayout {
    ...
    override func invalidationContext(forBoundsChange newBounds: CGRect) -> UICollectionViewLayoutInvalidationContext {
        let context = super.invalidationContext(forBoundsChange: newBounds)
        
        if newBounds.size != collectionView!.bounds.size {
            shouldLayoutEverything = true
        }
        return context
    }
    
    override func invalidateLayout(with context: UICollectionViewLayoutInvalidationContext) {
        if context.invalidateEverything || context.invalidateDataSourceCounts {
            shouldLayoutEverything = true
        }
        super.invalidateLayout(with: context)
    }
}
```
:octocat: If you have any difficulty in reSwitch to the `bootcamp_hd_flowlayout_parallax` branch
```git
git co -b bootcamp_hd_flowlayout_paralla
```

# 3. Accordion Animation of the Thumbnail CollectionView 
In this section, we will implement the accordion animation through the customized flowLayout of `thumbnailCollectionView`, called `ThumbnailMasterFlowLayout`. 

:octocat: Switch to the `bootcamp_thumbnail_flowlayout_accordion` branch
```git
git co -b bootcamp_thumbnail_flowlayout_accordion
```
Open `ThumbnailMasterFlowLayout.swift`. Similar to `HDFlowLayout`, this class also inherits from `UICollectionViewFlowLayout` and overrides some of the key methods. 

Accordion animation is more complicated than the parallax. Not only do we need to adjust the center and size of the cell being animated, the inset and offset of the UICollectionView also requires dynamic change. 

By default, the cell in the middle is unfolded while the rest are folded. As the user starts swiping, the unfolded cell quickly folds itself when the content view scrolls. After the scroll comes to a stop, whichever cell in the middle unfolds itself. 

Let's first analyze the lifecycle from folding to unfolding. 

<img src="/images/iOS/UI/ScrollingAlbum/thumbnail_accordion.png" width="774" height="518" />

1. `cellNormalSize`: The size of the cell when it is completely folded
2. `animatedCellSize`: The size of the cell that is being folded or unfolded which depends on the animation progress
3. `adjacentSpacingOfAnimatedCell`: The spacing by the side of the aniamted cell, which also depends on the animation progress
4. `cellNormalSpacing`: The spacing between two completed folded cells
5. `accordionAnimationManager.progress` Indicates how much the folding/unfolding has completed, based on the precomputed animation time and elapsed time

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate var animatedCellSize: CGSize {
    return CGSize(width: (cellFullWidth(for: animatedCellIndexPath) - cellNormalWidth) * accordionAnimationManager.progress() + cellNormalWidth, height: cellMaximumHeight)
}
```
:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate var adjacentSpacingOfAnimatedCell: CGFloat {
    return (cellFullSpacing - cellNormalSpacing) *  accordionAnimationManager.progress() + cellNormalSpacing
}
```
Determined by the progress, the `animatedCellSize.width` changes from `cellFullWidth` to `cellNormalWidth` linearly. So does the `adjacentSpacingOfAnimatedCell`.

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate var animatedCellCenter: CGPoint {
    return CGPoint(x: CGFloat(animatedCellIndex) * cellNormalWidthAndSpacing + adjacentSpacingOfAnimatedCell + animatedCellSize.width / 2, y: cellMaximumHeight / 2)
}
```
Once size and spacing of the animated cell are determined, then its center can be decided by counting how many cells with normal size to the left hand side of it. 

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate func centerAfterAnimatedCell(for indexPath: IndexPath) -> CGPoint {
    guard indexPath.item > animatedCellIndexPath.item else { return CGPoint.zero }
    return CGPoint(x: animatedCellCenter.x
        + animatedCellSize.width / 2.0
        + adjacentSpacingOfAnimatedCell
        + cellNormalWidthAndSpacing * fmax(0, CGFloat(indexPath.item - animatedCellIndex - 1))
        + cellNormalWidth / 2,
                   y: cellMaximumHeight / 2)
}
```
The centers of the cells to the right hand side of the animated cell should be adjusted when the center of the animated cell changes.

With these parameters being computed, it is easy to fill the blank of assigning values to the attributes. 

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    ...
    if indexPath.item < animatedCellIndex {
            attributes.size = cellNormalSize
            attributes.center = normalCenterPoints[indexPath.item]
        } else if indexPath.item > animatedCellIndex {
            attributes.size = cellNormalSize
            attributes.center = centerAfterAnimatedCell(for: indexPath)
        } else {
            attributes.size = animatedCellSize
            attributes.center = animatedCellCenter
        }
    }
    ...
```
Run the app, you will find the cells are stuck to the left hand side, and we need to set the inset:

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate var symmetricContentInset: CGFloat{
    return collectionView!.superview!.frame.size.width / 2.0
        - adjacentSpacingOfAnimatedCell
        - animatedCellSize.width / 2
}
```
Run the app, now the cell is no longer glued to the left hand side, but it still does not show up in the middle at the very beginning. The reason is the content offset value has not been set to initialize its position correctly.

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate func onAnimationUpdate(of type: AnimatedCellType) {
    ...
    if type == .unfolding {
        setContentOffset()
    }
}
```

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate func setContentOffset() {
    if accordionAnimationManager.progress() < 1,
        accordionAnimationManager.progress() > 0 {
        let insetOffset = symmetricContentInset - originalInsetAndContentOffset.0
        let cellCenterOffset: CGFloat = 0
        
        collectionView!.contentOffset.x = originalInsetAndContentOffset.1 - insetOffset - cellCenterOffset
    }
}
```
Run the app, everthing looks great except the cell doesn't stop exactly at its center. We therefore has to translate the cell when it is being unfolded by changing the offset value:

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate var unfoldingCenterOffset: CGFloat = 0

var originalInsetAndContentOffset: (CGFloat, CGFloat) = (0, 0) {
    didSet {
        if animatedCellType == .unfolding {
            unfoldingCenterOffset = originalInsetAndContentOffset.0 + originalInsetAndContentOffset.1 + cellNormalWidthAndSpacing / 2 - normalCenterPoints[currentCellIndex].x
        }
    }
}
```

:pencil2: `ThumbnailMasterFlowLayout.swift`
```swift
fileprivate func setContentOffset() {
    ...
    var cellCenterOffset: CGFloat = 0
    if animatedCellType == .unfolding {
        cellCenterOffset = unfoldingCenterOffset * accordionAnimationManager.progress()
    }
    ...
}
```
That should completes the accordion animation. :tada:  

:octocat: Switch to the branch `thumbnail_flowlayout_accordion` if you don't see the right accordion animation effect:
```git
git co -b thumbnail_flowlayout_accordion
```



