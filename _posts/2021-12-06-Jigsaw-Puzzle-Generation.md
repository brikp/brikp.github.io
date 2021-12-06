---
title: "Jigsaw Puzzle Generation"
permalink: /posts/jigsaw-puzzle-generation
layout: single
show_date: true
categories:
  - Devlog
tags:
  - Jigsaw
---

# Intro
Hi! I'm going to show you how I'm generating randomized puzzle pieces at runtime in my Jigsaw puzzle game (title still pending ;)). 
I'm using Godot engine, mono version (3.4 as of this post) with C#. 

This is my first devlog so it might be a bit rough. 
Let's get to it. 

# General processes outline
Currently, I'm only doing grid type puzzles with rectangle as a base (and it will probably stay that way). Shapes are applied using a set of masks. I'm heavily leveraging Godot Image class in the process. Each puzzle piece is a separate object with it's own texture and some additional data.
This is how it looks ingame:

![Animation.gif](/assets/images/jigsaw-puzzle-generation/animation.gif)

Full code for puzzle piece generating class - [gist](https://gist.github.com/brikp/e181c17577a922340442ebe0419e2c3a).

General algorithm looks more or less like this:

- Calculate puzzle piece amount in X and Y dimensions. Check how many full pieces fit on each axis, cut out an actual puzzle from the middle out 
- Generate puzzle pieces side data
- Generate pieces by taking parts of the image and applying masks to it according to the side data lookup
- Add pieces to the scene

# 1) Calculating actual puzzle size
This one is fairly simple. I take full image size and divide width and height by the piece size (piece size is chosen before puzzle generation). Then I check how many full pieces will fit and calculate actual puzzle size from it. Next I take the full image, and cut out the actual size from the middle out, leaving a part of the edge out. 

``` c#
int puzzleWidth = fullPuzzleTexture.GetWidth();  
int puzzleHeight = fullPuzzleTexture.GetHeight();  
// 1. Check how many pieces fit and get adjusted width and height  
int pieceXCount = puzzleWidth / PieceSize;  
int pieceYCount = puzzleHeight / PieceSize;  
int adjustedWidth = pieceXCount * pieceSize;  
int adjustedHeight = pieceYCount * pieceSize;  
// 2. Cut off new size from the full image from middle out  
int startingX = (puzzleWidth - adjustedWidth) / 2;  
int startingY = (puzzleHeight - adjustedHeight) / 2;  
Rect2 usedImagePartRect = new Rect2(startingX, startingY, adjustedWidth, adjustedHeight);  
adjustedPuzzleImage = new Image();  
adjustedPuzzleTexture = new ImageTexture();  
adjustedPuzzleImage = fullPuzzleImage.GetRect(usedImagePartRect);  
adjustedPuzzleTexture.CreateFromImage(adjustedPuzzleImage);
```

# 2) Puzzle side data
Here I create a lookup based on piece index (x,y) position, as in top left piece is (0,0), second row left piece would be (0,1) etc. 

Starting from top left corner, I go through each column topdown, and then move on to the next one.

SideData is a simple class that holds data for each side of the puzzle in tuples:
``` c#
(bool isEdge, PuzzleMasks mask, bool isTab)
```
Since I'm going top -> down and then left -> right, and the first piece is a corner, I assume that top and left sides can be matched to the adjacent piece (since it was generated already), and bottom and right sides are to be randomized. I randomize two things:
1) Is the side a tab (convex shape) or a blank (concave shape)
2) Which mask will be used for the shape

``` c#
// Top side of the puzzle  
if (puzzlePosition != PuzzlePosition.TOP_LEFT &&  
 puzzlePosition != PuzzlePosition.TOP &&   
puzzlePosition != PuzzlePosition.TOP_RIGHT)  
{  
 Vector2 neighbourPuzzleIndex = puzzleIndex;  
 neighbourPuzzleIndex.y -= 1;  
 maskKey = tabsAndBlanks[neighbourPuzzleIndex].Bottom.mask;  
 isTab = !tabsAndBlanks[neighbourPuzzleIndex].Bottom.isTab;  
  
 result.Top = (false, maskKey, isTab);  
}  
// Left side of the puzzle  
if (puzzlePosition != PuzzlePosition.TOP_LEFT &&  
 puzzlePosition != PuzzlePosition.LEFT &&  
 puzzlePosition != PuzzlePosition.BOT_LEFT)  
{  
 Vector2 neighbourPuzzleIndex = puzzleIndex;  
 neighbourPuzzleIndex.x -= 1;  
 maskKey = tabsAndBlanks[neighbourPuzzleIndex].Right.mask;  
 isTab = !tabsAndBlanks[neighbourPuzzleIndex].Right.isTab;  
  
 result.Left = (false, maskKey, isTab);  
}
// Right side of the puzzle  
if (puzzlePosition != PuzzlePosition.TOP_RIGHT &&  
 puzzlePosition != PuzzlePosition.RIGHT &&  
 puzzlePosition != PuzzlePosition.BOT_RIGHT)  
{  
 int randomKey = (int)GD.RandRange(0, availableMasksKeys.Length - 0.001);  
 maskKey = availableMasksKeys[randomKey]; isTab = GD.Randf() <= 0.5f;  
  
 result.Right = (false, maskKey, isTab);  
}  
// Bottom side of the puzzle  
if (puzzlePosition != PuzzlePosition.BOT_LEFT &&  
 puzzlePosition != PuzzlePosition.BOTTOM &&  
 puzzlePosition != PuzzlePosition.BOT_RIGHT)  
{  
 int randomKey = (int)GD.RandRange(0, availableMasksKeys.Length - 0.001);  
 maskKey = availableMasksKeys[randomKey]; isTab = GD.Randf() <= 0.5f;  
  
 result.Bottom = (false, maskKey, isTab);  
}
```

Tab/blank randomization is skewed towards 2/2 layout (~80%), since I thought it looked the most natural, with less pieces having 3/1 or 4/0 tab/blank ratio. This is applied after the above code.

``` c#
float twoTabFactor = 0.8f;  
if (result.Left.isTab && result.Top.isTab)  
{  
 if (GD.Randf() < twoTabFactor)  
 { result.Bottom.isTab = false;  
 result.Right.isTab = false;  
 }
}
// similiar for other combinations of top/left isTab
```

Side data is serialized and saved to file after it's generated so the puzzle can be reloaded later with the same layout.

# 3) Generating puzzle pieces
Now that side data lookup is available, I can generate the actual images for the pieces. This is how a mask and final pieces look:

![jigsaw mask](/assets/images/jigsaw-puzzle-generation/mask.jpg)
![pieces](/assets/images/jigsaw-puzzle-generation/pieces.jpg)

I go through all the pieces again (same order as before, it does not matter here though). 

First we just cut out square piece without tabs or blanks:
``` c#
Image image = adjustedPuzzleImage.GetRect(rect);
```

Then blanks are applied: 
``` c#
private void ApplyBlankMask(PuzzleSideData puzzleSideData, Image image, Image blankMask, PuzzlePosition side)  
{  
	Rect2 maskRect = blankMask.GetUsedRect();  
	Vector2 point = Vector2.Zero;  
	int maskX = (int)maskRect.Position.x;  
	int maskY;  

	image.Lock();  
	blankMask.Lock();  
	// 1. Get starting point - middle of side being masked  
	// 2. Move starting point to align with top left corner of the maskRect 
	switch (side)  
	{ 
		case PuzzlePosition.RIGHT:  
		point.x = image.GetWidth();  
		point.y = 0;  
		point.x -= maskRect.Size.x;  
		break;
 // ... similiar cases for all sides
 // 3. Apply the mask by iterating over the pixels  
	for (int x = (int)point.x; x < point.x + maskRect.Size.x; x += 1)  
	{  
	 maskY = (int)maskRect.Position.y;  
	 for (int y = (int)point.y; y < point.y + maskRect.Size.y; y += 1)  
		{ 
			Color color = blankMask.GetPixel(maskX, maskY);  
			Color imageColor = image.GetPixel(x, y);  
			if (color.a > 0.1f)  
			{ 
				imageColor.a = 1 - color.a;  
				image.SetPixel(x, y, imageColor);  
		 	} 
			maskY += 1;  
		} 
	 	maskX += 1;  
	}  
  
image.Unlock();  
blankMask.Unlock();
```

And lastly, tabs are applied: 
``` c#
private Image AddTabsToImage(Image image, Rect2 rect, PuzzleSideData puzzleSideData)  
{  
 Image mask, sideImage;  
 Vector2 topLeft = new Vector2(maskAdjustedSize / 2, maskAdjustedSize / 2);  
 Rect2 maskRect;  
  
 // 1. Create new image with target size  
 Vector2 targetSize = image.GetSize() + topLeft * 2;  
 Image targetImage = new Image();  
 targetImage.Create((int)targetSize.x, (int)targetSize.y, false, image.GetFormat());  
  
 // 2. Set pixels for base image (starting from top+left resize vector position)  
 targetImage.BlitRect(image, image.GetUsedRect(), topLeft);  
  
 // 3. Get pixels for the tabs from the full puzzle  
 // 4. Blit mask with image on each side if needed 
 if (puzzleSideData.Top.mask != PuzzleMasks.NOT_SET &&  
   puzzleSideData.Top.isTab)  
 {  
	 mask = maskImages[puzzleSideData.Top.mask].top;  
	 maskRect = mask.GetUsedRect();  

	 maskRect.Position = rect.Position;  
	 maskRect.Position -= new Vector2(0, maskRect.Size.y);  
	 sideImage = adjustedPuzzleImage.GetRect(maskRect);  
	 ApplyImageMask(sideImage, mask.GetRect(mask.GetUsedRect()));  

	 targetImage.BlitRect(  
	 sideImage, new Rect2(0, 0, maskRect.Size.x, maskRect.Size.y),  
	 new Vector2(topLeft.x, topLeft.y - maskRect.Size.y));
 }
 // Similiar for other sides
}

// Copy pixels if alpha != 0
private void ApplyImageMask(Image image, Image mask)  
{  
 image.Lock();  
 mask.Lock();  
  
 for (int x = 0; x < image.GetWidth(); x += 1)  
 { 
 	for (int y = 0; y < image.GetHeight(); y += 1)  
	{ 
		Color maskColor = mask.GetPixel(x, y); 
		 
		if (maskColor.a == 0)  
		{ 
			image.SetPixel(x, y, new Color(0, 0, 0, 0));  
		} 
		if (maskColor.a > 0 && maskColor.a < 1)  
		{ 
			Color imageColor = image.GetPixel(x, y);  
			imageColor.a = maskColor.a;  
			image.SetPixel(x, y, imageColor);  
		} 
	}
}  
image.Unlock();  
mask.Unlock();  
}
 
```

After all this is done, I also add some other data and collision shape to each piece (important to disable monitoring - it kills performance and I don't use it for puzzle pieces anyway). 
Last thing left to do is to add pieces to the scene, which is handled by another class. 

The game in it's current state can be downloaded on [itch.io](https://brikp.itch.io/infinite-jigsaw).
