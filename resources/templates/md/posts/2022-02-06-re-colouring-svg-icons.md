{:title "Re-colouring SVG icons"
 :layout :post
 :toc false
 :tags  ["colour" "SVG" "icon" "HTML"]}

Sometimes you have an icon drawn in SVG but you want to display it in different
colours depending on the state, for example when a given menu item is selected
or when you want to warn the user about something.

Most SVG icons are draw on a transparent background. The foreground colour in
such icons can be easily converted using SVG filter
[`feColorMatrix`](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/feColorMatrix).
All we have to do is creating a matrix where the original colours are zeroed out
and the constant component is taken from the new colour. The transparency
channel is left unmodified, so that we draw only there where the original icon
has something drawn. The matrix should look like this:

```svg
0 0 0 0 r
0 0 0 0 g
0 0 0 0 b
0 0 0 1 0
```

If you try putting the RGB values in the matrix, you will get a completely white
icon in most cases. This is because SVG uses the sRGB colour space where the R,
G, B channels get a value between 0 and 1. If you now try setting the R, G, B
values by dividing the target values by 255.0, you will get more colours but
they will be strangely off from the target. It turns out that the transformation
to sRGB is non-linear: the correct formula is `pow(x / 255.0, 2.2)`.

Now all that remains is adding the transformation filter to the HTML page and
setting it for the icon. This can be done like this:

```html
<svg height="0" width="0">
  <filter id="filter">
    <feColorMatrix in="SourceGraphic" type="matrix"
                   values="0 0 0 0 0.05 0 0 0 0 0.35 0 0 0 0 0.84 0 0 0 1 0"/>
  </filter>
</svg>
<span style="filter:url(#filter)">
  <img src="icon.svg"/>
</span>
```

The [example](https://github.com/bentomi/colour-change) shows the whole setup
with the colour matrix set dynamically.
