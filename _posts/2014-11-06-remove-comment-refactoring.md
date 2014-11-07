---
layout: post
---

I'm not a fan of comments in code. I agree with those who say it is generally a code
smell. Thus I was underwhelmed when I came across this CoffeeScript in our codebase.

{% highlight coffee %}
addKeypressNavigation: (event) ->
  # check if we have something in focus
  if $('textarea:focus').length == 0 && $('input:focus').length == 0
    ... do stuff
{% endhighlight %}

I did an Extract Method refactoring to clean the code up.

{% highlight coffee %}
addKeypressNavigation: (event) ->
  unless _.any ['textarea', 'input'], @hasFocus
    ... do stuff

hasFocus: (element) ->
  $("#{element}:focus").length isnt 0
{% endhighlight %}

I like this refactoring for a couple of reasons. Foremost it removes the comment, which was the
original goal. But if that were the only point I could have just removed the comment and
been done with it. However, I also ended up _expressing the
sentiments of the 
comment in code_. Lookit how that line reads now: "Unless any of these elements has focus,
do stuff." Poetry!

By any measure, this is a small refactoring. It doesn't make the code _that_ much cleaner,
but I feel a whole lot better about leaving the second implementation behind for those to follow
me. And that's a good thing.

#### A Closing Comment

I am not anti-comment. I think it does have a role, but I feel that the test for a comment
should be "why is this code here" not "what is this code doing." For example,
we have this comment in our code.

{% highlight ruby %}
class StudentResponse < ActiveRecord::Base
  def update_data_attr!(data_params)
    # work around for issue https://github.com/rails/rails/issues/6127
    self.data_will_change!
    data_params.each do |key, value|
      data[key] = value
    end
    save!
    self
  end
end
{% endhighlight %}

This feels entirely legitimate to me. Tell me again why we need
`StudentResponse#update_data_attr!`? Oh that's right. A bug with handling Postgres Hstore
columns. In Rails. And when we upgrade Rails we will remove this method.
