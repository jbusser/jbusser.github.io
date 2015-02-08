---
layout: post
---

At my day job, we have been making a transition from Marionette to EmberJS. My only
regret so far has been with our approach to testing our Ember code. We have generally
relied on Capybara's Poltergeist driver for that job, and that has been... unsatisfying. So I have
been looking into Ember's integration tests as a substitute. However, it didn't take long
to run into a problem:

How the hell do you deal with asynchronous relationships?

In case you read blog posts the way I do, which is to say I scan for interesting code
samples all the while muttering "enough words! Just gimme the answer," here's a pattern
I have come up with.

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    materializeComment = (comment) ->
      comment.get('user').then ->
        comment

    materializeComments = (comments) ->
      Em.RSVP.all(comments.map materializeComment).then ->
        comments

    @store.find('post', params.post_id).then (post) ->
      Em.RSVP.all([
        post.get('category')
        post.get('comments').then materializeComments
      ]).then ->
        post
{% endhighlight %}

This code will ensure that our Post model with have its related category,
comments, and for each comment its associated user before invoking the
`afterModel` hook.

OK, with that out of the way, let's see how we got there.

Let's say that in our Ember router file we have a resource like

{% highlight coffee %}
App.Router.map ->
  @resource 'posts', ->
    @resource 'post', path: ':post_id'
{% endhighlight %}

And the business tells us that when a visitor views to the Post page that he should
see the basic post attributes, such as headline, body, publication date, and so on. In
this case, the model hook for the route is simple. [We don't have to write anything at all.
Ember generates it for
us.](http://emberjs.com/guides/routing/defining-your-routes/#toc_dynamic-segments) Thanks,
Ember Core Team! What's next in the backlog?

### Moving Beyond the Simple ###

Later, the business comes back to us and says that in addition to seeing the basic post
attributes, a visitor to the Post page should also see the post's category.

That's a bit tricker, because in our object model, a Post belongs to a Category. And
since a Cateogry is a separate model, we will need to structure our tests accordingly.

{% highlight coffee %}
App.Cateogry = DS.Model.extend
  posts: DS.hasMany 'post'

App.Post = DS.Model.extend
  category: DS.belongsTo 'category'
{% endhighlight %}

Let's use Ember Data's fixture adpater for our testing. The initial integration test looks
like:

{% highlight coffee %}
module 'Post view',
  setup: ->
    App.ApplicationAdapter = DS.FixtureAdapter.extend()

    App.Category.reopenClass FIXTURES: [
      id: 10, name: 'Test Category'
    ]

    App.Post.reopenClass FIXTURES: [
      id: 1, headline: 'Post Headline', category: 10
    ]

test 'displays the post properties', ->
  expect 1

  Em.run ->
    visit('/posts/1').then ->
      # assert something about the rendered html
{% endhighlight %}

But this will fail. To make it work we have to change the nature of the `belongsTo`
relationship on the Post model so that it is asynchronous because `DS.FixtureAdapter`
assumes all relationships are async. The Post model definition now becomes:

{% highlight coffee %}
App.Post = DS.Model.extend
  category: DS.belongsTo 'category', async: true
{% endhighlight %}

Closer, but not quite there. The problem now is that, as far as Ember is concerned,
the post's category relationship has not been resolved yet. Looks like we have to write
some code in our route now. Thanks for nothing, Ember Core Team!

### Fetching the Category

So let's see what we can do with `PostRoute`. Here's one attempt:

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    # re-implement the default model hook and then...
    @store.find(params.post_id).then (post) ->
      # ...get the category! easy-peasy
      post.get('category')
{% endhighlight %}

Not quite. This doesn't work because the model hook will now return a promise that will
resolve to the related category, and the model for `PostRoute` should really be a post.

I want to take a minute and talk about this `@store.find().then()` business because it's
critical to understanding the rest of the solution. The RSVP.Promise
API talks about [chaining promises](http://emberjs.com/api/classes/RSVP.Promise.html#toc_chaining),
and that's what's going on here.

We need to read this code two ways to fully appreciate what's happening. For the first
pass, let's read it through a synchronous lens. When the model hook executes, it calls
`@store.find().then()`. The call to `@store.find()` returns a promise, on which we will
call `.then()`, and it too returns a promise. Ember is totally cool with this because
it expects the model hook to return either an object or a promise. When it receives a promise, the route will cease
processing until the promise has been settled, either resolved or rejected.

But what do those promises resolve to? That's where we read this code the second way.

In the case of `@store.find(params.post_id)`, that promise will resolve to a Post model,
and when that promise gets resolved, the function we passed in as an argument to the
"success" callback for the `.then()` will get called. That function will receive as its first argument whatever was
returned from its upstream promise, the Post model in this case. That's chaining.

But then things get interesting when we look at what happens inside the `then()` function.
We have our post, so we call `.get('category')` on it. Since we declared that
`Post#category` is an async relationship, that call to `.get('category')` returns a
(possibly unresolved) promise, and that's the promise we end up returning from the model hook. This
is not what we want because when that promise resolves, it will have the value of the
related category model, not the post model.

Let's get the code to do what we want.

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    @store.find(params.post_id).then (post) ->
      post.get('category').then ->
        post
{% endhighlight %}

### A Pattern Emerges ###

So this works. We find a post model asynchronously, and when it resolves, we fetch its related
category. When the related category comes back from the server, _we return the
object that the caller asked for_.

To generalize, the pattern is

* Request an object
* Satisfy any dependent relationships
* Return the requested object

The caller of PostRoute's model hook has no idea that we are generating two Ajax calls, one to find the post and
another to find the related category. This is an implementation that the promise API
abstracts away. From the caller's perspective, it only is aware of that original promise to
retrieve the post.

### Composing Promises ###

The post page now displays all the required post properties in addition to displaying the
related Category model. The business is happy.

Until it isn't.

The business has a new requirement: In addition to displaying the related category,
display all related comments as well.

The object model now looks like:

{% highlight coffee %}
App.Cateogry = DS.Model.extend
  posts: DS.hasMany 'post'

App.Comment = DS.Model.extend
  post: DS.belongsTo 'post'

App.Post = DS.Model.extend
  category: DS.belongsTo 'category', async: true
  comments: DS.hasMany 'comment', async: true
{% endhighlight %}

And the fixture data:

{% highlight coffee %}
module 'Post view',
  setup: ->
    App.ApplicationAdapter = DS.FixtureAdapter.extend()

    App.Category.reopenClass FIXTURES: [
      id: 10, name: 'Test Category'
    ]

    App.Comment.reopenClass FIXTURES: [
      { id: 20, body: 'First comment' }
    ]

    App.Post.reopenClass FIXTURES: [
      id: 1, headline: 'Post Headline', category: 10, comments: [20]
    ]
{% endhighlight %}

So now we have to resolve two async relationships before we exit the route and render our
template. To properly handle this condition we'll rely on `Em.RSVP.all`.

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    @store.find(params.post_id).then (post) ->
      # we have our Post model
      Em.RSVP.all([
        # "category" is async, so it returns a promise
        post.get('category')
        # "comments" is async, too
        post.get('comments')
      ]).then ->
        post
{% endhighlight %}

`Em.RSVP.all` takes an array of promises and wraps them up in a single larger promise (an
über-promise? a promise of promises?). Since `Em.RSVP.all` returns a promise, it is
thenable. Should Ember have a problem fetching either the category or comments relationships,
the entire promise will reject, and since we're still in the route's model hook, we can abort cleanly.

### Going a Bit Farther Down the Rabbit Hole ###

The new page goes live, and some time later a new requirement arrives: Add the name of the
commenter to each comment.

The new object model looks like:

{% highlight coffee %}
App.Cateogry = DS.Model.extend
  posts: DS.hasMany 'post'

App.Comment = DS.Model.extend
  post: DS.belongsTo 'post'
  user: DS.belongsTo 'user', async: true

App.Post = DS.Model.extend
  category: DS.belongsTo 'category', async: true
  comments: DS.hasMany 'comment', async: true

App.User = DS.Model.extend
  comments: DS.hasMany 'comment'
{% endhighlight %}

And the fixture data:

{% highlight coffee %}
module 'Post view',
  setup: ->
    App.ApplicationAdapter = DS.FixtureAdapter.extend()

    App.Category.reopenClass FIXTURES: [
      id: 10, name: 'Test Category'
    ]

    App.Comment.reopenClass FIXTURES: [
      id: 20, user: 30, body: 'First comment'
    ]

    App.Post.reopenClass FIXTURES: [
      id: 1, headline: 'Post Headline', category: 10, comments: [20]
    ]

    App.User.reopenClass FIXTURES: [
      id: 30, firstName: 'Concerned', lastName: 'Commenter'
    ]
{% endhighlight %}

With this new requirement we have to resolve an asycnronous belongs-to relationship for
each related comment. By applying our pattern, the first refactoring step looks like
this

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    materializeComments = (comments) ->
      comments

    @store.find('post', params.post_id).then (post) ->
      Em.RSVP.all([
        post.get('category')
        post.get('comments').then materializeComments
      ]).then ->
        post
{% endhighlight %}

This code behaves exactly the same as the previous implementation. We have just added the (at this point,
unnecessary) abstraction of the function `materializeComments`. Notice that the pattern
exists here: get a thing, perform work with that thing, return that thing.

Now let's try to retrieve the user for each comment. We need to call
`comment.get('user')` for each comment in the collection, which sounds like a great use
for `map()`. Since the user relationship is async, that will return a
promise. We want to halt execution until every comment has retrieved its related user,
another candidate for `Em.RSVP.all`. Here's a first pass:

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    materializeComments = (comments) ->
      Em.RSVP.all(comments.map (comment) ->
        comment.get('user').then ->
          comment).then ->
            comments

    @store.find('post', params.post_id).then (post) ->
      Em.RSVP.all([
        post.get('category')
        post.get('comments').then materializeComments
      ]).then ->
        post
{% endhighlight %}

And that's fine, I guess, but I think it's a little hard to comprehend at a glance. Let's
do a further refactoring and abstract out that innermost anonymous function and give it a
name.

{% highlight coffee %}
App.PostRoute = Em.Route.extend
  model: (params) ->
    # given a comment...
    materializeComment = (comment) ->
      # ...do something with the comment...
      comment.get('user').then ->
        # ...return that comment
        comment

    # given a collection of comments...
    materializeComments = (comments) ->
      # ...do something with each comment...
      Em.RSVP.all(comments.map materializeComment).then ->
        # ...return those comments
        comments

    # given a post...
    @store.find('post', params.post_id).then (post) ->
      # ...do something with the post...
      Em.RSVP.all([
        post.get('category')
        post.get('comments').then materializeComments
      ]).then ->
        # ...return that post
        post
{% endhighlight %}

That's better. We've gotten rid of some rightward shift (passes the [squint
test](http://www.confreaks.com/videos/3358-railsconf-all-the-little-things) with a higher
grade perhaps?) and made the `materializeComments` function easier to comprehend, I think.
Each call to `materializeComment` will return a promise that will be resolved with a value
of the comment when we retrieve the related user.

### But What About Sideloading?

Another possible solution to this problem involves enriching the JSON that gets returned
when Ember receives a post, e.g.

{% highlight coffee %}
# GET /posts/1.json
{
  "post": { "id": 1, "headline": "Post Headline", "category": 10, "comments": [10] },
  "comments": [ { "id": 20, "body": "First Comment", "user": 30 } ],
  "users": [ { "id": 30, "first_name": "Concerned", "last_name": "Commenter" } ]
}
{% endhighlight %}

It's definitely an option, and the strongest arguments in favor of this approach are that
it eliminates the complex model hook and that it reduces the number of HTTP calls required
to render our post template. For any given post, we need to make only one request and we
get the whole enchilada. However, it carries some drawbacks:

* Ember Data's Fixture Adapter does not support sideloading, which makes realistic testing difficult
* The design of your API endpoints may be too tightly coupled to the UI. Does it make
sense that you change your server API when the business wants the commenter's screen name
to our post page and you have to add that attribute to your JSON payload?
* Your API payloads may end up delivering redundant information, bloating your payloads, something I have
experienced in my company's app

In light of these issues, we have turned toward the async pattern I have outlined above.
The use of fixutres has forced us to look at creating smaller API payloads that return
only the model and references to related models, minimizing and eventually eliminating
sideloading altogether, and creating, I think, a more composable JSON API.
