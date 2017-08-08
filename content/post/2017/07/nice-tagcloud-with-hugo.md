+++
title = "Nice tagcloud with Hugo"
date = "2017-07-21T15:42:16+02:00"
tags = [ "hugo" ]
+++

In my old blog with octopress I used the tag cloud plugin with [logarithmic distribution] for calculation of tag sizes.
The rendered tag cloud was pretty nice from the optic side. All existing approaches I saw for hugo([1], [2]) were not so nice, the main reason is the usage of logarithmic distribution in the calculation of tag size.

<!--more-->

You can compare the rendering with and without log function below:

___

![comparison](comparison.png)

___

`math.Log` template function [was added](https://github.com/gohugoio/hugo/pull/3662) to Hugo in release [0.25](https://github.com/gohugoio/hugo/releases/tag/v0.25), so now it can be used within templates to create nice tag clouds.

Based on the code [from Hendrik Sommerfeld][1], here is the implementation with log function:

```html
{{ if not (eq (len $.Site.Taxonomies.tags) 0) }}
    {{ $fontUnit := "rem" }}
    {{ $largestFontSize := 2.0 }}
    {{ $largestFontSize := 2.5 }}
    {{ $smallestFontSize := 1.0 }}
    {{ $fontSpread := sub $largestFontSize $smallestFontSize }}
    {{ $max := add (len (index $.Site.Taxonomies.tags.ByCount 0).Pages) 1 }}
    {{ $min := len (index $.Site.Taxonomies.tags.ByCount.Reverse 0).Pages }}
    {{ $spread := sub $max $min }}
    {{ $fontStep := div $fontSpread $spread }}

    <div id="tag-cloud" style="padding: 5px 15px">
        {{ range $name, $taxonomy := $.Site.Taxonomies.tags }}
            {{ $currentTagCount := len $taxonomy.Pages }}
            {{ $currentFontSize := (add $smallestFontSize (mul (sub $currentTagCount $min) $fontStep) ) }}
            {{ $count := len $taxonomy.Pages }}
            {{ $weigth := div (sub (math.Log $count) (math.Log $min)) (sub (math.Log $max) (math.Log $min)) }}
            {{ $currentFontSize := (add $smallestFontSize (mul (sub $largestFontSize $smallestFontSize) $weigth) ) }}
            <!--Current font size: {{$currentFontSize}}-->
            <a href="{{ "/tags/" | relLangURL }}{{ $name | urlize }}" style="font-size:{{$currentFontSize}}{{$fontUnit}}">{{ $name }}</a>
        {{ end }}
    </div>
{{ end }}
```

[Here](https://github.com/artem-sidorenko/www.sidorenko.io/blob/a43e51b2ac4b89e727bc02446f43764a1ff96995/layouts/partials/bloc/content/sidebar.html#L29-L51) is the entire sidebar file of this blog. The tag cloud section can be used on the similar way with other bootstrap based themes.

[logarithmic distribution]: https://gist.github.com/yeban/2290195#file-jekyll-tag-cloud-rb-L69-L70
[1]: https://www.henriksommerfeld.se/hugo-tag-could/
[2]: https://discourse.gohugo.io/t/weighted-tag-cloud/3491
