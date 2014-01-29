---
layout: post
category : lessons
tagline: "Supporting tagline"
tags : [intro, beginner, jekyll, tutorial]
---
{% include JB/setup %}


This is an example of a draft. Read more here: [http://jekyllrb.com/docs/drafts/](http://jekyllrb.com/docs/drafts/)


{% highlight ruby %}
def show
  @widget = Widget(params[:id])
  respond_to do |format|
    format.html # show.html.erb
    format.json { render json: @widget }
  end
end
{% endhighlight %}

``` ruby
require 'rubygems'

def foo
puts 'foo'
end

#comment
```

{% highlight html linenos%}
<script type="text/javascript">
    $.getJSON("http://api.douban.com/book/subject/1220562?alt=xd&callback=?", function(book) {
        var subj = DOUBAN.parseSubject(book);
        var tl = subj.title ? subj.title : "";
        var author = subj.author ? subj.author : "";
        var tmp = "<img src="+subj.link.image+" style='margin:10px;float:left'>";
        tmp += "<div>Title : <a href="+subj.link.alternate+" target='_blank'>"+tl+"</a></div>";
        if (subj.attribute.author)
            tmp += "<div>Authors : "+(subj.attribute.author.join(' / '))+"</div>";
        if (subj.attribute.isbn13) 
            tmp += "<div>ISBN : "+(subj.attribute.isbn13.join(' / '))+"</div>";
        if (subj.attribute.price) 
            tmp += "<div>Price : "+(subj.attribute.price.join(' <br/>   '))+"</div>";
        if (subj.attribute.pages) 
            tmp += "<div>Pages : "+(subj.attribute.pages.join(' / '))+"</div>";
        if (subj.attribute.publisher) 
            tmp += "<div>Publisher : "+(subj.attribute.publisher.join(' <br/>   '))+"</div>";
        if (subj.attribute.pubdate) 
            tmp += "<div>Pubdate : "+(subj.attribute.pubdate.join(' / '))+"</div>";
        if (subj.rating.average)
            tmp +="<div>Rating: "+subj.rating.average+" / "+subj.rating.numRaters+decodeURI("%E4%BA%BA")+ "</div>"
        tmp += "<p>"+(subj.summary ? subj.summary : "")+"</p>";
        document.body.innerHTML = tmp;
    });
</script>
{% endhighlight %}

{% gist jeremybai/63ebca829c4c04db224c %}