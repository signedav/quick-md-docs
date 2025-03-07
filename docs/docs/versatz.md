# Versatzproblem bei Grenzpunkten des CadastraSymbol Fonts in QGIS

## What is the bug or the crash?

It should be possible to use the baseline of a font for symbol placement.

The following image shows the font [cadastra](https://www.cadastre-manual.admin.ch/de/plan-fur-das-grundbuch) which has set the baseline to the center of the symbol. The point should be centered over the intersection point of the lines.

![image](https://gist.github.com/user-attachments/assets/236b689c-1fc3-4e51-8d68-0d29bd48673f)

But the symbols of the cadastra font are centered on the baseline.

![image](https://gist.github.com/user-attachments/assets/a5272cb6-c266-44df-a4a6-9b41c8ca2a08)

It can be assumed that with the “Bottom” setting, the symbols should be placed in the correct position.

But it doensn't do that. Neither with the cadastra font:

![image](https://gist.github.com/user-attachments/assets/fb32e978-135e-4ac2-8423-9d03f2a3c0a8)

Nor with any other font (here Deja Vu):

![image](https://gist.github.com/user-attachments/assets/c7174885-028a-40bf-b0b7-d1d32c9cae2b)

The symbols are slighly above the bottom line.

[QGIS Issue](https://github.com/qgis/QGIS/issues/59732)

## The current situation

The *middlepoint* is everywhere the "source of truth": it's calculated in the font marker layer as the *middle* of the `baseline` and `QFontMetrics::ascent()`, what is basically the highest possible point of a font (like the top of the "aigu" of the À). This summed up to the point's coordinates defines, where to draw top left of the symbol. With the anchor "center" this would work.

But when we want to have the bottom of the symbol on the coordinate of the point, it sums up again the half of the symbol size. And this would work *but* here it doesn't calculate the half of the distance between the `baseline` and the `QFontMetrics::ascent()` but instead it takes the scaled symbolsize - and **these two measurements differed**. What leaded to the wrong placement:

![image](https://gist.github.com/user-attachments/assets/f4fc46ad-ff15-4115-85fa-97d72999ef6c)

The left is what it is expected to be and the right the one that was the situation. The purple line is one measurement and the cyan one the other. When summing up the half of the first size with the half of the second size then the symbol got misplaced.

With this fix, the two offsets are always calculated the same, which leads to a correct position of the FontMarkers on Bottom.

Means before on bottom (example with Font DejaVu Sans and CadastraSymbol):

![image](https://gist.github.com/user-attachments/assets/7f1b1998-a874-42ab-84dd-6c195295bf33)

Now:

![image](https://gist.github.com/user-attachments/assets/0fcc1902-7e01-4c96-8617-952650468cfb)

[QGIS Pull Request](https://github.com/qgis/QGIS/pull/60044)

## Other approaches

The first idea to fix it came up, by creating an anchor on the **bounds**, like this:

![image](https://gist.github.com/user-attachments/assets/20c67cbe-aec8-408a-9d93-47373aa26e79)

But this does not work in the case of cadastre since the "center" of the symbol is not the center of the bounds.

![image](https://gist.github.com/user-attachments/assets/0f4cd257-f494-4e96-97a0-d222b17cf381)

The cadastre needs the baseline setting. Although the approach with the **bounds** makes sense for other fonts and letters (like the j here on baseline compared to the bounds above):

![image](https://gist.github.com/user-attachments/assets/d7ce7abb-934a-4cb2-a052-a2278a58b478)

We did some research and tests on this approach and finally opened a draft Pull Request and an issue. Another QGIS core developer already mentioned interrests to implement it.

[QGIS Issue](https://github.com/qgis/QGIS/issues/60836)

[QGIS Pull Request Work in Progress (closed)](https://github.com/qgis/QGIS/pull/60080)
